# Kubernetes

This file includes instructions on how to setup Kubernetes and an image registry.

## Install Kubernetes

The easiest way to install Kubernetes in our experience is to use [k3sup](https://github.com/alexellis/k3sup), a "light-weight utility to get from zero to KUBECONFIG with k3s on any local or remote VM"

In order to install k3sup, it's enough to run:
```bash
curl -sLS https://get.k3sup.dev | sh
sudo install k3sup /usr/local/bin/
rm k3sup
```

After cloning the ctf-on-k8s repository on the server, we just need to run the following command:

```bash
k3sup install --local
```

Note that `k3sup` creates the `kubeconfig` file in the current working directory from where the command is run. We can move it to the default location with:

```bash
mkdir -p $HOME/.kube
mv kubeconfig $HOME/.kube/config
```

### Configure Kubernetes

#### Increase the Maximum Number of Pods Allowed

After installing Kubernetes, we need to increase the maximum number of pods allowed per node, if we plan to deploy many challenges. This can be done by editing the `k3s.service` file:

```bash
sudo vim /etc/systemd/system/k3s.service
```

and adding `'--kubelet-arg=max-pods=433'` to the end of the `ExecStart` command.

After that, we need to reload the systemd configuration and restart k3s:

```bash
sudo systemctl daemon-reload
sudo systemctl restart k3s
```

We can check the nodes by running:
```bash
sudo kubectl get nodes
```

We can check the number of pods allowed per node with the following command:

```bash
sudo kubectl get nodes <NODE_NAME> -o json | jq -r '.status.capacity.pods'
```

#### Set the Range of Node Ports

Since challenges are deployed on ports <30000 (in our case, you can decide to go higher than that though), we need to lower the range of ports allowed in the node port field of services. To do that, we need to edit the `k3s.service` file again:

```bash
sudo vim /etc/systemd/system/k3s.service
```

and add `'--service-node-port-range=MIN_PORT-MAX_PORT'` to the end of the `ExecStart` command.

After this, the `ExecStart` command should be something like this:
```bash
ExecStart=/usr/local/bin/k3s \
    server \
	'--tls-san' \
	'127.0.0.1' \
	'--kubelet-arg=max-pods=433' \
	'--service-node-port-range=30000-31000'
```

Then, we need to reload the systemd configuration and restart k3s:

```bash
sudo systemctl daemon-reload
sudo systemctl restart k3s
```

**Note**: the `MIN_PORT` should be high enough to not collide with previously existing services (it is recommended at least above 7000). Even better would be to have the challenges directly within the predefined Kubernetes range >30000.

## Setup a Local Registry

To deploy a local registry, we can use the official Docker registry image. We can create a Kubernetes deployment and service for the registry as follows.
`cd` into the `kubernetes/` folder of this repository, and run:

```bash
sudo kubectl create namespace docker-registry
```

Then, in the same folder, create this `docker-registry.yaml` file:

```yaml
# Local Docker registry running in Kubernetes - for k3s
# docker-registry.yaml
#---

apiVersion: v1
kind: Service
metadata:
  name: docker-registry-service
  labels:
    app: docker-registry
spec:
  selector:
    app: docker-registry
  ports:
    - protocol: TCP
      port: 5000

---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: docker-registry-pvc
  labels:
    app: docker-registry
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: local-path
  resources:
    requests:
      storage: 10Gi

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: docker-registry
  labels:
    app: docker-registry
spec:
  replicas: 1
  selector:
    matchLabels:
      app: docker-registry
  template:
    metadata:
      labels:
        app: docker-registry
    spec:
      containers:
      - name: docker-registry
        image: registry:2
        ports:
        - containerPort: 5000
          protocol: TCP
        volumeMounts:
        - name: storage
          mountPath: /var/lib/registry
        env:
        - name: REGISTRY_HTTP_ADDR
          value: :5000
        - name: REGISTRY_STORAGE_FILESYSTEM_ROOTDIRECTORY
          value: /var/lib/registry
      volumes:
      - name: storage
        persistentVolumeClaim:
          claimName: docker-registry-pvc
```

This YAML file creates a service and a deployment for the Docker registry, along with a persistent volume claim to store the images. You can apply this configuration with the following command:

```bash
kubectl apply -f docker-registry.yaml -n docker-registry
```

If you want to check the successful result of the deployment operation, you can run:
```bash
sudo kubectl get deployment,service,persistentvolumeclaim -n docker-registry
```

and you should see:
```
NAME                              READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/docker-registry   1/1     1            1           2m22s

NAME                              TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
service/docker-registry-service   ClusterIP   10.43.21.127   <none>        5000/TCP   2m23s

NAME                                        STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
persistentvolumeclaim/docker-registry-pvc   Bound    pvc-61e4e2cd-13e9-49cb-bf9e-d95fb079b410   10Gi       RWO            local-path     <unset>                 2m22s
```

We need to configure k3s to use the local registry by adding the following to the `/etc/rancher/k3s/registries.yaml` file, or create the file with the following, if it does not exist yet:

```yaml
mirrors:
  registry.localhost:
    endpoint:
      - "http://registry.localhost:5000"
```

We may need to restart k3s for the changes to take effect:

```bash
sudo systemctl restart k3s
```

Finally, we need to configure the DNS to resolve `registry.localhost` to the IP address of the Kubernetes node. This can be done by adding an entry to the `/etc/hosts` file on the server:

```bash
REGISTRY_IP=$(sudo kubectl get svc docker-registry-service -n docker-registry -o jsonpath='{.spec.clusterIP}')
echo "$REGISTRY_IP registry.localhost" | sudo tee -a /etc/hosts
```

After deploying the registry, we can push images to it using the following command:

```bash
docker build -t registry.localhost/test:latest .
docker push registry.localhost/test:latest
```

## Python env

The script `deployer/kubernetes_deployer.py` requires specific modules. The fastest way to do this is to create an environment with [micromamba](https://mamba.readthedocs.io/en/latest/installation/micromamba-installation.html).

After installation, create the enviroment by using, inside `kubernetes/`:

```bash
micromamba create -f micromamba-env.yaml
```

It will automatically install all the dependencies. To activate it, simply use:

```bash
micromamba activate ctfkube
```

## k9s

To manage the Kubernetes cluster, the best tool in our opinion is [k9s](https://k9scli.io/), a terminal-based UI to interact with the Kubernetes cluster.
