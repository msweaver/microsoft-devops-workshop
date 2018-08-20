# Kubernetes API

K8s Documentation: https://kubernetes.io/docs/concepts/overview/kubernetes-api/

The Kubernetes API is stateless by itself, managing data stored remotely in an etcd cluster. When new work is scheduled it is done through the API, even when using the command line tools. Authorization and validation of requests are performed at this level, as well.

In a Highly Available configuration, this is distributed on at least 3 nodes behind a loadbalancer. If building your own small to medium cluster, you might choose to run the Kubernetes API, on each of the etcd nodes. Depending on how many nodes, and cluster services are running, it may make sense to separate these pools, as losing etcd due to API load is something you want to avoid.

## Demo

**Note:** This is part of the Kubernetes "Control Plane" that is abstracted from the end-user for most managed platforms, such as AKS. This quick demonstration serves to help you understand the components, and the primary DevOps tasks associated.

### Hyperkube to Run it Locally

If etcd is no longer running, you should start it up first, so that our api will have a valid etcd backend.

`export HostIP=$(ipconfig getifaddr en0)`

```
docker run -p 6443:6443 -p 8080:8080 -d --name k8s-api gcr.io/google_containers/hyperkube:v1.11.1 /hyperkube apiserver --service-cluster-ip-range=172.17.17.1/24 --insecure-bind-address=0.0.0.0 --advertise-address=${HostIP} --token-auth-file=/dev/null --address=127.0.0.1 --etcd-servers=http://${HostIP}:2379  --v 2
```

*Open your local browser or `curl -k` to `https://${HostIP}:6443/`

If you are in the browser, you can simply navigate to `/swagger-ui` and you will be presented with the docs in swagger format.

In the event you are using curl, you can see similar text from the output returned here:
`curl -k https://${HostIP}:6443/swaggerapi/api/v1/`

The `ui` endpoint, is a special designation, that gets used later when the cluster has certain cluster services running. It makes use of a primitive called an `Endpoint`, which we will discuss later. ie: `curl -Lk https://${HostIP}:6443/ui`

### kubectl

We can use the `kubectl` tool to interact with the Kubernetes API server. Normally, you would run `kubectl` locally, and configure the client to connect to the cluster, but for our demonstration purposes, we can exec into the container, and use its local unauthenticated access.

`docker exec -it k8s-api bash`

--------- Container Context

`kubectl get namespaces`

*Output:*
```
NAME          STATUS    AGE
default       Active    25m
kube-public   Active    25m
kube-system   Active    25m
```

When scripting kubectl interactions, there are some useful global flags such as `-o`:

`kubectl get namespaces -o yaml`
`kubectl get namespaces -o json`
`kubectl get namespaces -o jsonpath='{.items[*].metadata.name}'`

`kubectl create namespace test`

`exit` to leave the container context.

-------- End of Container Context

To observe that state is stored in etc, you can stop and replace your api container.

```
docker stop k8s-api
docker rm k8s-api
docker run -p 6443:6443 -p 8080:8080 -d --name k8s-api gcr.io/google_containers/hyperkube:v1.11.1 /hyperkube apiserver --service-cluster-ip-range=172.17.17.1/24 --insecure-bind-address=0.0.0.0 --advertise-address=${HostIP} --token-auth-file=/dev/null --address=127.0.0.1 --etcd-servers=http://${HostIP}:2379  --v 2
docker exec -it k8s-api bash
```

`kubectl get namespaces`

You should still see the `test` namespace, since the data was in etcd, the whole time. There are things relative to authentication, such as certs, which are stored on the Kubernetes API server, but should be backed up to an HA storage service. In this way, each new instance of the Kubernetes API, can download config from a central location.