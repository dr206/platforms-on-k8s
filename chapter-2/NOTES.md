# DevOps Book Club - Week 2

<!-- TOC -->
* [DevOps Book Club - Week 2](#devops-book-club---week-2)
  * [Running our cloud-native applications](#running-our-cloud-native-applications)
  * [Installing the conference application with a single command](#installing-the-conference-application-with-a-single-command)
    * [Validate Helm state exists](#validate-helm-state-exists)
    * [Inspect web app](#inspect-web-app)
    * [Clean up](#clean-up)
  * [Inspecting the walking skeleton](#inspecting-the-walking-skeleton)
    * [Deployments](#deployments)
    * [ReplicaSets](#replicasets)
      * [Debugging](#debugging)
    * [Services](#services)
    * [Ingress](#ingress)
    * [Troubleshooting](#troubleshooting)
      * [Dashboard](#dashboard)
        * [Headlamp](#headlamp)
  * [Cloud-native application challenges](#cloud-native-application-challenges)
    * [Downtime is not allowed](#downtime-is-not-allowed)
      * [Two pods are resilient](#two-pods-are-resilient)
      * [One pod has downtime](#one-pod-has-downtime)
    * [Service's resilience built-in](#services-resilience-built-in)
    * [Dealing with the application state is not trivial!](#dealing-with-the-application-state-is-not-trivial)
    * [Dealing with inconsistent data](#dealing-with-inconsistent-data)
    * [Understanding how the application is working (Observability.)](#understanding-how-the-application-is-working-observability)
    * [Security & Identity Management](#security--identity-management)
  * [Linking back to platform engineering](#linking-back-to-platform-engineering)
<!-- TOC -->

## Running our cloud-native applications

Types of envs:
* local envs (e.g., minikube, kind, k3d)
* on-prem envs (e.g., OpenShift, Rancher)
* cloud envs (e.g., EKS, GKE, AKS)

Kubernetes resources:
* Deployments
* Services
* Ingress
* ConfigMaps/Secrets

Packaging:
* template engines vs. package managers, e.g.
  * Helm
  * Kustomize
* Centralized repositories:
  * Artifact Repositories (e.g., Nexus, Artifactory)
  * Container Registries (e.g., DockerHub, ECR, GCR, ACR)
  * Helm Chart Repositories/OCI Registries

```shell

cat <<EOF | kind create cluster --name dev --config=-
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  kubeadmConfigPatches:
  - |
    kind: InitConfiguration
    nodeRegistration:
      kubeletExtraArgs:
        node-labels: "ingress-ready=true"
  extraPortMappings:
  - containerPort: 30376
    hostPort: 80
    protocol: TCP
  - containerPort: 32069
    hostPort: 443
    protocol: TCP
- role: worker
- role: worker
- role: worker
EOF
```

```shell
./kind-load.sh
```

```shell
docker exec -it dev-control-plane crictl images
```

```shell
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
```

Install `NodePort` ingress with ports that map to the `kind` cluster definition above:
```shell
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.service.type=NodePort \
  --set controller.service.nodePorts.http=30376 \
  --set controller.service.nodePorts.https=32069
```

```shell
kubectl get pods -n ingress-nginx
```

## Installing the conference application with a single command

:warning: Works with Helm 3, not with Helm 4.

```shell
helm install conference oci://docker.io/salaboy/conference-app --version v1.0.0 --post-renderer ./rewrite-bitnami-images.sh
```

```shell
helm show all oci://docker.io/salaboy/conference-app --version v1.0.0
```

```shell
helm list
```

```shell
kubectl get pods -owide
```

### Validate Helm state exists

```shell
kubectl get secrets
```

```shell (incomplete resource name is a prefix for many resources)
kubectl describe secrets sh.helm.release.v1.conference.v
```

```shell
date -jf "%s" "TBC" +"%Y-%m-%d %H:%M"
```

### Inspect web app

```shell
open http://localhost/
```

```shell
kubectl logs -f conference-notifications-service-deployment-
```

### Clean up

```shell
kubectl get pvc
```

```shell
kubectl delete pvc  data-conference-kafka-0 data-conference-postgresql-0 redis-data-conference-redis-master-0
```

```shell
kind delete clusters dev
```

##  Inspecting the walking skeleton

### Deployments

```shell
kubectl get deployment
```

```shell
kubectl describe deployment conference-frontend-deployment 
```

```shell (highlights events)
kubectl describe deployment conference-agenda-service-deployment
```

```shell
brew install kubectl-tree
```

```shell
kubectl tree deployment conference-frontend-deployment
```

### ReplicaSets

```shell
kubectl get replicaset
```

```shell
kubectl describe replicasets.apps conference-frontend-deployment-
```

```shell
kubectl scale deployment conference-frontend-deployment --replicas=2
```

```shell
kubectl describe deployment conference-frontend-deployment
```

```shell
kubectl get pods -w
```

#### Debugging

```shell
kubectl set env deployment/conference-frontend-deployment FEATURE_DEBUG_ENABLED=true
```

```shell
kubectl describe deployment conference-frontend-deployment
```

```shell (watch debug tab's pods + node change)
open http://localhost/backoffice/
```

### Services

```shell
kubectl get svc
```

```shell
kubectl describe svc frontend
```

### Ingress

```shell
kubectl get ingres
```

```shell
kubectl describe ingres conference-frontend-ingress
```

### Troubleshooting

```shell
kubectl port-forward services/agenda-service 8080:80
```

```shell
open http://localhost:8080/service/info
```

```shell
curl -s http://localhost:8080/service/info | jq --color-output
```

#### Dashboard

Kubernetes Dashboard is [deprecated](https://kubernetes.io/docs/tasks/access-application-cluster/web-ui-dashboard/).

##### Headlamp

Install:

```shell
brew install --cask --no-quarantine headlamp
```

```shell
open -a headlamp
```

## Cloud-native application challenges

### Downtime is not allowed

####  Two pods are resilient

```shell
kubectl scale deployment conference-frontend-deployment --replicas=2
```

```shell
kubectl get pods
```

```shell
kubectl describe pods conference-frontend-deployment- 
```

```shell
kubectl get pods
```

####  One pod has downtime

```shell
kubectl scale deployment conference-frontend-deployment --replicas=1
```

```shell
kubectl describe pods conference-frontend-deployment-
```

```shell (is down)
open http://localhost/
```

### Service's resilience built-in

```shell
kubectl scale deployment conference-agenda-service-deployment --replicas=0
```

```shell (works)
open http://localhost/
```

```shell (uses cache)
open http://localhost/agenda
```

```shell (debug shows service down)
open http://localhost/backoffice
```

> now your service is responsible for dealing with errors generated by downstream services.

### Dealing with the application state is not trivial!

```shell
kubectl scale deployment conference-agenda-service-deployment --replicas=1
```

```shell
kubectl scale deployment conference-agenda-service-deployment --replicas=2
```

```shell
kubectl get pods
```

:question: What would happen if the Agenda service keeps the state in-memory?

### Dealing with inconsistent data

* A `CronJob` can be used to aggregate data from across multiple services. 
  Then use the aggregated outcome to send notifications in cases where inconsistencies are found.

###  Understanding how the application is working (Observability.)

* OpenTelemetry
  * Logs
  * Traces
  * Metrics

###  Security & Identity Management

* IdM
  * Authorization
  * Authentication
* OAuth2
  * KeyCloak
  * Zitadel

---

:question: What other ways can we break this first version of the application?

## Linking back to platform engineering

* Kubernetes provides building blocks
* Old practices can be bad habits
* Packaging and versioning aids in porting applications to different environments
* Two phases:
  * Build packages &rarr; `Grokking Continuous Delivery` (book)
  * Deploy packages &rarr; **this book**!
