# Assumptions

- You installed PCF Ops Manager on GCP using [Terraforming GCP](https://github.com/pivotal-cf/terraforming-gcp)
- Your terraform.tfvars file used the `pks="true"` flag
- You have deployed the PKS v1.1+ tile
- `GCP_PROJECT_ID`, `PCF_SUBDOMAIN_NAME` and `PCF_DOMAIN_NAME` are set appropriately to identify your PCF instance
- You have `gcloud` cli tool installed and authenticated
- You have `pks` and `kubectl` cli tools installed locally (available from [PivNet](https://network.pivotal.io/products/pivotal-container-service/))

# Set shortcut variables
- Execute `PCF_OPSMAN=pcf.${PCF_SUBDOMAIN_NAME}.${PCF_DOMAIN_NAME}`
- Execute `PCF_PKS=pks.${PCF_SUBDOMAIN_NAME}.${PCF_DOMAIN_NAME}`

# Find the product guid and UAA admin password for PKS

1. Navigate to `https://${PCF_OPSMAN}/api/v0/deployed/products`
1. Identify the guid for product with `"type": "pivotal-container-service"`
1. Store this value in a shell variable named `PCF_PKS_GUID`
1. Navigate to `https://${PCF_OPSMAN}/api/v0/deployed/products/${PCF_PKS_GUID}/credentials/.properties.uaa_admin_password`
1. Identify the value of `credential.value.secret`
1. Store this value in a shell variable named `PCF_PKS_UAA_ADMIN_PASSWORD`

# Connect to PKS

```bash
pks login \
  --api api.${PCF_PKS} \
  --username admin \
  --password ${PCF_PKS_UAA_ADMIN_PASSWORD} \
  --skip-ssl-validation
```

# Create your Kubernetes cluster

```bash
pks create-cluster k8s \
  --external-hostname k8s.${PCF_PKS} \
  --plan small  \
  --num-nodes 1 \
  --wait
```

# Discover the external IP of the master node

```bash
K8S_MASTER_INTERNAL_IP=$(pks cluster k8s --json | jq --raw-output '.kubernetes_master_ips[0]')
K8S_MASTER_EXTERNAL_IP=$(gcloud compute instances list --project ${GCP_PROJECT_ID} --format json | \
  jq --raw-output --arg V "${K8S_MASTER_INTERNAL_IP}" '.[] | select(.networkInterfaces[].networkIP | match ($V)) | .networkInterfaces[].accessConfigs[].natIP')
```

# Create an A Record in DNS for your cluster

```bash
gcloud dns record-sets transaction start --project ${GCP_PROJECT_ID} --zone=${PCF_SUBDOMAIN_NAME}-zone

  gcloud dns record-sets transaction \
    add ${K8S_MASTER_EXTERNAL_IP} \
    --name=k8s.${PCF_PKS}. \
    --project ${GCP_PROJECT_ID} \
    --ttl=300 --type=A --zone=${PCF_SUBDOMAIN_NAME}-zone

gcloud dns record-sets transaction execute --project ${GCP_PROJECT_ID} --zone=${PCF_SUBDOMAIN_NAME}-zone
```

# Verify DNS

This step may take about 5 minutes to propagate.

When run locally, the following watch commands should eventually yield different external IP addresses

```bash
watch dig api.${PCF_PKS}
watch dig k8s.${PCF_PKS}
```

# Attach the client to your cluster

```bash
pks get-credentials k8s
```

# Allow remote access to the k8s dashboard

```bash
kubectl create -f - <<-EOF
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: kubernetes-dashboard
  labels:
    k8s-app: kubernetes-dashboard
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: kubernetes-dashboard
  namespace: kube-system
EOF
```

# Open a proxy in the background

```bash
kubectl proxy &
```

# Inspect the Kubernetes dashboard

Navigate to `http://localhost:8001/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy`

# Deploy the nginx webserver docker image

```bash
kubectl create -f - <<-EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webserver
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
        ports:
        - containerPort: 80
EOF
```

# Inspect our deployment

```bash
kubectl get deployments
kubectl get replicasets
kubectl get pods
```

# Expose our deployment

Create a service to expose a port for our webserver:

```bash
kubectl create -f - <<-EOF
apiVersion: v1
kind: Service
metadata:
  name: web-service
  labels:
    run: web-service
spec:
  type: NodePort
  ports:
  - port: 80
    protocol: TCP
  selector:
    app: nginx
EOF
```

Extract the port:

```bash
NODEPORT=$(kubectl get services --output=json | \
  jq '.items[] | select(.metadata.name=="web-service") | .spec.ports[0].nodePort')
```

Add a firewall rule for the web-service port

```bash
gcloud compute firewall-rules create nginx \
  --network=${PCF_SUBDOMAIN_NAME}-pcf-network \
  --action=ALLOW \
  --rules=tcp:${NODEPORT} \
  --target-tags=worker \
  --project=${GCP_PROJECT_ID}
```

Add a target pool to represent all the worker nodes:

```bash
gcloud compute target-pools create "nginx" \
  --region "us-central1" \
  --project "${GCP_PROJECT_ID}"
  
for instance in $(gcloud compute instances list --project=${GCP_PROJECT_ID} --filter="tags.items:worker" --format="value(name)"); do
  gcloud compute target-pools add-instances "nginx" \
    --project "ps-amcginlay" \
    --instances-zone "us-central1-a" \
    --instances "${instance}"
done
```

Create a forwarding rule to expose our app:

```bash
gcloud compute forwarding-rules create nginx2 \
  --region=us-central1 \
  --network-tier=STANDARD \
  --ip-protocol=TCP \
  --ports=${NODEPORT} \
  --target-pool=nginx2 \
  --project=${GCP_PROJECT_ID}
```

Extract the external IP address:

```bash
LOAD_BALANCER_IP=$(gcloud compute forwarding-rules list \
  --filter="name:nginx2" \
  --format="value(IPAddress)" \
  --project=${GCP_PROJECT_ID})
```

Navigate to `http:/[LOAD_BALANCER_IP]:[NODEPORT]`
