---
title: "Creating a simple build pipeline with Tekton"
date: 2021-04-19T21:48:09+02:00
---
> *Note: This article uses lots of Tekton beta features. Should you discover it in 6 months from now, it is highly likely that something has changed.*

[Tekton](https://tekton.dev) is a continuous integration and delivery platform on top of Kubernetes. It is still under intensive development, but is shaping up to be a real Argo-killer. This blog post will explore Tekon's feature set by way of converting a GitHub Actions workflow into a Tekton Pipeline. This is meant to be the first of a series of Tekton-centered blog posts.

This post is laid out as a step-by-step instruction. If you want to follow along and do not already have a suitable Kubernetes cluster on hand, you can [install Kind](/posts/install-kind) which will give you a good starting point.

## The Tekton data model

Tekton has two main primitives: [Pipelines](https://tekton.dev/docs/pipelines/pipelines/) and [Tasks](https://tekton.dev/docs/pipelines/tasks/). A pipeline is akin to a GitHub Actions workflow and tasks are similar to the steps in a job in that workflow. Pipelines and Tasks are represented by custom resources. We also need a third Tekton concept to complete this post: each task is technically one or more container executions and as any container, it has an ephemeral file system, so we need a directory that can be shared between tasks. Such a directory is called a [Workspace](https://tekton.dev/docs/pipelines/workspaces/).

Tekton will have a "marketplace" called [Tekton Hub](https://hub.tekton.dev/) where standard tasks and pipelines can be published. The site is operational today, if still a bit rough.

This model is already a little more light-weight than GitHub actions and Bitbucket Pipelines. However, reuse in CI/CD is often about repeating fragments of pipelines, which those models do not support well. Tekton will introduce the ability to have [sub-pipelines](https://github.com/tektoncd/community/blob/main/teps/0056-pipelines-in-pipelines.md) which is likely to make reuse across pipelines much more practical.

## Installing Tekton

As a good citizen of Kubernetes-land, Tekton is distributed as a set of manifests. Let's install Tekton and its user interface into the test cluster.
```bash
kubectl apply -f https://storage.googleapis.com/tekton-releases/pipeline/previous/v0.23.0/release.yaml
kubectl apply -f https://github.com/tektoncd/dashboard/releases/download/v0.16.0/tekton-dashboard-release.yaml
```

We want to access the Tekton UI during this post. If your test cluster does not expose services to the outside world, you can install an ingress:

{{% code file="/static/simple-pipeline-with-tekton/tekton-dashboard-ingress.yaml" language="yaml" %}}

There is also a [Tekton cli](https://github.com/tektoncd/cli) tool. However, it does not provide a lot of benefit for this instruction, so I wil skip it and use kubectl directly.

## Let's write a pipeline

We need a non-trivial project so that we can explore difficulties that tend to arise when writing build pipelines. This post will try to recreate the [striv master pipeline](https://github.com/bittrance/striv/blob/cfac501a45db4c58ae526a3e58242c6245305624/.github/workflows/default-test.yml). Striv is a typical webapp with a Vue-based frontend and a Python-based backend which keeps its state in MySQL or Postgres - in other words a bog-standard single-page application (SPA). Its main Github action contains the following steps:

1. Run node unit tests for the frontend (`npm run test:unit`)
1. Run python unit and integration tests for the backend (`pytest`)
1. Build and push a Docker image (`docker build` and `docker push`)

## Our first task - checkout

The Tekton pipeline is a Kubernetes custom resource with `kind: Pipeline`. It describes a series of tasks to perform. The first task our pipeline will have to do is to check out the source code that we want to test and build. Tekton provides a standard task for checking out sources from GitHub that we can install in our cluster:
```bash
kubectl apply -f https://raw.githubusercontent.com/tektoncd/catalog/main/task/git-clone/0.3/git-clone.yaml
```

This allows us to start a new pipeline:
```yaml
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: deploy-striv
spec:
  workspaces:
    - name: sources # The invoker will need to provide this workspace
  tasks:
    - name: checkout
      taskRef:
        # The standard task we installed above
        name: git-clone
        kind: Task
      workspaces:
        - # This task requires us to provide a workspace where the code
          # should be checked out. Here we tell Tekton to provide our 
          # "sources" workspace as the task's "output" workspace.
          name: output
          workspace: sources
      params:
        - name: url
          value: https://github.com/bittrance/striv
        - # In order for this blog post to be repeatable, we refer to a 
          # particular revision.
          name: revision
          value: kubernetes
```
The above example builds a specific branch. That is not so useful in real life, but we will get back to parameterizing this in a later blog post.

## Run frontend unit tests

The next step in our pipeline is to run the frontend unit tests. This too can user a standard task that we can install in our cluster:
```bash
kubectl apply -f https://raw.githubusercontent.com/tektoncd/catalog/main/task/npm/0.1/npm.yaml
```

With this installed, we can add two tasks to our pipeline:
```yaml
    - name: install-frontend-dependencies
      runAfter: [checkout]
      taskRef:
        name: npm
        kind: Task
      workspaces:
        - name: source
          workspace: sources
      params:
        - name: IMAGE
          value: node:15.3
        - name: ARGS
          value: [clean-install]
    - name: test-frontend
      runAfter: [install-frontend-dependencies]
      taskRef:
        name: npm
        kind: Task
      workspaces:
        - name: source
          workspace: sources
      params:
        - name: IMAGE
          value: node:15.3
        - name: ARGS
          value: [run, test:unit]
```

By default, all pipeline tasks are run in parallel, but running tests cannot be performed until the source code is checked out, so we need to be able to define task dependencies. This is done using the `runAfter` directive. This gives us a simple DAG:
```
checkout <- install-frontend-dependencies <- test-frontend
```

## Testing the pipeline

There are more steps to complete before we are done, but we should make sure our pipeline works so far!

When you want to invoke a pipeline, you create another custom resource, called a PipelineRun. Executing the pipeline manually shows how Tekton works; in the really real world, this should be performed automatically on pushing to the Git repository, but more on that in a later blog post.
Write this to a file and apply it to your Kubernetes cluster:

{{% code file="/static/simple-pipeline-with-tekton/striv-pipelinerun.yaml" language="yaml" %}}

Navigating to the [Tekton UI](http://tekton.lvh.me), if all went well, the run will be green:

![deploy-striv run successful](/simple-pipeline-with-tekton/deploy-striv-run-1.png "deploy-striv run successful")

Each task run by the pipeline is just a pod. We can use `kubectl get pods` to inspect them:
```
NAME                                                              READY   STATUS      RESTARTS   AGE
deploy-striv-run-1-checkout-hmvxb-pod-fqx8b                       0/1     Completed   0          45s
deploy-striv-run-1-install-frontend-dependencies-2q6nn-po-dzfpb   0/1     Completed   0          31s
deploy-striv-run-1-test-frontend-c2779-pod-z642x                  0/1     Completed   0           8s
```
You can similarly request logs and terminate errant tasks.

## Next step

This blog post has shown how to get started with using Tekton for continuous integration. There is more to do on the Striv pipeline and there are many more things Tekton can do, but that is for a future blog post.