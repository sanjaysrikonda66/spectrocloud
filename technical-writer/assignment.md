# Debug Operations in Kubernetes

The `kubectl` Command-Line Interface (CLI) is the primary tool for managing Kubernetes clusters. It acts as a bridge between your local environment and the Kubernetes API server, allowing you to perform CRUD (Create, Read, Update, Delete) operations on cluster resources.

Beyond basic resource management, we use `kubectl` to investigate the internal state of the cluster. For example, you can view real-time events, inspect hardware resource consumption, and port-forward local traffic into a private pod for isolated testing. 

## Prerequisites

 * You must have access to the Kubernetes containers.
 * A SpectroCloud account with access to see your containers and pods.
 * An active [host cluster](https://docs.spectrocloud.com/glossary-all/#host-cluster). Refer to [Getting Started](https://docs.spectrocloud.com/getting-started/) for tutorials and instructions on how to deploy a cluster.
 * [Kubectl](https://docs.spectrocloud.com/clusters/cluster-management/palette-webctl/)  installed and configured to access your host cluster. Refer to the [Access Cluster with CLI](https://docs.spectrocloud.com/clusters/cluster-management/palette-webctl/) page for guidance on how to access your cluster with the `kubectl` CLI.


## Relationship of Kubernetes Clusters, Namespaces, Pods, and Containers

Cluster(s) --> Namespace(s) --> Pod(s) --> Container(s)

A Kubernetes cluster is the top-level entity containing worker nodes that run applications. Namespaces serve as logical partitions for isolating groups of resources within a single cluster. A cluster may contain more than one namespace. Namespaces contain pods that are the smallest deployable units that live within a namespace. Pods act as wrappers that house one or more containers. Containers package applications, while pods define the shared network, storage, and lifecycle for those containers. For a more in-depth discussion of Kubernetes components, see [Kubernetes Components](https://kubernetes.io/docs/concepts/overview/components/). 


## Debug a Container

The reason to debug a container is if something goes wrong. For example, an application may be having issues. 

We recommend following this sequence when you debug a container:

1. Issue `kubectl get pods` to check the status of your pods.
2. Issue `kubectl logs` to review container output.
3. Issue `kubectl exec` to explore the internal container environment.

## Retrieve Pod Status

Issue the following command to retrieve a list of all available pods and their current status. You must specify the Namespace if the pods are not in the default Namespace:

### Retrieve All Pods In A Namespace

```shell
kubectl get pods --namespace 
```

For example, run this command for a namespace called `production`, to see the list of pods in the `production` namespace:

```shell
kubectl get pods --namespace production
```

```shell
NAME                                     READY   STATUS    RESTARTS   AGE
api-gateway-748c6678-x9z2m               1/1     Running   0          12h
auth-service-59d99789-b5p4r              1/1     Running   2          4d2h
payment-processor-6f8d6f54-m7q3l         1/1     Running   0          22h
background-worker-8456d9bc-s2v4k         1/1     Running   0          12h
database-proxy-5567b4f9-c6n8t            1/1     Running   0          30d
```

### Retrieve A List of All Pods Across Every Namespace

If you have more than one namespace, issue the following command to retrieve a list of all pods across every namespace in your Kubernetes cluster:

```shell
kubectl get pods --all-namespaces
```

The shortcut command to retrieve the same information is: 

```shell
kubectl get pods -A
```

```shell
NAMESPACE     NAME                                      READY   STATUS    RESTARTS   AGE
kube-system   coredns-78fcdf6894-j7v2n                  1/1     Running   0          14d
kube-system   etcd-control-plane                        1/1     Running   0          14d
kube-system   kube-apiserver-control-plane              1/1     Running   0          14d
kube-system   kube-proxy-s2w8l                          1/1     Running   0          14d
production    api-gateway-748c6678-x9z2m                1/1     Running   0          12h
production    auth-service-59d99789-b5p4r               1/1     Running   2          4d2h
staging       test-db-0                                 1/1     Running   0          3h
default       frontend-v1-67f789bb-h5n1l                1/1     Running   0          45m
```

Note there are more parameters that can be passed for more detailed pod information. For a full list of pod commands, see [Get Detailed Pod Information](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#get).

## Review Container Logs

Once you have the name of the resource that you want to review, use that information to retrieve the logs. 

Issue the `logs` command to retrieve the logs of a specific resource. This is a primary tool for reviewing errors or debugging a container. 

### Retrieve Logs For A Specific Pod In A Namespace

To retrieve the standard output (stdout) and standard error (stderr) streams from the primary container inside of a specific pod, issue the following command:

```shell
kubectl logs <pod-name> --namespace <namespace>
```

Note that if a pod only has one container, the container name is optional. 

For this example, `api-gateway-748c6678-x9z2m` is the pod name and `production` is the namespace:

```shell
kubectl logs api-gateway-748c6678-x9z2m --namespace production
```

```shell
2026-04-27 14:10:05 INFO  Starting production server on port 8080...
2026-04-27 14:10:06 INFO  Loading configuration from /etc/config/settings.yaml
2026-04-27 14:10:08 INFO  Successfully connected to Redis at 10.0.12.45:6379
2026-04-27 14:11:15 DEBUG Incoming request: GET /api/v1/health
2026-04-27 14:11:15 INFO  Request processed: 200 OK in 14ms
2026-04-27 14:12:30 ERROR Failed to connect to billing-service: Connection timeout
2026-04-27 14:12:30 WARN  Retrying connection in 5 seconds...
```
### Retrieve Logs Of All Containers Running Inside A Specified Pod

Issue the command to retrieve the logs of all containers running inside of a specific pod:

```shell 
kubectl logs <pod-name> --all-containers=true --namespace <namespace-name>
```

```shell
# Output from the 'main-app' container
[main-app] 2026-04-27 14:05:01 INFO  Starting application on port 8080
[main-app] 2026-04-27 14:05:05 INFO  Connected to database at 10.0.1.5

# Output from the 'istio-proxy' sidecar container
[istio-proxy] [2026-04-27T14:05:06.123Z] "GET /healthz HTTP/1.1" 200 - 0 15 1 - "-"
[istio-proxy] [2026-04-27T14:05:10.456Z] "POST /api/v1/data HTTP/1.1" 201 - 0 124 5 - "-"

# Output from the 'log-collector' container
[log-collector] 2026-04-27 14:05:11 DEBUG Flushing buffer to S3 bucket...
```

For a full list of log commands, see [Get Detailed Log Information](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#logs).

## Execute Commands In A Container

Issue the `exec` command to debug a container from the inside. This command allows you to explore the environment and review configuration files.

```shell 
kubectl exec -it <pod-name> 
```

In this example, `-it` stands for:

 * i (interactive) - interactive keeps `stdin` open even if not attached.
 * t (tty) - allocates a pseudo-terminal.

If you want to use your own shell program you have to add:

```shell 
kubectl exec -it <pod-name> -- /bin/bash
```
where: 

 * `--` - tells Kubernetes that its own flags have ended and everything following it is the command to be executed inside the container.
 * `/bin/bash` - the specific shell program you want to start.


### Target Specific Containers

If a pod contains multiple containers (such as sidecar), you must specify the container name using the `-c` flag. If you do not specify a container, `kubectl` will default to the first container listed in the pod specification. 

For example, if your pod contains two containers:

 * `app-logic` - the main application.
 * `cloud-sql-proxy` - a sidecar for database connectivity.

Issue the following command to run against a specific container:

```shell
kubectl exec -it order-processor-v1 -c cloud-sql-proxy -- /bin/sh
```

```shell
user@local:~$ kubectl exec -it order-processor-v1 -c cloud-sql-proxy -- /bin/sh
/ # 
/ # ls /etc/cloudsql
credentials.json
/ # exit
user@local:~$
```

## Run Commands Without A Shell

You can also issue a single command without entering an interactive session. This is useful for quick checks of the filesystem or environment variables.

```shell
# List files in the configuration directory
kubectl exec <pod-name> -- ls /etc/config

# Check an environment variable
kubectl exec <pod-name> -- printenv DATABASE_URL
```

For a complete list of flags that you can use for the `exec` command, see [Kubernetes `exec` Command](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#exec).

## Caveats About Making Changes Inside A Pod

 * **Temporary Changes** - Any changes you make inside of a pod are temporary and will be lost if the pod restarts or is deleted. 
 * **Permissions** - In production environments `exec` is oftentimes restricted as it can be used to bypass security controls or extract sensitive data from the environment.
 * **Pod State** - You can only execute commands in a pod in a running state. You cannot `exec` commands into pods that are in `Pending`, `Succeeded`, or `Failed` states.




# References

- https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#-strong-getting-started-strong-

- [What is Kubernetes](https://kubernetes.io/docs/concepts/overview/)
