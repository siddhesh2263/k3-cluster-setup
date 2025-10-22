# Setting Up a Lightweight Kubernetes Cluster

<br>

## Table of Contents

- [Part 1 - Introduction](#part-1---introduction)
- [Part 2 - Hardware and system setup](#part-2---hardware-and-system-setup)
- [Part 3 - Setting up a 3-node K3s cluster](#part-3---setting-up-a-3-node-k3s-cluster)
- [Part 4 - Setting up kubectl on the development machine](#part-4---setting-up-kubectl-on-the-development-machine)
- [Part 5 - Deploying applications: Building, pushing, and managing Docker images with kubectl](#part-5---deploying-applications-building-pushing-and-managing-docker-images-with-kubectl)
- [Part 6 - Initial setup - Single master node with SQLite](#part-6---initial-setup---single-master-node-with-sqlite)
- [Part 7 - Upgrading to high availability (HA): Converting to etcd-based storage](#part-7---upgrading-to-high-availability-ha-converting-to-etcd-based-storage)
- [Part 8 - Summary of the K3s cluster setup guide](#part-8---summary-of-the-k3s-cluster-setup-guide)
- [Part 9 - Future improvements and next steps](#part-9---future-improvements-and-next-steps)
- [Part 10 - Troubleshooting notes](#part-10---troubleshooting-notes)

<br>

## Part 1 - Introduction

This project is about building a lightweight Kubernetes cluster from the ground up using K3s. The goal is to understand how multiple machines can work together as a single system to run and manage containerized applications. We're not using managed cloud services - we're setting up everything ourselves: the servers, the networking, the cluster orchestration.

The motivation behind this setup is to move beyond abstract concepts and actually see what it takes to get a distributed system running. What software is required? How do the machines communicate? What happens when one of them fails? These are questions that often get abstracted away in modern development workflows, but are critical to understand as a software developer. By building and managing our own cluster, we gain a clearer picture of how software is deployed, scaled, and kept running in the real world.

<br>

![alt text](https://github.com/siddhesh2263/k3-cluster-setup/blob/main/assets/server-merged.png?raw=true)

<br>

## Part 2 - Hardware and system setup

For this cluster, I used three Linux servers. They’re connected to the same network, providing the base for the K3s environment.

### What is K3s?

K3s is a lightweight, streamlined version of Kubernetes, designed for running clusters in resource-constrained environments. It simplifies setup by stripping away extra features and bundling core components into a single binary. K3s makes it easier to experiment with Kubernetes and deploy applications on smaller, more manageable clusters.

### Server Description:

All nodes in the cluster have:

* **Memory**: Between 8 and 16 GB of DDR3/4 RAM
* **CPU**: At least 4 threads, up to 12
* **Storage**: At least 256GB of storage, up to 1 TB

The cluster consists of:

* **Server 1 (master)** – handles the control plane, API server, and runs workloads.
* **Server 2 (worker)** – runs workloads
* **Server 3 (worker)** – runs workloads

The development system (labeled as "K3s User") connects to the cluster remotely, managing and deploying applications.

<br>

## Part 3 - Setting up a 3-node K3s cluster

This section covers the process of setting up our K3s cluster across three Linux servers. We'll go through installing K3s on the master node, retrieving the join token, and then adding the worker nodes to create a functional, multi-node cluster.

### Setup the master node:

The below command downloads the latest K3s binary, sets it up as a systemd service (`k3s.service`,) and then starts the K3s server using `SQLite` as the default data store:

```
curl -sfL https://get.k3s.io | sh -
```

Once the command is run, we can verify that the K3s server is running:

```
sudo kubectl get nodes
```

![alt text](https://github.com/siddhesh2263/k3-cluster-setup/blob/main/assets/k3s-master-setup.png?raw=true)

<br>

### Retrieve the join token and master node IP:

We need a token from the master node, which will be used to join the worker nodes into the cluster:

```
sudo cat /var/lib/rancher/k3s/server/node-token
```

The master node IP address can be found using:

```
hostname -I
```

<br>

### Setup the worker nodes:

On each worker node, run the below command:

```
curl -sfL https://get.k3s.io | K3S_URL=https://<MASTER_IP>:6443 K3S_TOKEN=<NODE_TOKEN> sh -
```

Replace the `MASTER_IP` and the `NODE_TOKEN` with the IP and join token retrieved from the master node.

<br>

### Verify worker nodes joined:

On the master node, run the below command:

```
sudo kubectl get nodes
```

The output should consist of 1 master node (with `control-plane` role,) and 2 worker nodes.

![alt text](https://github.com/siddhesh2263/k3-cluster-setup/blob/main/assets/verify-joined.png?raw=true)

<br>

## Part 4 - Setting up kubectl on the development machine

In this section, we'll set up `kubectl`, the command-line tool that lets us interact with the K3s cluster from our development machine. We'll copy the configuration file from the master node, adjust it so it points to the master's IP, and verify that we can view and manage cluster resources directly from our development environment.

The `kubectl` utility uses the `kubeconfig` file on the development machine to authenticate and communicate with the K3s API server.

### Retrieve the kubeconfig file:

On the master node, K3s stores the kubeconfig at:

```
sudo cat /etc/rancher/k3s/k3s.yaml
```

Copy this file's content.

### Transfer the kubeconfig to development machine:

On the development machine, first create a `.kube` directory if not present. Then, save the above copied `k3s.yaml` file to the `~/.kube/config`.

Replace the default `127.0.0.1` server address in the `~/.kube/config` with the master node's IP address. This change ensures the development machine can reach the K3s API server.

### Verify kubectl connectivity:

On the development machine, run the below command:

```
kubectl get nodes
```

1 master node and 2 worker nodes should be visible.

![alt text](https://github.com/siddhesh2263/k3-cluster-setup/blob/main/assets/windows-kubectl-nodes.png?raw=true)

<br>

## Part 5 - Deploying applications: Building, pushing, and managing Docker images with kubectl

This section focuses on turning our development work into running applications within the cluster. We'll build Docker images, push them to a container registry, and create deployment YAML files to define how these images run in K3s. Finally, we'll use `kubectl` to apply these configurations and manage our applications directly in the cluster.

### Build the Docker image, and push to container registry:

Go to the app's root directory (wherever the Dockerfile is present,) and run the below command:

```
docker build -t siddhesh2263/api-gateway:latest .
```

Once image is built, push it to Docker Hub:

```
docker push siddhesh2263/api-gateway:latest
```

<br>

### Create a Kubernetes deployment YAML:

An example of a YAML file would be something like this:

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-gateway
spec:
  replicas: 2
  selector:
    matchLabels:
      app: api-gateway
  template:
    metadata:
      labels:
        app: api-gateway
    spec:
      containers:
      - name: api-gateway
        image: siddhesh2263/api-gateway:latest
        ports:
        - containerPort: 5000  # Change to your app's port

```

Save it as `api-gateway-deployment.yaml`.

<br>

### Deploy to the K3s cluster with kubectl:

Apply the deployment from the development machine CLI:

```
kubectl apply -f api-gateway-deployment.yaml
```

Once this is done, verify if the pods are running:

```
kubectl get deployments
kubectl get pods
```

To check for any errors/issues, fetch the logs:

```
kubectl logs <pod_name>
```

<br>

## Part 6 - Initial setup - Single master node with SQLite

### Overview:

We started with a single master node using SQLite as the default datastore in K3s. This approach is ideal for development and testing because SQLite requires minimal configuration and resources - it's embedded direclty into the K3s binary, so there's no need to set up an external database. This lightweight setup allows us to bring up a functional Kubernetes environment quickly and understand how K3s organizes resources and manages workloads.

![alt text](https://github.com/siddhesh2263/k3-cluster-setup/blob/main/assets/sqlite-k3.png?raw=true)

K3s runs a control plane on the master node, storing the cluster state in `kine.sqlite` located at:

```
/var/lib/rancher/k3s/server/db/state.db
```

When K3s was installed, it defaulted to using SQLite wihtout any extra config.

<br>

### Issues with this setup:

This setup comes with significant limitations. The biggest drawback is the lack of fault tolerance - if the master node goes down, the entire cluster’s state is at risk since SQLite doesn’t support distributed storage. This setup is best suited for local development or experimentation, not for critical or highly available applications.

<br>

## Part 7 - Upgrading to high availability (HA): Converting to etcd-based storage

### Overview:

To ensure the cluster can withstand failures and stay available, we upgrade from SQLite to etcd as the backing data store. By running etcd on all master nodes, we create a highly available data layer that automatically synchronizes state. This change transforms our cluster from a single-point-of-failure system into a more resilient system.

![alt text](https://github.com/siddhesh2263/k3-cluster-setup/blob/main/assets/etcd-k3.png?raw=true)

<br>

### Install K3s on the first master node (cluster init):

As this stage, although not ideal, I removed the existing single master node K3s setup, and have started a clean HA K3s cluster setup from the start.

On the first master node, run the below command:

```
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="server --cluster-init" sh -
```

This starts etcd and sets up the first master as the initial etcd member.

<br>

### Get the join token from the first master:

On the first master node, copy the token present at the below directory. This will be used by the other master nodes:

```
sudo cat /var/lib/rancher/k3s/server/node-token
```

<br>

### Join additonal master nodes to the cluster:

On the 2nd and the 3rd master nodes, execute the below command:

```
curl -sfL https://get.k3s.io | K3S_TOKEN=<NODE_TOKEN> \
INSTALL_K3S_EXEC="server --server https://<MASTER_1_IP>:6443" sh -
```

The `NODE_TOKEN` and the `MASTER_1_IP` are taken from the first master node.

<br>

### Verify all master nodes joined the cluster:

On any master node (or the development machine,) run the below command:

```
kubectl get nodes
```

It should show the nodes with the `control-plane`, `etcd` tag attached.

![alt text](https://github.com/siddhesh2263/k3-cluster-setup/blob/main/assets/ha-setup-nodes.png?raw=true)

<br>

## Part 8 - Summary of the K3s cluster setup guide

We set up a 3-node K3s cluster starting with a single master node using SQLite as the data store, then expanded to a highly available setup using etcd.

* We began by installing K3s on the master and joining worker nodes using the join token and IP address of the master.

* We configured `kubectl` on a development machine to interact with the cluster remotely.

* We walked through how to build Docker images, push them to a registry, and deploy them on the K3s cluster with `kubectl`.

* After identifying limitations of the single-master SQLite setup (no fault tolerance), we transitioned to HA by cleaning up the single-node cluster and starting a fresh setup.

* Finally, we used the `--cluster-init` flag on the first master to start etcd, and joined the other two masters as additional etcd members for full HA capability.

<br>

## Part 9 - Future improvements and next steps

While this setup gets us up and running with a functional K3s cluster, there's still a lot to refine. In this section, we'll look at what's missing - like security, backups, and monitoring.

### Security and access control:
Right now, the cluster has no strict rules on who can access it. In the future, we’ll need to set up better ways to control who can see and change what - like roles and permissions that ensure only trusted users or services can do sensitive tasks.

### Backups and recovery:
If something goes wrong, we don’t want to lose everything. We’ll set up ways to automatically back up the cluster’s data and test restoring from those backups.

### Storing persistent data:
Some applications need to save data that sticks around, even if they restart. We’ll explore how to provide reliable storage that can handle this data properly.

### Monitoring and logs:
To know what’s happening in the cluster, we’ll need tools that can show us performance numbers, errors, and warnings in one place. This will help us keep the cluster healthy and fix issues quickly.

### Network reliability and load balancing:
We’ll look into setting up systems that balance incoming traffic so no single server gets overwhelmed. This makes the cluster more resilient and ensures it can handle more users and data.

### Updates and maintenance:
Keeping everything up to date without breaking things is important. We’ll create a plan for updating the cluster software while keeping services running smoothly.

### Testing high availability:
We’ve set up high availability, but we need to regularly test what happens if a server goes down - so we know for sure the cluster can recover without issues.

### Managing secrets and sensitive data:
Applications often have sensitive data like passwords. We’ll look into better ways to store these securely and avoid accidental leaks.

### Better deployment practices:
We’ll improve how we roll out updates to applications in the cluster, so they can start faster, scale properly, and stay healthy even under load.

<br>

## Part 10 - Troubleshooting notes

### Issue 1 - kubectl configuration confusion:

I initially faced issues where `kubectl` was using a different `kubeconfig` file location than where I was making my updates. This caused confusion because changes to the config file weren’t taking effect in the `kubectl` commands. I realized I needed to ensure that the environment variable `KUBECONFIG` or the default `~/.kube/config` matched the file I was actually editing.

### Issue 2 - WiFi adapter compatibility:

Another challenge was getting the WiFi adapters to work with Linux. While the adapters worked fine in Windows, they didn’t have Linux drivers out of the box. I had to source different WiFi adapters that were supported by Linux to ensure the servers could connect to the network.