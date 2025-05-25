# Setting Up a Lightweight K3s Cluster

![alt text](https://github.com/siddhesh2263/k3-cluster-setup/blob/main/assets/servers.png?raw=true)

<br>

## Part 1 - Setting up a 3-node K3s cluster

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

## Part 2 - Setting up kubectl on the development machine

The kubectl utility uses the `kubeconfig` file on the development machine to authenticate and communicate with the K3s API server.

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

## Part 3 - Deploying applications: Building, pushing, and managing Docker images with kubectl

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

