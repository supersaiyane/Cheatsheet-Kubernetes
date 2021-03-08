![licence:free to use](https://img.shields.io/badge/licence-free--to--use-blue)  [![Linkedin Badge](https://img.shields.io/badge/-gurpreetsingh89-blue?style=flat&logo=Linkedin&logoColor=white&link=https://www.linkedin.com/in/gurpreetsingh89/)](https://www.linkedin.com/in/gurpreetsingh89/)  [![dev.to Badge](https://img.shields.io/badge/-@gurpreetsingh-000000?style=flat&labelColor=000000&logo=dev.to&link=https://dev.to/gurpreetsingh)](https://dev.to/gurpreetsingh)

# Kubernetes Cheatsheet

## Table of Contents

- [Overview](#overview)
  - …
- [Minikube](#minikube)
  - …
- [Kubeadm](#kubeadm)
  - …
- [Kubectl](#kubectl)
  - …
- [PODs](#pod)
  - …
- [Replication Controller](#replication-controller)
  - …
- [Services](#services-1)
  - …
- [Deployments](#deployments)
  - …
- [Secrets](#secrets)
  - …
- [Useful](#useful)
  - [image from private registry](#image-from-private-registry)
- [Troubleshooting](#troubleshooting)
  - [Use local docker image](#use-local-docker-image)
  - [Docker Logging](#docker-logging)

## Overview

- Kubernetes is an orchestrator for microservice apps.

### Pods

- Pod run Containers (that share the pod environment)
- Pods usually have only one container, they can have sidecars
- Sidecars are for example if you have a main container that writes logs + a log scraper that collects the logs and then expose them somewhere that log scraper would be called sidecar container.
- A pod is mortal, if it dies, a new one is created. They are never brought back to life.
- Every time a new pod is spin up it gets a new ip, that is why pod ips are not reliable
- That is why services are useful.

### Replication Controller / Replica Sets

- Are constructs designed to make sure the required number of pods is always running
- Kind of replaced by deployments
- Replica Sets is how they are called inside deployments (with subtle no needed to know diffs)

### Services

- Is a simple object defined by a manifest
- Provides a stable IP and DNS for pods sitting behind it
- Loadbalances requests
- Pods belong to services using labels (for example Prod+BE+1.3, then to update just change label to 1.4, so a rollback and forward is just a matter of changing labels)

### Deployment

- Is defined in yaml as desired state
- Add features to replication controllers/sets and takes care of it
- Simple rolling updates and rollbacks (blue-green / canary)

## Minikube

- Play around with kubernetes locally (single host kubernetes cluster)

### Install

#### OSX

https://minikube.sigs.k8s.io/docs/getting-started/macos/

Enable Kubernetes on your Docker for Mac Settings

```
brew install kubectl
brew cask install minikube

# Optional

brew install docker-machine-driver-xhyve
# follow instruction commands
```

#### Windows

https://minikube.sigs.k8s.io/docs/getting-started/windows/

#### Linux

https://minikube.sigs.k8s.io/docs/getting-started/linux/

### Basics

```bash
minikube start --vm-driver=<driver> --kubernetes-version=<version>
# example: minikube start --vm-driver=xhyve --kubernetes-version="v1.6.0"
```

- `--vm-driver`: which driver to use (default is a virtual machine)
- `--kubernetes-version`: default is latest

```bash
minikube stop
minikube delete
minikube status
minikube dashboard
# Opens a dashboard GUI for minikube
```

## Kubeadm

### Startup – Multinodes

https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/

```
apt-get install docker.io kubeadm kubectl kubelet kubernetes-cni -y
```

```bash
kubeadm init
# follow instructions
```

You’ll need to have a pod network running. You can use the weave networking setup:

```bash
kubectl apply --filename https://git.io/weave-kube-1.6
# You can also run it with other versions using https://git.io/weave-kube without the 1.6
```

Now join other nodes (servers) with the join token you received from `kubeadm init`.

### Startup – Singlenode

See: https://medium.com/@vivek_syngh/setup-a-single-node-kubernetes-cluster-on-ubuntu-16-04-6412373d837a

```bash
sudo apt-get update
sudo apt-get upgrade
sudo curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | apt-key add
touch /etc/apt/sources.list.d/kubernetes.list
echo "deb http://apt.kubernetes.io/ kubernetes-xenial main" >> /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl kubernetes-cni
sysctl net.bridge.bridge-nf-call-iptables=1
kubeadm init --pod-network-cidr=10.244.0.0/16
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/bc79dd1505b0c8681ece4de4c0d86c5cd2643275/Documentation/kube-flannel.yml
kubectl taint nodes --all node-role.kubernetes.io/master-
```

## Kubectl

### Basics

```bash
kubectl config current-context
# Displays the current working context. For example `minikube`
kubectl cluster-info
# Information about your cluster. For example the IP
```

### Nodes

```
kubectl get nodes
```

Displays current running nodes

### Pods

```bash
kubectl get pods
# normal
kubectl get pods --all-namespaces
# Display the pods from all namespaces (= also system ones)
kubectl get pods/<pod-name>
# Example: kubectl get pods/hello-pod
# retrieves a single pod
kubectl describe pods
# Displays more information on the pods
```

### Replication Controllers

```bash
kubectl get rc
# normal
kubectl get rc/<rc-name>
# Example: kubectl get rc/hello-rc
# retrieves a single rc
kubectl describe rc
# Displays more information on the rcs
```

### logs

https://kubernetes.io/docs/reference/kubectl/docker-cli-to-kubectl/#docker-logs

## POD

- Smallest unit in Kubernetes
- Contains 1 or more containers
- 1 IP to 1 POD relationship
- POD is either running or down, never half running
- Declared in a manifest file

### Example

*Note: usually you never work directly on a pod*

```yml
apiVersion: v1
kind: Pod
metadata:
  name: hello-pod
  labels:
    zone: prod
    version: v1
spec:
  containers:
    - name: hello-ctr
      image: nigelpoulton/pluralsight-docker-ci:latest
      ports:
        containerPort: 8080
```

- `apiVersion`: version to use
- `kind`: type of object we are declaring
- `metadata`: extra information
- `labels`: is a key-value pair object
- `containers`: list of containers to run in a pod

```bash
kubectl create -f <path>
# example: kubectl create -f pod.yml
```

remove it:
```bash
kubectl delete pods <name>
# example: kubectl delete pods hello-pod
```

## Replication Controller

### Example

```yml
apiVersion: v1
kind: ReplicationController
metadata:
  name: hello-rc
spec:
  replicas: 10
  selector:
    app: hello-world
  template:
    metadata:
      labels:
        app: hello-world
    spec:
      containers:
        - name: hello-pod
          image: nigelpoulton/pluralsight-docker-ci:latest
          ports:
            containerPort: 8080
```


```bash
kubectl create -f <path>
# example: kubectl create -f rc.yml
```

### Update

- Just edit the yml file

```bash
kubectl apply -f <path>
```

## Services

- Sits in front of pods
- Exposes an IP that can be called to reach the PODs
- It will loadbalance between PODs automatically
- Services matches PODs via labels

### Example

#### Iterative

```bash
kubectl expose rc <rc-name> --name=<service-name> --target-port=<port> --type=<type>
# Example: kubectl expose rc hello-rc --name=hello-svc --target-port=8080 --type=NodePort
kubectl describe svc <service-name>
kubectl delete svc <service-name>
```

#### Declatative

```yml
apiVersion: v1
kind: Service
metadata:
  name: hello-svc
  labels:
    app: hello-world
spec:
  type: NodePort
  ports:
    - port: 8080
      nodePort: 30001
      protocol: TCP
  selector:
    app: hello-world
```

- `type`: the service type.
  - `CluserIP`: Stable internal cluster IP. Default. Makes the service only available to other nodes in the same cluster.
  - `NodePort`: Exposes the app outside of the cluster by adding a cluster wide port on top of the IP
  - `LoadBalancer`: Integrates the NodePort with an existing LoadBalancer
- `port`: is the port exposed within the container. It gets mapped through the NodePort (convention something above 30000) on the whole cluster (here 30001).
- `selector`: has to match the label of the PODs/RC

```bash
kubectl create -f <path>
# example: kubectl create -f svc.yml
# kubectl delete svc hello-svc
```

### Updates / Deployments

- Usually you give the pods a version label. E.g. (app=foo;zone=prod;ver=1.0.0)
- The service matches everything but the version label. E.g. (app=foo;zone=prod)
- New version comes: spin up new pods with the new version tag. E.g. (app=foo;zone=prod;ver=2.0.0)
- Now it’s loadbalanced across the new and the old
- If you’re happy, let the service match only the new version by adding the version label to the service. E.g. (app=foo;zone=prod;ver=2.0.0)
- The old pods are still there. To rollback, you just revert the label of the service to the old version  E.g. (app=foo;zone=prod;ver=1.0.0) and it will only target the old app. No downtime.
This is basically called blue/green deployment

## Deployments

- All about rolling updates and simple rollbacks
- Deployments wrap around replication controllers (in the world of deployment called replica set)
- Deployments manage Replica Sets, Replica Sets manage Pods

### Example

```yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-deploy
spec:
  selector:
    matchLabels:
      app: hello-world
  replicas: 10
  minReadySeconds: 10
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
  template:
    metadata:
      labels:
        app: hello-world
    spec:
      containers:
        - name: hello-pod
          imagePullPolicy: IfNotPresent
          image: nigelpoulton/pluralsight-docker-ci:latest
          ports:
            - containerPort: 8080
```

- `minReadySeconds`: let the pod run for 10 second before marking it as ready
- `strategy`: select what kind of update strategy to use (here we use rolling updates)
  - `maxUnavailable` & `maxSurge`: by setting it to 1 we tell the deployment to do them 1 by 1

```bash
kubectl create -f <path>
# example: kubectl create -f deploy.yml
kubectl describe deploy hello-deploy
```

### Updates

- Edit the yml file.
- `kubectl apply -f <file> --record` will apply the changes
- `kubectl rollout status deployment <name-of-deployment>` to watch it
- `kubectl get deploy <name-of-deployment>` check if all are available
- `kubectl rollout history deployment <name-of-deployment>` to see the versions and why it happened
- `kubectl get rs` you’ll see that you have 2 replica sets, 1 with 0 pods and another with the new pods

### Rollbacks

- `kubectl rollout undo deployment <name-of-deployment> --to-revision=<number>` (you can see revisions via `kubectl rollout history deployment <name-of-deployment>`)
- does the same as for the update but in reverse to match the desired state of the specified revision number.

## Secrets

https://kubernetes.io/docs/concepts/configuration/secret/

### Example

```yml
apiVersion: v1
kind: Secret
metadata:
  name: mysecret
type: Opaque
data:
  username: YWRtaW4=
  password: MWYyZDFlMmU2N2Rm
```

- `data`: is a key-value store with the value set as a BASE64 encoded sring.

```
kubectl apply -f ./secret.yaml
```

*Note: secrets can also be created from files:*

```bash
kubectl create secret generic <name> --from-file=<path>
```

- `name`: the name of the secret

And then retrieved with:

```bash
kubectl get secret <name> -o yaml
```

## Useful

### Image from private registry

https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/

```
docker login <registry uri>
```
```
kubectl create secret generic regcred \
    --from-file=.dockerconfigjson=<path/to/.docker/config.json> \
    --type=kubernetes.io/dockerconfigjson
```

- `<path/to/.docker/config.json>`: is usually in the users folder `Users/<username>/.docker/config.json`

OR

```bash
kubectl create secret docker-registry regcred --docker-server=<your-registry-server> --docker-username=<your-name> --docker-password=<your-pword> --docker-email=<your-email>
# Example: kubectl create secret docker-registry regcred --docker-server=https://example.com/ --docker-username=docker --docker-password=password --docker-email=example@mail.com
```

You can get the secret:
```
kubectl get secret regcred --output=yaml
```

Or create it on the spot (don’t forget to add `https://` and `/v2/` to the docker server variable):
```
kubectl create secret docker-registry regcred --docker-server=<your-registry-server> --docker-username=<your-name> --docker-password=<your-pword> --docker-email=<your-email>
```

Now you can add it to your pods:

```yml
…
spec:
  containers:
  …
  imagePullSecrets:
  - name: regcred
```


## Troubleshooting

### Use local docker image

Minikube runs in a VM hence it will not see the images you've built locally on a host machine, but... as stated in https://github.com/kubernetes/minikube/blob/master/docs/reusing_the_docker_daemon.md you can use `eval $(minikube docker-env)` to actually utilise docker daemon running on minikube, and henceforth build your image on the minikubes docker and thus expect it to be available to the minikubes k8s engine without pulling from external registry

### Docker Logging

https://kubernetes.io/docs/reference/kubectl/docker-cli-to-kubectl/#docker-logs



- [Kubernetes cheatsheet](#kubernetes-cheatsheet)
  - [Getting Started](#getting-started)
  - [Sample yaml](#sample-yaml)
  - [Workflow](#workflow)
  - [Physical components](#physical-components)
    - [Master](#master)
    - [Node](#node)
  - [Everything is an object - persistent entities](#everything-is-an-object---persistent-entities)
    - [Namespaces](#namespaces)
    - [Labels](#labels)
      - [ClusterIP](#clusterip)
    - [Controller manager](#controller-manager)
    - [Kube-scheduler](#kube-scheduler)
    - [Pod](#pod)
      - [Status](#status)
      - [Probe](#probe)
      - [Pod priorities](#pod-priorities)
      - [Multi-Container Pods](#multi-container-pods)
      - [Init containers](#init-containers)
      - [Lifecycle hooks](#lifecycle-hooks)
      - [Quality of Service (QoS)](#quality-of-service-qos)
      - [PodPreset](#podpreset)
    - [ReplicaSet](#replicaset)
    - [Deployments](#deployments)
    - [ReplicationController](#replicationcontroller)
    - [DaemonSet](#daemonset)
    - [StatefulSet](#statefulset)
    - [Job (batch/v1)](#job-batchv1)
    - [Cronjob](#cronjob)
    - [Horizontal pod autoscaler](#horizontal-pod-autoscaler)
    - [Services](#services)
    - [Volumes](#volumes)
      - [Persistent volumes](#persistent-volumes)
    - [Role-Based Access Control (RBAC)](#role-based-access-control-rbac)
    - [Custom Resource Definitions](#custom-resource-definitions)
  - [Notes](#notes)
    - [Basic commands](#basic-commands)
    - [jsonpath](#jsonpath)
    - [Resource limit](#resource-limit)
      - [CPU](#cpu)
      - [Memory](#memory)
    - [Chapter 13. Integrating storage solutions and Kubernetes](#chapter-13-integrating-storage-solutions-and-kubernetes)
      - [Downward API](#downward-api)
  - [Labs](#labs)
    - [Guaranteed Scheduling For Critical Add-On Pods](#guaranteed-scheduling-for-critical-add-on-pods)
    - [Set command or arguments via env](#set-command-or-arguments-via-env)

## Getting Started

- Fault tolerance
- Rollback
- Auto-healing
- Auto-scaling
- Load-balancing
- Isolation (sandbox)

## Sample yaml

```yaml
apiVersion: <>
kind: <>
metadata:
  name: <>
  labels:
    ...
  annotations:
    ...
spec:
  containers:
    ...
  initContainers:
    ...
  priorityClassName: <>
```

## Workflow

Credit: https://www.reddit.com/user/__brennerm/

![](https://i.redd.it/cqud3rjkss361.png)

- (kube-scheduler, controller-manager, etcd) --443--> API Server

- API Server --10055--> kubelet
  - non-verified certificate
  - MITM
  - Solution:
    - set kubelet-certificate-authority
    - ssh tunneling

- API server --> (nodes, pods, services)
  - Plain HTTP (unsafe)

## Physical components

### Master

- API Server (443)
- kube-scheduler
- controller-manager
  - cloud-controller-manager
  - kube-controller-manager
- etcd

Other components talk to API server, no direct communication

### Node

- Kubelet
- Container Engine
  - CRI
    - The protocol which used to connect between Kubelet & container engine

- Kube-proxy

## Everything is an object - persistent entities

- maintained in etcd, identified using
  - names: client-given
  - UIDs: system-generated
- Both need to be unique

- three management methods
  - Imperative commands (kubectl)
  - Imperative object configuration (kubectl + yaml)
    - repeatable
    - observable
    - auditable
  - Declarative object configuration (yaml + config files)
    - Live object configuration
    - Current object configuration file
    - Last-applied object configuration file

```text
      Node Capacity
---------------------------
| kube-reserved             |
|---------------------------|
| system-reserved           |
| ------------------------- |
| eviction-threshold        |
| ------------------------- |
|                           |
| allocatable               |
| (available for pods)      |
|                           |
|                           |
---------------------------
```

### Namespaces

- Three pre-defined
  - default
  - kube-system
  - kube-public: auto-readable by all users

- Objects without namespaces
  - Nodes
  - PersistentVolumes
  - Namespaces

### Labels

- key / value
- loose coupling via selectors
- need not be unique

#### ClusterIP

- Independent of lifespan of any backend pod
- Service object has a static port assigned to it

### Controller manager

- ReplicaSet, deployment, daemonset, statefulSet
- Actual state <-> desired state
- reconciliation loop

### Kube-scheduler

- nodeSelector
- Affinity & Anti-Affinity
  - Node
    - Steer pod to node
  - Pod
    - Steer pod towards or away from pods
- Taints & tolerations (anti-affinity between node and pod!)
  - Base on predefined configuration (env=dev:NoSchedule)
    ```yaml
    ...
    tolerations:
    - key: "dev"
      operator: "equal"
      value: "env"
      effect: NoSchedule
    ...
    ```
  - Base on node condition (alpha in v1.8)
    - taints added by node controller

### Pod

```bash
kubectl run name --image=<image>
```

What's available inside the container?

- File system
  - Image
  - Associated Volumes
    - ordinary
    - persistent
  - Container
    - Hostname
  - Pod
    - Pod name
    - User-defined envs
  - Services
    - List of all services

Access with:

- Symlink (important):

  - /etc/podinfo/labels
  - /etc/podinfo/annotations

- Or:

```yaml
volumes:
  - name: podinfo
    downwardAPI:
      items:
        - path: "labels"
          fieldRef:
            fieldPath: metadata.labels
        - path: "annotations"
          fieldRef:
            fieldPath: metadata.annotations
```

#### Status

- Pending
- Running
- Succeeded
- Failed
- Unknown

#### Probe

- Liveness
  - Failed? Restart policy applied
- Readiness
  - Failed? Removed from service

#### Pod priorities

- available since 1.8
- PriorityClass object
- Affect scheduling order
  - High priority pods could jump the queue
- Preemption
  - Low priority pods could be pre-empted to make way for higher one (if no node is available for high priority)
  - These preempted pods would have a graceful termination period

#### Multi-Container Pods

- Share access to memory space
- Connect to each other using localhost
- Share access to the same volume
- entire pod is host on the same node
- all in or nothing
- no auto healing or scaling

#### Init containers

- run before app containers
- always run to completion
- run serially

#### Lifecycle hooks

- PostStart
- PreStop (blocking)

Handlers:

- Exec
- HTTP

```yaml
...
spec:
  containers:
    lifecycle:
      postStart:
        exec:
          command: <>
      preStop:
        http:
...
```

Could invoke multiple times

#### Quality of Service (QoS)

When Kubernetes creates a Pod it assigns one of these QoS classes to the Pod:

- Guaranteed (all containers have limits == requests)

>If a Container specifies its own memory limit, but does not specify a memory request, Kubernetes automatically assigns a memory request that matches the limit. Similarly, if a Container specifies its own cpu limit, but does not specify a cpu request, Kubernetes automatically assigns a cpu request that matches the limit.

- Burstable (at least 1 has limits or requests)
- BestEffort (no limits or requests)

#### PodPreset

You can use a podpreset object to inject information like secrets, volume mounts, and environment variables etc into pods at creation time. This task shows some examples on using the PodPreset resource

```yaml
apiVersion: settings.k8s.io/v1alpha1
kind: PodPreset
metadata:
  name: allow-database
spec:
  selector:
    matchLabels:
      role: frontend
  env:
    - name: DB_PORT
      value: "6379"
  volumeMounts:
    - mountPath: /cache
      name: cache-volume
  volumes:
    - name: cache-volume
      emptyDir: {}
```

### ReplicaSet

Features:

- Scaling and healing
- Pod template
- number of replicas

Components:

- Pod template
- Pod selector (could use matchExpressions)
- Label of replicaSet
- Number of replica

- Could delete replicaSet without its pods using `--cascade =false`
- Isolating pods from replicaSet by changing its labels

### Deployments

- versioning and rollback
- Contains spec of replicaSet within it
- advanced deployment
- blue-green
- canary

- Update containers --> new replicaSet & new pods created --> old RS still exists --> reduced to zero
- Every change is tracked

- Append `--record` in kubectl to keep history
- Update strategy
  - Recreate
    - Old pods would be killed before new pods come up
  - RollingUpdate
    - progressDeadlineSeconds
    - minReadySeconds
    - rollbackTo
    - revisionHistoryLimit
    - paused
      - spec.Paused

- `kubectl rollout undo deployment/<> --to-revision=<>`
- `kubectl rollout statua deployment/<>`
- `kubectl set image deployment/<> <>=<>:<>`
- `kubectl rollout resume/pause <>`

### ReplicationController

- RC = ( RS + deployment ) before
- Obsolete

### DaemonSet

- Ensure all nodes run a copy of pod
- Cluster storage, log collection, node monitor ...

### StatefulSet

- Maintains a sticky identity
- Not interchangeable
- Identifier maintains across any rescheduling

Limitation

- volumes must be pre-provisioned
- Deleting / Scaling will not delete associated volumes

Flow

- Deployed 0 --> (n-1)
- Deleted (n-1) --> 0 (successor must be completely shutdown before proceed)
- Must be all ready and running before scaling happens

### Job (batch/v1)

- Non-parallel jobs
- Parallel jobs
  - Fixed completion count
    - job completes when number of completions reaches target
  - With work queue
    - requires coordination
- Use spec.activeDeadlineSeconds to prevent infinite loop

### Cronjob

- Job should be idempotent

### Horizontal pod autoscaler

- Targets: replicaControllers, deployments, replicaSets
- CPU or custom metrics
- Won't work with non-scaling objects: daemonSets
- Prevent thrashing (upscale/downscale-delay)

### Services

Credit: https://www.reddit.com/user/__brennerm/

![](https://i.redd.it/brjcbq9xk7261.png)

- Logical set of backend pods + frontend
- Frontend: static IP + port + dns name
- Backend: set of backend pods (via selector)

- Static IP and networking.
- Kube-proxy route traffic to VIP.
- Automatically create endpoint based on selector.

- CluterIP
- NodePort
  - external --> NodeIP + NodePort --> kube-proxy --> ClusterIP
- LoadBalancer
  - Need to have cloud-controller-manager
    - Node controller
    - Route controller
    - Service controller
    - Volume controller
  - external --> LB --> NodeIP + NodePort --> kube-proxy --> ClusterIP
- ExternalName
  - Can only resolve with kube-dns
  - No selector

`Service discovery`

- SRV record for named port
  - port-name.port-protocol.service-name.namespace.svc.cluster.local
- Pod domain
  - pod-ip-address.namespace.pod.cluster.local
  - hostname is `metadata.name`

`spec.dnsPolicy`

- default
  - inherit node's name resolution
- ClusterFirst
  - Any DNS query that does not match the configured cluster domain suffix, such as “www.kubernetes.io”, is forwarded to the upstream nameserver inherited from the node
- ClusterFirstWithHostNet
  - if host network = true
- None (since k8s 1.9)
  - Allow custom dns server usage

Headless service

- with selector? --> associate with pods in cluster
- without selector? --> forward to externalName

Could specify externalIP to service

### Volumes

Credit: https://www.reddit.com/user/__brennerm/

![](https://i.redd.it/iaflueca8m261.png)

Lifetime longer than any containers inside a pod.

4 types:

- configMap

- emptyDir
  - share space / state across containers in same pod
  - containers can mount at different times
  - pod crash --> data lost
  - container crash --> ok
- gitRepo

- secret
  - store on RAM

- hostPath

#### Persistent volumes

### Role-Based Access Control (RBAC)

Credit: https://www.reddit.com/user/__brennerm/

![](https://i.redd.it/868lf3pp70361.png)

- Role
  - Apply on namespace resources
- ClusterRole
  - cluster-scoped resources (nodes,...)
  - non-resources endpoint (/healthz)
  - namespace resources across all namespaces

### Custom Resource Definitions

CustomResourceDefinitions themselves are non-namespaced and are available to all namespaces.

```yaml
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  # name must match the spec fields below, and be in the form: <plural>.<group>
  name: crontabs.stable.example.com
spec:
  # group name to use for REST API: /apis/<group>/<version>
  group: stable.example.com
  # version name to use for REST API: /apis/<group>/<version>
  version: v1
  # either Namespaced or Cluster
  scope: Namespaced
  names:
    # plural name to be used in the URL: /apis/<group>/<version>/<plural>
    plural: crontabs
    # singular name to be used as an alias on the CLI and for display
    singular: crontab
    # kind is normally the CamelCased singular type. Your resource manifests use this.
    kind: CronTab
    # shortNames allow shorter string to match your resource on the CLI
    shortNames:
    - ct
    # categories is a list of grouped resources the custom resource belongs to.
    categories:
    - all
  validation:
   # openAPIV3Schema is the schema for validating custom objects.
    openAPIV3Schema:
      properties:
        spec:
          properties:
            cronSpec:
              type: string
              pattern: '^(\d+|\*)(/\d+)?(\s+(\d+|\*)(/\d+)?){4}$'
            replicas:
              type: integer
              minimum: 1
              maximum: 10
  # subresources describes the subresources for custom resources.
  subresources:
    # status enables the status subresource.
    status: {}
    # scale enables the scale subresource.
    scale:
      # specReplicasPath defines the JSONPath inside of a custom resource that corresponds to Scale.Spec.Replicas.
      specReplicasPath: .spec.replicas
      # statusReplicasPath defines the JSONPath inside of a custom resource that corresponds to Scale.Status.Replicas.
      statusReplicasPath: .status.replicas
      # labelSelectorPath defines the JSONPath inside of a custom resource that corresponds to Scale.Status.Selector.
      labelSelectorPath: .status.labelSelector
```

## Notes

### Basic commands

```bash
# show current context
kubectl config current-context

# get specific resource
kubectl get (pod|svc|deployment|ingress) <resource-name>

# Get pod logs
kubectl logs -f <pod-name>

# Get nodes list
kubectl get no -o custom-columns=NAME:.metadata.name,AWS-INSTANCE:.spec.externalID,AGE:.metadata.creationTimestamp

# Run specific command | Drop to shell
kubectl exec -it <pod-name> <command>

# Describe specific resource
kubectl describe (pod|svc|deployment|ingress) <resource-name>

# Set context
kubectl config set-context $(kubectl config current-context) --namespace=<namespace-name>

# Run a test pod
kubectl run -it --rm --generator=run-pod/v1 --image=alpine:3.6 tuan-shell -- sh
```

- from @so0k [link](https://gist.github.com/so0k/42313dbb3b547a0f51a547bb968696ba#gistcomment-2040702)

- access dashboard

```bash
# bash
kubectl -n kube-system port-forward $(kubectl get pods -n kube-system -o wide | grep dashboard | awk '{print $1}') 9090

# fish
kubectl -n kube-system port-forward (kubectl get pods -n kube-system -o wide | grep dashboard | awk '{print $1}') 9090
```

### jsonpath

From [link](https://github.com/kubernetes/website/blob/master/content/en/docs/reference/kubectl/jsonpath.md)

```json
{
  "kind": "List",
  "items":[
    {
      "kind":"None",
      "metadata":{"name":"127.0.0.1"},
      "status":{
        "capacity":{"cpu":"4"},
        "addresses":[{"type": "LegacyHostIP", "address":"127.0.0.1"}]
      }
    },
    {
      "kind":"None",
      "metadata":{"name":"127.0.0.2"},
      "status":{
        "capacity":{"cpu":"8"},
        "addresses":[
          {"type": "LegacyHostIP", "address":"127.0.0.2"},
          {"type": "another", "address":"127.0.0.3"}
        ]
      }
    }
  ],
  "users":[
    {
      "name": "myself",
      "user": {}
    },
    {
      "name": "e2e",
      "user": {"username": "admin", "password": "secret"}
    }
  ]
}
```

| Function          | Description               | Example                                                       | Result                                          |
|-------------------|---------------------------|---------------------------------------------------------------|-------------------------------------------------|
| text              | the plain text            | kind is {.kind}                                               | kind is List                                    |
| @                 | the current object        | {@}                                                           | the same as input                               |
| . or []           | child operator            | {.kind} or {['kind']}                                         | List                                            |
| ..                | recursive descent         | {..name}                                                      | 127.0.0.1 127.0.0.2 myself e2e                  |
| \*                | wildcard. Get all objects | {.items[*].metadata.name}                                     | [127.0.0.1 127.0.0.2]                           |
| [start:end :step] | subscript operator        | {.users[0].name}                                              | myself                                          |
| [,]               | union operator            | {.items[*]['metadata.name', 'status.capacity']}               | 127.0.0.1 127.0.0.2 map[cpu:4] map[cpu:8]       |
| ?()               | filter                    | {.users[?(@.name=="e2e")].user.password}                      | secret                                          |
| range, end        | iterate list              | {range .items[*]}[{.metadata.name}, {.status.capacity}] {end} | [127.0.0.1, map[cpu:4]] [127.0.0.2, map[cpu:8]] |
| ''                | quote interpreted string  | {range .items[*]}{.metadata.name}{'\t'}{end}                  | 127.0.0.1    127.0.0.2                          |

Below are some examples using jsonpath:

```shell
$ kubectl get pods -o json
$ kubectl get pods -o=jsonpath='{@}'
$ kubectl get pods -o=jsonpath='{.items[0]}'
$ kubectl get pods -o=jsonpath='{.items[0].metadata.name}'
$ kubectl get pods -o=jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.startTime}{"\n"}{end}'
```

### Resource limit

#### CPU

The CPU resource is measured in cpu units. One cpu, in Kubernetes, is equivalent to:

- 1 AWS vCPU
- 1 GCP Core
- 1 Azure vCore
- 1 Hyperthread on a bare-metal Intel processor with Hyperthreading

#### Memory

The memory resource is measured in bytes. You can express memory as a plain integer or a fixed-point integer with one of these suffixes: E, P, T, G, M, K, Ei, Pi, Ti, Gi, Mi, Ki. For example, the following represent approximately the same value:

128974848, 129e6, 129M , 123Mi

### Chapter 13. Integrating storage solutions and Kubernetes

- External service without selector (access with `external-database.svc.default.cluster` endpoint)

```yaml
kind: Service
apiVersion: v1
metadata:
  name: external-database
spec:
  type: ExternalName
  externalName: "database.company.com
```

- external service with IP only

```yaml
kind: Service
apiVersion: v1
metadata:
  name: external-ip-database
---
kind: Endpoints
apiVersion: v1
metadata:
  name: external-ip-database
subsets:
  - addresses:
    - ip: 192.168.0.1
    ports:
    - port: 3306
```

#### Downward API

The following information is available to containers through environment variables and downwardAPI volumes:

Information available via fieldRef:

- spec.nodeName - the node’s name
- status.hostIP - the node’s IP
- metadata.name - the pod’s name
- metadata.namespace - the pod’s namespace
- status.podIP - the pod’s IP address
- spec.serviceAccountName - the pod’s service account name
- metadata.uid - the pod’s UID
- metadata.labels['<KEY>'] - the value of the pod’s label <KEY> (for example, metadata.labels['mylabel']); available in Kubernetes 1.9+
- metadata.annotations['<KEY>'] - the value of the pod’s annotation <KEY> (for example, metadata.annotations['myannotation']); available in Kubernetes 1.9+
- Information available via resourceFieldRef:
- A Container’s CPU limit
- A Container’s CPU request
- A Container’s memory limit
- A Container’s memory request

In addition, the following information is available through downwardAPI volume fieldRef:

- metadata.labels - all of the pod’s labels, formatted as label-key="escaped-label-value" with one label per line
- metadata.annotations - all of the pod’s annotations, formatted as annotation-key="escaped-annotation-value" with one annotation per line

## Labs

### Guaranteed Scheduling For Critical Add-On Pods

See [link](https://kubernetes.io/docs/tasks/administer-cluster/guaranteed-scheduling-critical-addon-pods/)

- Marking pod as critical when using Rescheduler. To be considered critical, the pod has to:
  - Run in the `kube-system` namespace (configurable via flag)
  - Have the `scheduler.alpha.kubernetes.io/critical-pod` annotation set to empty string
  - Have the PodSpec’s tolerations field set to `[{"key":"CriticalAddonsOnly", "operator":"Exists"}]`.

> The first one marks a pod a critical. The second one is required by Rescheduler algorithm.

- Marking pod as critical when priorites are enabled. To be considered critical, the pod has to:
  - Run in the `kube-system` namespace (configurable via flag)
  - Have the priorityClass set as `system-cluster-critical` or `system-node-critical`, the latter being the highest for entire cluster
  - `scheduler.alpha.kubernetes.io/critical-pod` annotation set to empty string(This will be deprecated too).

### Set command or arguments via env

```yaml
env:
- name: MESSAGE
  value: "hello world"
command: ["/bin/echo"]
args: ["$(MESSAGE)"]
```
