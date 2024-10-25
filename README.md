# How to Deploy eBPF Demo on Kubernetes

## Prerequisites

To complete the tutorial, you'll need all of the following:

- A Kubernetes cluster: You need an up-and-running Kubernetes cluster that you'll observe using the eBPF DDoS detector. You can use any local cluster, such as kind, minikube, or K3s. This tutorial uses kind.
- The kubectl CLI: The kubectl CLI provides access to your Kubernetes cluster resources and allows you to issue commands. You can install kubectl by following the steps in the official documentation.
- BCC: BCC is a toolkit that makes it easy to create eBPF programs. It allows engineers to make kernel instrumentation in C and has Python and Lua frontends. You can install it by following the official installation instructions.
- hping3: hping3 is a CLI-based flooder that will be used to simulate a DDoS attack on your Kubernetes cluster. You can use any other tool of your choice here.

## Demo

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


Delete Daemonset and the configmap as well to avoid kube-proxy being reinstalled during a Kubeadm upgrade


```bash
kubectl -n kube-system delete ds kube-proxy
kubectl -n kube-system delete cm kube-proxy
```


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
docker image ls
```

> ```bash
> REPOSITORY                         TAG       IMAGE ID       CREATED         SIZE
> r.deso.tech/ebpf-demo/ebpf-probe   latest    6374cadc8d5c   2 minutes ago   310MB
> ```

```bash
kind load docker-image r.deso.tech/ebpf-demo/ebpf-probe --name desocilium
```

With the image loaded, you can deploy an webserver to your cluster. The `k8s/deploy.yaml` file includes configuration details for deploying a webserver with three replicas in the default namespace:


Deploy this manifest using the following command:

```bash
kubectl apply -f k8s/deploy.yaml
```

> ```
> deployment.apps/webserver-deployment created
> service/webserver-service created
> ```


After deploying a webserver in your local Kubernetes cluster, you can access it via the internal IP and NodePort.
Retrieve the internal IP of your kind cluster:

```bash
kubectl get node -o wide
```
Access the webserver:

```bash
http://<INTERNAL-IP>:30080
```

Now, deploy the eBPF program using a Kubernetes DaemonSet for full cluster coverage, improving observability and enabling DDoS detection:


```bash
kubectl apply -f k8s/ds.yaml
```

Now that the eBPF program is ready to observe incoming DDoS attacks on your cluster, you can simulate an attack and see if the program detects it. You can easily simulate a DDoS attack on your kind cluster by supplying the <INTERNAL-IP> to a flooder like hping3:

```
sudo hping3 10.10.10.10 -S -A -V -p 30080 -i u100
```

Check the logs to verify if the eBPF program detected the attack:

```bash
kubectl logs ds/ebpf-daemonset --tail 20
```

The name of the DaemonSet pod will be different on your machine, so make sure you replace that accordingly.

This command should print out the last twenty lines of output from the eBPF container, indicating that your program detected the DDoS simulation as expected:

Conclusion

eBPF offers powerful observability with minimal overhead in Kubernetes. By deploying an eBPF program as a DaemonSet, we can efficiently detect network anomalies like DDoS attacks. With tools like BCC, writing and implementing eBPF programs becomes accessible, improving troubleshooting and cluster monitoring.
