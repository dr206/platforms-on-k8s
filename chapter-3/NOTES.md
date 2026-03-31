# DevOps Book Club - Week 3

<!-- TOC -->
* [DevOps Book Club - Week 3](#devops-book-club---week-3)
  * [What does it take to delivery cloud-native applications continuously?](#what-does-it-take-to-delivery-cloud-native-applications-continuously)
  * [Service Pipelines](#service-pipelines)
  * [Conventions that will save you time](#conventions-that-will-save-you-time)
  * [Service pipeline structure](#service-pipeline-structure)
  * [Service pipelines in action](#service-pipelines-in-action)
    * [Tekton](#tekton)
      * [Installing Tekton](#installing-tekton)
      * [Tasks in Tekton](#tasks-in-tekton)
      * [Pipelines in Tekton](#pipelines-in-tekton)
      * [Service pipeline examples](#service-pipeline-examples)
      * [Benefits of Tekton](#benefits-of-tekton)
    * [Dagger](#dagger)
      * [Installing Dagger](#installing-dagger)
      * [Running Dagger pipelines locally](#running-dagger-pipelines-locally)
      * [Running Dagger pipelines remotely on Kubernetes](#running-dagger-pipelines-remotely-on-kubernetes)
    * [GitHub Actions](#github-actions)
      * [Run locally](#run-locally)
  * [Linking back to platform engineering](#linking-back-to-platform-engineering)
<!-- TOC -->

## What does it take to delivery cloud-native applications continuously?

[blank]

## Service Pipelines

[blank]

## Conventions that will save you time

* Trunk-based development
* Source code and configuration management
  * One service/one repo/one pipeline
  * Monorepo with multiple services/multiple pipelines
* [Consumer-driven contract testing](https://medium.com/insiderengineering/consumer-driven-contract-testing-cdct-b6c05c18ba25)

## Service pipeline structure

* Consider external dependencies:
  * webhooks (e.g. GitHub, Slack, etc.)
  * artefact/container/Helm repositories

## Service pipelines in action

As a prerequisite for the following tutorials, you will need to have a Kubernetes Cluster available. You can create one using [KinD as we did in the Chapter 2 notes](../chapter-2/NOTES.md#running-our-cloud-native-applications).

### Tekton

#### Installing Tekton

:warning: Tekton has [migrated](https://tekton.dev/blog/2025/04/03/migration-to-github-container-registry/) its images to [GitHub Container Registry](https://github.com/orgs/tektoncd/packages).  
The Kubernetes commands to apply Tekton comes with a rewrite script to change container locations from `gcr.io` to `ghcr.io`.

```shell
curl -L https://storage.googleapis.com/tekton-releases/pipeline/previous/v0.45.0/release.yaml | ./tekton/rewrite-tekton-images.sh | kubectl apply -f -
```

```shell
curl -L https://github.com/tektoncd/dashboard/releases/download/v0.33.0/release.yaml | ./tekton/rewrite-tekton-images.sh | kubectl apply -f -
```

```shell
kubectl port-forward svc/tekton-dashboard  -n tekton-pipelines 9097:9097
```

```shell
open http://localhost:9097
```

(Optional)
```shell
brew install tektoncd-cli
```

#### Tasks in Tekton

```shell
kubectl apply -f tekton/hello-world/hello-world-task.yaml
```

```shell
kubectl get tasks
```

```shell
kubectl describe tasks hello-world-task
```

A named `TaskRun` can only be executed once, if you want to execute the same `TaskRun` multiple times, you can:
* remove the `metadata.name` field and let Kubernetes generate a random name for each execution,
* updated the `metadata.name` field with a different name for each apply.
```shell
kubectl apply -f tekton/hello-world/task-run.yaml
```

```shell
kubectl get pod | grep task
```

```shell
kubectl get taskrun
```

```shell
kubectl logs -f pod/hello-world-task-run-1-pod
```

#### Pipelines in Tekton

```shell
kubectl apply -f https://raw.githubusercontent.com/tektoncd/catalog/main/task/wget/0.1/wget.yaml
```

```shell
kubectl apply -f tekton/hello-world/hello-world-pipeline.yaml
```

```shell
kubectl get pipeline
```
Similar to above, a named `PipelineRun` can only be executed once, if you want to execute the same `PipelineRun` multiple times, you can update the name or use a randomized name.
```shell
kubectl apply -f tekton/hello-world/pipeline-run.yaml
```

```shell
kubectl get pipelinerun
```

```shell
kubectl get taskrun
```

```shell
kubectl get pod | grep pipeline
```

> [!NOTE]  
> There is no `affinity-assistant` pod created.

```shell
open http://localhost:9097/#/namespaces/default/pipelineruns/hello-world-pipeline-run-1?pipelineTask=wget&step=wget
```
#### Service pipeline examples

* [Generic Go application](./tekton/README.md#tekton-for-service-pipelines)
* [Conference application](./tekton/hello-world/README.md#tekton-for-service-pipelines)

#### Benefits of Tekton

* Runs natively in Kubernetes, no need for extra infrastructure.
* Trigger pipelines based on events (e.g. GitOps).
* Map tasks together via inputs and outputs to share data.
* A [catalog of pipelines and tasks](https://github.com/tektoncd/catalog) is available to be reused and extended:

:warning: Tekton Hub was [shutdown](https://github.com/tektoncd/hub): 
  * > The public Tekton Hub instance (hub.tekton.dev) was shut down on January 8th, 2026.

The resources have been migrated to [Artifact Hub](https://artifacthub.io/docs/topics/repositories/tekton-tasks/).

### Dagger

| Feature                  | Tekton            | Dagger                      |
|--------------------------|-------------------|-----------------------------|
| Runtime                  | Kubernetes only   | local / CI env / Kubernetes |
| Language                 | yaml / CRDs       | any language                |
| Distribution same as app | no                | yes                         |
| Caching                  | (_beta_)          | yes                         |
| Conditionals             | not really (yaml) | yes                         |

#### Installing Dagger

Ensure you have Go installed:
```shell
 go version
```

Ensure you have a container runtime (such as OrbStack) running locally:
```shell
orbctl version
```

#### Running Dagger pipelines locally

```shell
cd ../conference-application/; go mod tidy
```

The replacement is needed to run the test task, which uses a container image that is not available in `docker.io/bitnami` but is available in `docker.io/bitnamilegacy`.
```shell
cd ../conference-application/
awk '{gsub(/docker\.io\/bitnami\//, "docker.io/bitnamilegacy/"); print}' service-pipeline.go > service-pipeline.go.tmp
mv service-pipeline.go.tmp service-pipeline.go
```

```shell
cd ../conference-application/; go run service-pipeline.go build notifications-service
```

:warning: The `publish` task requires you to be logged in to the container registry where you want to publish your container images to.  
This is not covered here.

#### Running Dagger pipelines remotely on Kubernetes

[See Kubernetes section in the Dagger README](./dagger/README.md#running-your-pipelines-remotely-on-kubernetes)

### GitHub Actions

| Feature       | GitHub Actions | Tekton/Dagger |
|---------------|----------------|---------------|
| Managed infra | yes            | no            |
| Cost          | high           | not so high   |

#### Run locally

Run GitHub Actions locally with [act](https://nektosact.com/introduction.html):

```shell
brew install act
```

```shell
cd ../; 
act --list; 
act -j api-compliance-test
```

## Linking back to platform engineering

* Standardise how services are built and packaged.
* Increase velocity my having a pipelines that runs locally, or provide an environment to test with before pushing code changes.
* The relationship between services' source code (directory/repo structures) influences the shape of our services pipelines. 
* :question: Where do service pipelines start and end?
  * e.g. VCS change submitted --> publishing container image to registry
* Need a clear understanding of who will consume each artefact.
* Inner-loop vs. outer-loop pipelines ~ local vs. remote pipelines.
