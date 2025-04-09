# Kubernetes Web Server with NFS Storage on Amazon EKS

This project demonstrates the deployment of a simple HTTP web server running in a Kubernetes cluster hosted on Amazon EKS. The application uses an NFS server (deployed via Helm) to dynamically provision persistent storage, which is shared between a content population Job and an Nginx web server.

---

## Prerequisites

To deploy this application, ensure the following tools are installed and configured locally:

- [kubectl](https://kubernetes.io/docs/tasks/tools/)
- [eksctl](https://eksctl.io/)
- [helm](https://helm.sh/)
- AWS CLI configured via `aws configure` (with access to create EKS clusters and EC2 resources)

---

## Deployment Steps

### 1. Create an EKS Cluster

Use `eksctl` to create a new EKS cluster with two managed nodes:

```bash
eksctl create cluster \
  --name nfs-cluster \
  --region us-east-1 \
  --nodes 2 \
  --managed
```

This command will also update your kubeconfig to point to the new cluster.

### 2. Install the NFS Server Provisioner with Helm

Add the Helm repository and install the `nfs-server-provisioner` chart:

```bash
helm repo add stable https://charts.helm.sh/stable
helm repo update

helm install nfs-server-provisioner stable/nfs-server-provisioner \
  --set persistence.enabled=false \
  --set storageClass.name=nfs-sc \
  --set storageClass.defaultClass=true
```

This installs the NFS server and configures a dynamic storage class named nfs-sc.

### 3. Create a Persistent Volume Claim

Apply the Persistent [Volume Claim YAML](https://github.com/kacpercwiertnia/lsc_kubernetes/blob/main/pvc.yaml):

```bash
kubectl apply -f ./manifests/pvc.yaml
```

This PVC will be dynamically provisioned by the NFS provisioner and used by the HTTP server and content copy job.

### 4. Create a Persistent Volume Claim

Apply the [Deployment YAML](https://github.com/kacpercwiertnia/lsc_kubernetes/blob/main/deployment.yaml):

```bash
kubectl apply -f ./manifests/deployment.yaml
```

The nginx container mounts the NFS volume at /usr/share/nginx/html so that it can serve dynamic content written to the volume.

### 5. Expose the Web Server via LoadBalancer Service

Apply the [Service YAML](https://github.com/kacpercwiertnia/lsc_kubernetes/blob/main/service.yaml):

```bash
kubectl apply -f ./manifests/service.yaml
```

The service will expose the nginx deployment publicly. You can retrieve the external IP using:

```bash
kubectl get svc web-service
```

### 6. Populate the Web Content using a Job

Apply the [Job YAML](https://github.com/kacpercwiertnia/lsc_kubernetes/blob/main/job.yaml):

```bash
kubectl apply -f ./manifests/job.yaml
```

This job writes a sample index.html file to the shared volume that is mounted by the nginx server.

### 7. Test in Browser

Once the service has an external IP, open it in your browser. You should see:

![Service working](https://github.com/kacpercwiertnia/lsc_kubernetes/blob/main/service_working.png)

This job writes a sample index.html file to the shared volume that is mounted by the nginx server.