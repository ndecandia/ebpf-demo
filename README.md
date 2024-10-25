# How to Deploy eBPF Demo on Kubernetes

The code snippets and configurations used in this guide are available in a Git repository. You can download the repository to your Linux machine by executing:

```bash
git clone git@github.com:ndecandia/ebpf-demo.git
```

Navigate to the `loft-ebpf-test` directory and set up the local cluster:

```bash
cd ebpf-demo
kind create cluster
```


After initializing the cluster, proceed to build the Docker image from the Dockerfile. This image includes all necessary components to run the eBPF program, such as BCC and the eBPF script named dddos.py. To view the contents of this script, run:


```bash
cat docker/dddos.py
```

To build the Docker image from the Dockerfile, use the following command:

```bash
cd docker
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
