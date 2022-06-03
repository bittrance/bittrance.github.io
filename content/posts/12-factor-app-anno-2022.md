---
title: Twelve-factor app anno 2022
description: >
  The Twelve-factor app methodology turns 10. This blog posts re-evaluates 
  the original factors against a decade of experience with 
  software-as-a-service development and the maturing of serverless 
  development.
authors: andersq
tags:
  - 12-factor
  - devops
  - serverless
keywords:
  - 12-factor
  - devops
  - serverless
---

[The Twelve-factor app](https://12factor.net/) is a methodology for building software-as-a-service apps that was first formulated by developers associated with Heroku. It's been ten years since the first presentation of this methodology. Despite the criticism that it is only applicable to Heroku and similar webapp services, it remains a relevant yard stick for software-as-a-service development. Some of its tenets have been incorporated into Docker and thence into OCI, effectively making them the law of container-land. This blog post looks at each of the twelve factors and tries to evaluate whether they remain relevant or whether they need updating.

<!-- truncate -->

In one respect, the criticism of being Heroku-centric is relevant. Heroku (and Google App Engine and similar services) offer a single packaging model which we today might refer to as "IaC-less": you provide an HTTP server and Heroku runs it for you (or a war file in Google App Engine's case). Any non-trivial software-as-a-service offering require composing many apps into a service, where each has a distinct role: authentication, caching, API, serving static files, et.c., with some infrastructure-as-code that describes how these apps are exposed and interconnect. We end up with an "app" level and a "service" level and we have to be careful when considering the original text, since it talks only about the app level, but some of its concerns now reside at the service level.

## Factor I: Codebase

> A codebase is any single repository. [...] The codebase is the same across all deploys, although different versions may be active in each deploy.

This factor is first and foremost an argument for expressing as much as possible of your service as code that can be put under version control. This was probably already a strawman attack ten years ago. Nevertheless, the latest incarnation of mandating version control is [GitOps](https://www.weave.works/technologies/gitops/), namely the idea that your infrastructure maintains itself by reading your version control system and automatically applies changes as necessary - remarkably similar to the original Heroku model.

> There is always a one-to-one correlation between the codebase and the app. If there are multiple codebases, it’s not an app – it’s a distributed system. Each component in a distributed system is an app, and each can individually comply with twelve-factor.

This part of the factor remains relevant at the app level, but modern public cloud providers' infrastructure-as-code tooling and platforms like Kubernetes allow us to describe a set of apps and their supporting resources (e.g. secrets) as one package or service. Some organizations separate infrastructure-as-code and app code into separate repositories while others keep the app and its supporting IaC together; neither of these can be said unconditionally to be best practice. Similarly, [Single-page apps](https://en.wikipedia.org/wiki/Single-page_application) are often deployed to a [Content Distribution Network](https://www.cloudflare.com/learning/cdn/what-is-a-cdn/), while their backend may be deployed to a public cloud provider or to a [Kubernetes](https://kubernetes.io/) cluster. Whether these should be kept in the same repository or in different repositories depends on how tightly coupled they are.

The 2022 developer considers the relationship between repositories and artifacts carefully. Pertinent aspects include:

- branching strategy
- continuous integration completeness and run times
- continuous delivery pipelines
- infrastructure-as-code maintainability
- configuration management
- automated deployments

Expect to reorganize your sources as your apps evolve.

## Factor II: Dependencies

> A twelve-factor app never relies on implicit existence of system-wide packages. It declares all dependencies, completely and exactly, via a dependency declaration manifest. [...] Twelve-factor apps also do not rely on the implicit existence of any system tools.

In the container and function-as-a-service worlds, this factor has been elevated to fact. These execution environments provide virtually no implicit dependencies.

Modern apps tend to have more than one dependency declaration manifest, namely its project manifest(s) (e.g. `go.mod` or `package.json`) and a `Dockerfile`. A consequence of this factor is that you should use Docker base images that are as bare-bone as they come, forcing the explicit installation of supporting libs and tools.

The up-to-date interpretation of this factor is that upgrading dependency versions should always be a conscious action. This slightly shifts the interpretation of the original factor's "exactly". The various ecosystems and tool chains (maven, npm, cargo, et.c.) work differently in when they resolve dependencies. Some resolve dependencies when the developer performs a build and some require an explicit "upgrade" operation to change what goes into a build. It is therefore vital to have a codified workflow for updating dependencies. For example, when using Node.js and npm, a developer should normally do [npm ci](https://blog.npmjs.org/post/171556855892/introducing-npm-ci-for-faster-more-reliable) and only use the traditional `npm install` (or `npm update`) when the intent is to modernize dependencies.

> it uses a dependency isolation tool during execution to ensure that no implicit dependencies “leak in” from the surrounding system. The full and explicit dependency specification is applied uniformly to both production and development.

One of the innovations introduced by Docker is that this factor is enforced already at build time, making it easy to ensure uniformity across dev and prod. Run your automated tests with the built container and there is very little space for execution environment differences.

## Factor III: Config

> An app’s config is everything that is likely to vary between deploys (staging, production, developer environments, etc). The twelve-factor app stores config in environment variables. Env vars are easy to change between deploys without changing any code; unlike config files, there is little chance of them being checked into the code repo accidentally; and unlike custom config files, [...] they are a language- and OS-agnostic standard. [...] A litmus test for whether an app has all config correctly factored out of the code is whether the codebase could be made open source at any moment, without compromising any credentials.

This factor remains mostly relevant as written, but there are some nuances to consider.

Infrastructure-as-code tools like Terraform allows us to create files and database entries. Kubernetes allows us to create [ConfigMaps](https://kubernetes.io/docs/concepts/configuration/configmap/) which will be made available to a container as a file. In both cases, the source configuration can be put under version control, any manual edits will be overwritten on the next deploy and the mechanism is easy for a developer to emulate locally. Thus, they achieve the same result as using environment variables by different means.

Also, while environment variables are operations-friendly, they are problematic when writing tests, since they are global state. In ecosystems that default to parallel test execution (e.g. Rust) environment variables cannot be used. Thus, while an environment variable remains the preferred way to accept simple configuration values, a 12-factor app should convert them to internal state as early as possible.

This factor is now obsolete in one respect. As much as possible, secrets (passwords, private keys, et c) should be stored using a secrets management system such as Hashicorp Vault or Azure Key Vault. Particularly where we can rely on the infrastructure to authenticate the calling process (e.g. via a Kubernetes service account) access to secrets will not directly require credentials. The existence of the secret is under version control, but the actual secret content is immaterial.

Furthermore, with platforms such as Kubernetes, service discovery means that some aspects need no configuration at all. Additionally, some forms of configuration can better be managed as references between IaC-controlled resources, which removes them from direct configuration management consideration.

## Factor IV: Backing services

> A backing service is any service the app consumes over the network as part of its normal operation. The code for a twelve-factor app makes no distinction between local and third party services. To the app, both are attached resources, accessed via a URL or other locator/credentials stored in the config. [...] Resources can be attached to and detached from deploys at will.

This factor remains relevant as written. Its current iteration is sometimes referred to as ["API first"](https://swagger.io/resources/articles/adopting-an-api-first-approach/) which can be described as the principle that all services you create should be able to act as a backing service. More generally, with the advent of [Zero Trust](https://www.crowdstrike.com/cybersecurity-101/zero-trust-security/) and the proliferation of cloud services, the logical end result of this factor is that any service can interact with any other service on the planet.

Even your orchestration platform itself is a backing service, not just the services that run inside it. A service can leverage the Kubernetes control plane to run a one-off job or provision cloud resources to serve a new customer.

The original text focuses a lot on relational databases. It is worth pointing out that you can create a service which scalably serves long-lived state without violating the 12 factors: as long as there is a procedure to claim or negotiate access to a particular shard of the state (e.g. backups stored in an Azure storage account), the actual process can remain stateless. Contemporary thinking in this matter is still coloured by software that predates the cloud era (e.g. MySQL, Elasticsearch, RabbitMQ). For an example of what is possible in 2022, we can look at [Loki](https://github.com/grafana/loki).

## Factor V: Build, release, run

> The twelve-factor app uses strict separation between the build, release, and run stages. The build stage is a transform which converts a code repo into an executable bundle known as a build.

This factor is more or less a prerequisite for developing software-as-a-service in 2022, but we need to complement this factor with a requirement for automating these stages. The maturing of CI/CD software-as-a-service providers such as GitHub, [ACR Tasks](https://docs.microsoft.com/en-us/azure/container-registry/container-registry-tasks-overview) and [Circle CI](https://circleci.com/) means that it is now relatively easy to automate this process.

Typically, the build stage will push a container image to some container registry or upload a function-as-a-service zip files to cloud storage.

> The release stage takes the build produced by the build stage and combines it with the deploy’s current config

The normal practice today is for a pipeline to push the release to the runtime environment. This is very useful early in the lifecycle of an app since the release process typically evolves with the app. To achieve stronger separation between the build and release, you might want to consider going GitOps. In Kubernetes-land, you would use a service such as [Flux](https://fluxcd.io).

## Factor VI: Processes

> Twelve-factor processes are stateless and share-nothing. Any data that needs to persist must be stored in a stateful backing service.

This factor remains relevant as written. In the world of REST APIs, this effectively means that we should hold no domain state in memory between HTTP requests - it should always be handed over to a caching service. This is the main enabler for scale-out in a software-as-a-service.

Adhering to this rule is also a good way to avoid memory leaks, which tend to plague garbage-collected ecosystems such as Java, Node, Python and Ruby. You will still get leaks after you out-source your caching to Redis, but it will be much easier to measure and the incitement to properly architecture the caching is stronger.

## Factor VII: Port binding

> The twelve-factor app is completely self-contained and does not rely on runtime injection of a webserver into the execution environment to create a web-facing service. The web app exports HTTP as a service by binding to a port, and listening to requests coming in on that port.

This factor is now standard in containerized scenarios. Widespread adoption of port binding has enabled a whole ecosystem of supporting proxies (e.g. [Envoy](https://www.envoyproxy.io/), [Traefik](https://traefik.io/), [Toxiproxy](https://github.com/Shopify/toxiproxy)) which (ironically) means that a typical app today is often not self-contained, but depends on other containerized apps to perform e.g. authentication and tracing. This is a improves [separation of concerns](https://deviq.com/principles/separation-of-concerns) and consequently, in 2022 we consider this factor at the service level.

> The port-binding approach means that one app can become the backing service for another app, by providing the URL to the backing app as a resource handle in the config for the consuming app.

The original text focuses on network protocols such as HTTP and [XMPP](https://xmpp.org/extensions/). In order to become a backing service in 2022, the app should also adhere to an [API contract](https://apievangelist.com/2019/07/15/what-is-an-api-contract/) of some sort, defining the backing service's intended role.

Many developers implicitly assume that using high-level protocols like HTTP incurs latency. The overhead of a REST call over a local network (e.g. within a cloud provider) is typically 2-4 ms so you need to get a significant number of non-parallelizable requests before this overhead is noticeable over RDBMS operations and latency towards the consumer.

## Factor VIII: Concurrency

> In the twelve-factor app, processes are a first class citizen. Processes in the twelve-factor app take strong cues from the unix process model for running service daemons. [...] This does not exclude individual processes from handling their own internal multiplexing. But an individual VM can only grow so large (vertical scale), so the [app] must also be able to span multiple processes running on multiple physical machines.

This factor is now more or less written into law. Function-as-a-service platforms typically provide transparent horizontal scaling. In Kubernetes deployments you just give the number of pods you expect.

The mechanics of horizontal scalability is thus addressed in 2022. The central challenge for any successful app is to distribute work across multiple processes and backing services in such a way that it actually achieves meaningful horizontal scalability. The typical RDBMS-backed web app usually has its scalability completely constrained by its backing database, forcing vertical scaling of the RDBMS - a matter of allocating additional CPUs. This is often expensive and tends to yield only marginal improvement.

Is short, horizontal scalability is much preferable over vertical scalability but it is strictly a result of software architecture. It is therefore vital to identify early those apps that will actually require significant scale-out so that they can be properly architected.

Despite the dominance of the serverless paradigm in the software-as-a-service realm, there is still an overhang from the era of vertical scaling which the Twelve Factor App tries to break with. For example, the virtual machines of Java, Node, Python and Ruby maintain large volumes of reflection metadata and are very reluctant to de-allocate memory, leading to significant inefficiency on scale-out. A new generation of ecosystems, lead by Go and Rust, are more frugal in this respect.

## Factor IX: Disposability

> The twelve-factor app’s processes are disposable, meaning they can be started or stopped at a moment’s notice. [...] Processes should strive to minimize startup time. [...] Processes shut down gracefully when they receive a SIGTERM signal, [...] allowing any current requests to finish. [...] A twelve-factor app is architected to handle unexpected, non-graceful terminations.

This factor remains relevant as written and remains nearly as elusive today as it was ten years ago. For example, the HTTP server included in Node.js does [not by default perform graceful shutdown](https://blog.dashlane.com/implementing-nodejs-http-graceful-shutdown/) (see also [nodejs issue 2642](https://github.com/nodejs/node/issues/2642)).

Fulfilling this factor on an existing code base is much harder than it sounds. It means mapping all the (often implicit) state machines involved in the app and coordinating them so that there are no undefined transitions. For example, the database connection pool must be ready before we bring up our HTTP listener and must not shut down until the HTTP listener is down _and_ all in-flight HTTP requests are done. Workers must "hand back" in-progress work items so that replacement workers do not have to wait for timeout to process those work items. This difficulty is compounded with distributed systems as a conceptual "transaction" may span more than one app or backing service (e.g. writing to file storage and sending mail), requiring [two-phase commit](https://en.wikipedia.org/wiki/Two-phase_commit_protocol) semantics.

This factor is nevertheless the key to the always-on experience that we take for granted in large cloud services. A service with working graceful shutdown and comprehensive health checks can be updated at any time and builds developer and operations confidence. This can result in significant productivity gains, particularly when combined with automated testing.

## Factor X: Dev/prod parity

> Historically, there have been substantial gaps between development [...] and production [...], the time gap, the personnel gap [and] the tools gap. [...] The twelve-factor app is designed for continuous deployment by keeping the gap between development and production small.

This factor is now colloquially known as [DevOps](https://aws.amazon.com/devops/what-is-devops/) and remains as relevant as ever, but has still not established itself fully in software-as-a-service development: many production environments are hard to squeeze onto a developer's laptop. Generally speaking, the public cloud providers put too little effort into supporting development use cases for their services. Kubernetes goes furthest in this respect: [Kind](https://kind.sigs.k8s.io/) deserves mentioning for its heroic effort to achieve a dev-friendly, multi-node Kuberentes cluster using only Docker.

Docker has introduced a borderland where it is possible to develop a container using just Docker Engine for dev environment and still be reasonably confident that it will execute properly in e.g. Kubernetes. Still, some provisioning of backend services is still needed and time is wasted maintaining two different sets of instrumentation. For example, apps often have a Docker Compose file to get developers started, and a Kubernetes manifest for test/prod. When these desynch, nasty surprises can occur at deployment.

> The twelve-factor developer resists the urge to use different backing services between development and production

Many full-stack development setups includes "dev" servers whose role is to automatically reload the app as its source code change. Similarly, tools like Docker Desktop and [Tilt](https://tilt.dev/) provide capabilities and environments that are subtly different from e.g. Azure Container instances or Kubernetes. All these will color developers' choices and risk introducing issues that will not be discovered until late in the process.

The 2022 developer considers both developer experience, continuous integration/delivery/deployment and operability when choosing tools.

## Factor XI: Logs

> Logs provide visibility into the behavior of a running app. [...] A twelve-factor app never concerns itself with routing or storage of its output stream. It should not attempt to write to or manage logfiles. Instead, each running process writes its event stream, unbuffered, to stdout. [...] Destinations are not visible to or configurable by the app, and instead are completely managed by the execution environment.

Interestingly, this factor does not actually advise on the use of logging. Rather it treats them much as pre-contraceptive times viewed children: as something that inevitably accumulates as a result of marriage.

The factor should thus be updated to mandate that an app should be "observable", meaning that it should volunteer information on its performance and behavior. We normally break this down into logging, metrics, tracing and audit trails. The relative merit of these differ greatly between apps, but all apps should have an observability strategy. Often, the need changes as an app evolve: early in the lifecycle, logging may dominate, but as usage grows, focus shifts to metrics. The apps in a service are considered as a unit and typically provide different observability needs.

## Factor XII: Admin processes

> One-off admin processes should be run in an identical environment as the regular long-running processes of the app. They run against a release, using the same codebase and config as any process run against that release.

This factor captures a practice that is common in the [Rails](https://rubyonrails.org/) and [Drupal](https://www.drupal.org/) ecosystems, among others. These are based on interpreted languages where it is relatively easy to give scripting or interactive access to the app's internals: the main reason is to ensure that database access occurs using the same models that are used during runtime. In compiled languages, this requires a modularization (e.g. making the data model a separate library) that would complicate development.

However, the factor has two underlying principles which holds even today. First, that tools used to administer an app should be versioned and released with the same discipline as your app is. Second, that operating and upgrading an app is part of its ordinary usage and there is nothing strange with adding endpoints for internal administrative use. Put differently, at least factors I - IV should apply to the app's tooling just as it does to the app itself.

## What else is there?

Various additional factors have been proposed over the years, for example [Beyond the Twelve-factor app](https://raw.githubusercontent.com/ffisk/books/master/beyond-the-twelve-factor-app.pdf) and [Seven missing factors from the Twelve-factor app](https://www.ibm.com/cloud/blog/7-missing-factors-from-12-factor-applications). The appeal of the original methodology springs from the universality of its factors. These contributions have relevance, but often only for a subsection of all apps that could (should) strive to live up to the original twelve factors. However, there is two aspects that are clearly missing: security and automated testing.

The good people at WhiteHat Security has written a good analysis called [Security and the twelve-factor app](https://www.devopsdigest.com/security-and-12-factor-app-step-1) which analyses each factor from a security perspective and provides recommendations for securing your app. Their main point is that rather than being an additional factor, security needs to permeate all the twelve factors. The charge that security is underrepresented in the original methodology har merit, and the 2022 developer no longer has the luxury of treating security as an afterthought.

Finally, much of the benefit of adhering to the methodology is lost without extensive and automated testing. Version control hygiene and horizontal scalability matters little if apps break as soon as they are deployed. Ten years ago, there was still a discussion about whether writing programmatic tests was worth the effort. That discussion is now settled and in 2022, the discussion is about what the automated testing strategy should look like for a particular app or service. The discerning developer considers:

- when to apply unit, component, integration and end-to-end tests at app and/or service level
- when to use in-memory implementations of backing services
- what long-lived testing environments are needed
- how much automated [static analysis](https://en.wikipedia.org/wiki/Static_program_analysis) to include

These additions notwithstanding, the Twelve-factor app methodology remains remarkably relevant today. All developers producing software-as-a-service offerings can benefit from adhering to its factors.
