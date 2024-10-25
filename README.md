# How to Deploy eBPF Demo on Kubernetes

The code snippets and configurations used in this guide are available in a Git repository. You can download the repository to your Linux machine by executing:

```bash
git clone https://github.com/ndecandia/ebpf-demo.git
```

Navigate to the `ebpf-demo` directory and set up the local cluster:

```bash
cd ebpf-demo
```


The given command uses Kind to create a Kubernetes cluster using a YAML configuration. It specifies the cluster name, networking settings, and node roles. The configuration disables the default CNI, sets the pod and service subnets, and defines control-plane and worker nodes using a specific Docker image. The command combines the configuration provided through a multiline input with the kind create cluster command to create the cluster.

```bash
kind create cluster --name desocilium --config k8s/cluster.yaml
```

The command `kind get clusters` is used to retrieve a list of existing Kubernetes clusters managed by Kind. When executed, it will display the names of the available clusters that have been previously created using Kind. This command helps to quickly check and verify the names of the existing clusters within the Kind environment.

```
kind get clusters
```

> ```
> desocilium
> ```

List the kubernetes nodes:

```bash
kubectl get nodes -A
```

>```bash
> NAME                       STATUS     ROLES           AGE     VERSION
> desocilium-control-plane   NotReady   control-plane   7m1s    v1.31.0
> desocilium-worker          NotReady   <none>          6m46s   v1.31.0
> desocilium-worker2         NotReady   <none>          6m46s   v1.31.0
>```

Note the **“NotReady”** status.
This is because we have not yet installed a CNI plugin to provide the >networking.

Install Cilium CLI, Cilium is an open-source project that provides advanced networking and observability capabilities for Kubernetes clusters.

```bash
CILIUM_CLI_VERSION=v0.16.2
curl -L --fail --remote-name-all https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-amd64.tar.gz
sudo tar xzvfC cilium-linux-amd64.tar.gz /usr/local/bin
rm cilium-linux-amd64.tar.gz
```


Install a specific version of Cilium with a particular configuration option:


```
cilium install --version="1.16.2" -f k8s/cilium.yaml
```



After initializing the cluster, proceed to build the Docker image from the Dockerfile. This image includes all necessary components to run the eBPF program, such as BCC and the eBPF script named dddos.py. To view the contents of this script, run:


```bash
cat docker/dddos.py
```

To build the Docker image from the Dockerfile, use the following command:

```bash
cd docker
cat Dockerfile
docker build -t r.deso.tech/ebpf-demo/ebpf-probe .
cd ..
```

You may choose any name for the image, but ensure consistency throughout the tutorial. After creating the image, load it into the Kubernetes cluster with:

```bash
kind load docker-image r.deso.tech/ebpf-demo/ebpf-probe
```

With the image loaded, you can deploy an Nginx server to your cluster. The `k8s/ds.yaml` file includes configuration details for deploying a webserver with three replicas in the default namespace:

Deploy this manifest using the following command:

```bash
kubectl apply -f k8s/ds.yaml
```

You can access to the loadbalancer IP
