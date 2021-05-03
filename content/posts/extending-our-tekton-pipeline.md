---
title: "Extending Our Tekton Pipeline"
date: 2021-05-04T14:31:00+02:00
---
This blog post continues my exploration of [Tekton](https://tekton.dev). In the [first article](/posts/extending-our-tekton-pipeline/) we created a simple pipeline that ran our frontend tests. It will show how to set up pods to support tests. This also serves as a sketch of how to interact with Kubernetes in general.

## Running backend unit tests

The Striv backend tests are mostly straight-forward unit tests, but includes some tests that test the interaction with supported databases, namely MySQL and Postgres. It does not, however provide [pytest "marks"](https://docs.pytest.org/en/stable/mark.html) to select only the unit test. Instead, there is a JSON config file called `testing-databases.conf` which controls what databases are tested. There is a standard task that allows us to write an arbitrary file to disk:

```bash
kubectl apply -f https://raw.githubusercontent.com/tektoncd/catalog/main/task/write-file/0.1/write-file.yaml
```

Writing a simple "unit test only" `testing-databases.conf` looks like this:
```yaml
    - name: write-testing-databases-conf
      runAfter: [checkout]
      taskRef:
        name: write-file
        kind: Task
      workspaces:
        - name: output
          workspace: striv-workspace
      params:
        - name: path
          value: testing-databases.json
        - name: contents
          value: '[["sqlite", {"database": ":memory:"}]]'
```

We can now use the standard [pytest task](https://hub.tekton.dev/tekton/task/pytest) to run our tests.
```bash
kubectl apply -f https://raw.githubusercontent.com/tektoncd/catalog/main/task/pytest/0.1/pytest.yaml
```

The actual pytest task is quite straight-forward:
```yaml
    - name: test-backend
      runAfter: [write-testing-databases-conf]
      taskRef:
        name: pytest
        kind: Task
      workspaces:
        - name: source
          workspace: striv-workspace
      params:
        - name: PYTHON
          value: "3.7"
        - name: REQUIREMENTS_FILE
          value: requirements-dev.txt
```

## Integration tests

However, we do want to run those MySQL and Postgres tests! So how are we going to get a Postgres **and** a MySQL up? There are at least two different strategies to provide supporting resources to your tests:

1. Use sidecars. Tekton has explicit support for running sidecars to Tasks. 
1. Deploy Kubernetes resources (i.e. pods) to support our integration tests.

Sidecars sound tempting, but has two problems. First, they can only be used by task definitions and not from a pipeline. That would mean that we could not use the standard pytest task above, but would have to define our own task that includes the sidecars. Second, we are likely to run into readiness issues and would have to include an explicit script step to wait for the database sockets to become available. These issues are surmountable and had I been at work, I would probably have taken this route.

But this is a blog post, and we are here to learn Tekton. Deploying a set of Kubernetes resources to provide those databases is a much more generally applicable technique, so let's see what it takes to do that!

### Deploying the resources

We can use [kubernetes-actions](https://hub.tekton.dev/tekton/task/kubernetes-actions) to deploy resources to a Kubernetes cluster.
```bash
kubectl apply -f https://raw.githubusercontent.com/tektoncd/catalog/main/task/kubernetes-actions/0.2/kubernetes-actions.yaml
```

Once we have credentials available, we can load the task to set up a service and pod with mysql. For the sake of simplicity, I'm inlining the manifest in the task. I could equally set it up as a ConfigMap or mount the `striv-workspace` and reference a file there.
```yaml
    - name: deploy-test-dependencies
      taskRef:
        name: kubernetes-actions
        kind: Task
      params:
        - name: script
          value: |
            kubectl apply -f - <<EOF
            apiVersion: v1
            kind: Service
            metadata:
              # Unique name so it won't collide with concurrent runs
              name: mysql-$(context.pipelineRun.uid)
              labels:
                # So we can delete both this and Postgres
                test-dependencies: $(context.pipelineRun.uid)
            spec:
              ports:
                - name: mysql
                  port: 3306
                  targetPort: mysql
              selector:
                mysql: $(context.pipelineRun.uid)
            ---
            apiVersion: v1
            kind: Pod
            metadata:
              name: mysql-$(context.pipelineRun.uid)
              labels:
                # So we can delete both this and Postgres
                test-dependencies: $(context.pipelineRun.uid)
                # So the mysql service can find only this pod (and not Postgres)
                mysql: $(context.pipelineRun.uid)
            spec:
              containers:
                - name: mysql
                  image: mysql:5.6
                  ports:
                    - name: mysql
                      containerPort: 3306
                  env:
                    - name: MYSQL_ROOT_PASSWORD
                      value: Cmdu1OG2vLfL
                    - name: MYSQL_DATABASE
                      value: striv_test
            EOF
            kubectl wait pods -l mysql=$(context.pipelineRun.uid) --timeout=30s --for condition=ready
```

Note that this task has no `runAfter` key. Test resource creation will begin immediately when the pipeline is run, which means that it is very likely that they will be ready by the time we run the backend tests. Likely enough that we can ignore the issue of checking for readiness. One reason for not solving this issue is that the standard MySQL container restarts mysql once as part of its setup. This means that even succesfully connecting to port 3306 will not guarantee that MySQL is ready to serve our requests.

A very similar entry will be needed for Postrgres 11, but that is left as an exercise to the reader.

The configuration file injected by `write-testing-databases-conf` will need updating to include MySQL:
```yaml
    - name: write-testing-databases-conf
      runAfter: [checkout]
      taskRef:
        name: write-file
        kind: Task
      workspaces:
        - name: output
          workspace: striv-workspace
      params:
        - name: path
          value: testing-databases.json
        - name: contents
          value: |
            [
              ["sqlite", {"database": ":memory:"}],
              [
                "mysql",
                {
                  "user": "root",
                  "password": "Cmdu1OG2vLfL",
                  "host": "mysql-$(context.pipelineRun.uid)",
                  "database": "striv_test",
                  "create_database": true
                }
              ]
            ]
```

### Tearing down the testing resources

Once the pipeline is finished, we want to drop the database pods again. We could just add a normal task with `runAfter: ["test-backend"]`, but that might trip us up in the future when we add more tasks to our pipeline. Better to drop them as the very last activity. Tekton provides a `finally` section for this scenario where you can add tasks that should be run at the end of the pipeline, regardless of success.
```yaml
  finally:
    - name: delete-test-dependencies
      taskRef:
        name: kubernetes-actions
        kind: Task
      params:
        - name: script
          value: |
            kubectl delete pods -l test-dependencies=$(context.pipelineRun.uid) --timeout=30s
            kubectl delete services -l test-dependencies=$(context.pipelineRun.uid) --timeout=30s
```

## Running the pipeline

If we were to run the pipeline now, the `deploy-test-dependencies` task would fail with a "Forbidden" error message, because the task does not have permission to interact with our Kubernetes cluster. Tekton pipelines normally use the "default" service account, which has very few permissions. We can assign a different account, either to the whole pipeline, or to an individual task. This pipeline draws code from several different sources (e.g. Docker Hub, GitHub, PyPI and npmjs.org) so there is plenty opportunity to sneak in malevolent code. It therefore makes sense to grant permissions only to the necessary tasks.

First, we need to create this service account and give it some permissions. For the sake of completeness, we create a role for it as well; in a production scenario, you probably have a reasonalbe group or role available already.
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: test-dependencies-deployer
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: test-dependencies-deployer
rules:
  - apiGroups: [""]
    resources: [pods, services]
    verbs: [create, delete, get, list, watch]
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: RoleBinding
metadata:
  name: test-dependencies-deployer
subjects:
  - kind: ServiceAccount
    name: test-dependencies-deployer
roleRef:
  kind: Role
  name: test-dependencies-deployer
  apiGroup: rbac.authorization.k8s.io
```

Finally, we need to tell Tekton that the two test dependency tasks should use this service account. This is part of the PipelineRun resource and looks like this:
```yaml
apiVersion: tekton.dev/v1beta1
kind: PipelineRun
metadata:
  name: deploy-striv-run-2
spec:
  pipelineRef:
    name: deploy-striv # We want to run this pipeline
  serviceAccountNames:
    # Tasks to set up and tear down test dependencies need to use a service account
    - taskName: deploy-test-dependencies
      serviceAccountName: test-dependencies-deployer
    - taskName: delete-test-dependencies
      serviceAccountName: test-dependencies-deployer
  workspaces:
    - # This describes how to provide the workspace that the pipeline requires
      name: striv-workspace
      volumeClaimTemplate:
        spec:
          accessModes:
            - ReadWriteOnce
          resources:
            requests:
              storage: 1Gi
```

The [updated pipeline](/extending-our-tekton-pipeline/deploy-striv-extended.yaml) is here if you want to test it yourself.

At last, we can apply this pipeline run and get a successful test run:

![deploy-striv run successful](/extending-our-tekton-pipeline/deploy-striv-run-2.png "deploy-striv run successful")

## Next step

We now have a complete testing pipeline. To make this a complete continuous delivery pipeline, it needs to build and deploy the Docker container that is the main artifact. However, that should happen only from some branches, so first we need to set up a trigger so that GitHub informs us when and what changes are pushed to the striv repository. More on this in the next blog post.