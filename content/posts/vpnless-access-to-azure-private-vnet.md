---
title: "VPN-less access to Azure Private vnet"
date: 2021-09-29T13:49:00+02:00
summary: A simple method to get access to private networks on private clouds.
---

Accessing cloud resources that are not exposed to the Internet is a constant headache. Neither Azure nor AWS have a VPN solution that can be attached to any private subnet and just works. Traditionally, you would spin up a VM with openssh and connect it to some public subnet as well as the relevant private subnet, punch a hole in the network policy and use SSH tunnelling. Forgotten "bastion" VMs will put you on the shortlist to "win" the next security audit. Is there no way avoid this embarrassment?

There is a new class of "VPN-less" systems you can use. Have a look at [Boundary](https://www.boundaryproject.io/), [Teleport](https://goteleport.com/) or [StrongDM](https://www.strongdm.com/). However, it requires some effort to introduce one of these services into existing infrastructure. If your need is temporary, the cost is probably bigger than the benefit.

With Azure, there is a method that supports some tunnelling scenarios with simpler setup *and* better security. There is a somewhat obscure service called [Azure Relay](https://docs.microsoft.com/en-us/azure/azure-relay/relay-what-is-it) which has grown out of Azure Service Bus. Its "Hybrid connection" mode allows two parties to establish a point-to-point tunnel by connecting to Azure Relay from each end, thus allowing both ends to traverse NAT and avoid inbound firewalling. Using this service, you can establish a connection *from* a private vnet to Azure Relay and thus forward a tunnel into the private vnet without directly involving an Internet-facing vnet.

In order to facilitate using Hybrid connections for tunnelling, Azure has published the [Azure Relay Bridge](https://github.com/Azure/azure-relay-bridge). It is a command line tool called azbridge, which can act as both client and forwarder with Azure Relay. azbridge can be run with either -L or -R (inspired by openssh flags with similar meaning). `azbridge -L` opens a local TCP listener socket and forwards TCP connections to a Hybrid connection, while `azbridge -R` receives traffic from a Hybrid connection and forwards onto some remote target host/port pair. Connections are authenticated with a SAS URI.

| ![Azure Relay tunnelling across two outbound connections](/vpnless-access-to-azure-private-vnet/azure-relay.drawio.png) |
|:--:|
| *Azure Relay tunnelling across two outbound connections* |

This method has several advantages over the traditional ssh bastion host:
- few assumptions: any Docker-enabled host which can reach the Internet can reach any private vnet,
- least privilege: precise control over both who can connect and what they can connect to, and
- auditable: inspecting a hybrid connection tells us both who can connect and what they can connect to.

In order to make deploying this tool simple, I have taken the liberty to publish this tool to Docker Hub as [bittrance/azbridge](https://hub.docker.com/repository/docker/bittrance/azbridge).

## Demonstration

For easy cleanup, we create a Resource Group for our demonstration:

```bash
az group create \
    --name rg-access-bridge \
    --location westeurope
```

Here is the private vnet that we want to access. The second command gives you the VNET_ID that we need when we set up further resources.

```bash
az network vnet create \
    --resource-group rg-access-bridge \
    --name vnet-access-bridge \
    --address-prefixes 10.0.0.0/16

az network vnet show \
    --resource-group rg-access-bridge \
    --name vnet-access-bridge \
    --query id \
    --output tsv
```

### Example target resource

For the sake of this demonstration, we can set up a simple echo server which serves as our "target resource" in the picture above. The goal is to be able to connect to this container despite it having no public IP.

In order to be able to create containers on a vnet, ACI needs to have a subnet delegated to it. Such a subnet can only contain containers. For more information, see ACI [Virtual Networking Scenarios](https://docs.microsoft.com/en-us/azure/container-instances/container-instances-virtual-network-concepts) documentation. In this example, we allow ACI to create the subnet for itself.

```bash
az container create \
    --resource-group rg-access-bridge \
    --image hashicorp/http-echo \
    --command-line '/http-echo -listen=:8080 -text="Hello World"' \
    --ports 8080 \
    --name aci-echo-server \
    --vnet VNET_ID \
    --subnet snet-access-bridge \
    --subnet-address-prefix 10.0.0.0/24
```

### Setting up a relay

First, we create a Relay namespace. It acts as a grouping resource for our forwarders. Each forwarder is represented by a hybrid connection, which is basically just a name that allows connecting and forwarding parties to meet.

```bash
az relay namespace create \
    --resource-group rg-access-bridge \
    --name relns-access-bridge \
    --location westeurope

az relay hyco create \
    --resource-group rg-access-bridge \
    --namespace-name relns-access-bridge \
    --name hyco-echo-server \
    --requires-client-authorization
```

In order to authenticate with the with the service, we need to create an SAS policy. For the sake of this demonstration, we create a single policy and corresponding key which can both send (i.e. connect) and listen (i.e. forward). If you want to be a bit more security-conscious, you can create different rules for listening and sending. The list command will output a string starting with `SAS_POLICY` which we will use below to authenticate our tunnels.

```bash
az relay hyco authorization-rule create \
    --resource-group rg-access-bridge \
    --namespace-name relns-access-bridge \
    --hybrid-connection-name hyco-echo-server \
    --name sas-echo-server-bittrance \
    --rights Listen Send

az relay hyco authorization-rule keys list \
    --resource-group rg-access-bridge \
    --namespace-name relns-access-bridge \
    --hybrid-connection-name hyco-echo-server \
    --name sas-echo-server-bittrance \
    --query primaryConnectionString \
    --output tsv
```

With this, we can now spin up an Azure Container instance that attaches to the vnet and forwards traffic to the target resource. ACI appears not to want to auto-create more than one subnet for itself in a particular vnet, so in this demonstration we put the bridge on the same subnet as the echo server.

```bash
az container create \
    --resource-group rg-access-bridge \
    --image bittrance/azbridge:latest \
    --command-line '/app/azbridge -R hyco-echo-server:10.0.0.1:8080 -x SAS_POLICY' \
    --name aci-access-bridge \
    --vnet VNET_ID \
    --subnet snet-access-bridge
```

### Testing the relay

We can now use Docker to establish a connection with the Hybrid connection and thence onto the target service. This example binds port 8080 on the local machine and forwards it to the echo server:

```bash
docker run -p 127.0.0.1:8080:8080 --rm -it bittrance/azbridge \
    -L 0.0.0.0:8080:hyco-echo-server \
    -x 'SAS_POLICY'
```

Once azbridge has acquired a tunnel, we can test our access to the target resource.

```bash
curl -v http://localhost:8080/
```

If everything works, you will see something like this:

```
*   Trying 127.0.0.1:8080...
* Connected to localhost (127.0.0.1) port 8080 (#0)
> GET / HTTP/1.1
> Host: localhost:8080
> User-Agent: curl/7.78.0
> Accept: */*
>
* Mark bundle as not supporting multiuse
< HTTP/1.1 200 OK
< X-App-Name: http-echo
< X-App-Version: 0.2.3
< Date: Wed, 29 Sep 2021 11:29:42 GMT
< Content-Length: 12
< Content-Type: text/plain; charset=utf-8
<
Hello World
* Connection #0 to host localhost left intact
```

## Teardown

Don't forget to tear down your resources.
```bash
az group delete --name rg-access-bridge
```

## Finally

Please note that a Hybrid connection costs about USD 13/month and each listener costs another USD 10/month and that traffic above 5 GB is charged at a steep USD 1/GB. For most cases, these numbers are small, but if you are really shoestring, they may be a concern and you may want to create and tear down the setup between usage.

Latency is not great with this setup. After all, there may be upwards of six different TCP sessions involved (client -> Docker proxy -> azbridge -L -> Azure Relay inbound -> Azure Relay outbound -> (Docker proxy -> ?) azbridge -R -> Service). Massive data transfers and highly interactive web apps are likely to suffer accordingly. Still, as a service hatch, it works quite well.

With this setup, certificates will not be valid since the remote end will claim to be whatever service we are trying to connect to, but on the local machine, we will of course connect to localhost. Your driver/lib/browser will have to be persuaded not to verify the certificate.

Also, while this procedure is most useful on Azure, there is nothing to stop you from using it with any other cloud provider or even on-premise access. Wherever you can run `azbridge -R`, this process should be usable.