# Assumptions

- You installed PCF Ops Manager on GCP using [Terraforming GCP](https://github.com/pivotal-cf/terraforming-gcp)
- Your terraform.tfvars file used the `pks="true"` flag
- You have deployed the PKS v1.1+ tile
- `PCF_SUBDOMAIN_NAME` and `PCF_DOMAIN_NAME` are set appropriately to identify your PCF instance
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

# To create a cluster

```bash
pks login \
  --api api.${PCF_PKS} \
  --username admin \
  --password ${PCF_PKS_UAA_ADMIN_PASSWORD} \
  --skip-ssl-validation
  
pks create-cluster k8s \
  --external-hostname k8s.${PCF_PKS} \
  --plan small  \
  --num-nodes 1 \
  --wait
```

# To expose a cluster

- Add a `k8s.pks.${PCF_SUBDOMAIN_NAME}.${PCF_DOMAIN_NAME}` entry in Cloud DNS pointing at the master node's external IP address

# To expose a deployment

- Deployed app must have NodePort service. Note its high-order port number
- Add a firewall named "my-app" for that port with target set to "worker"
- In Load Balancing (advanced):
  - Add a target pool named "my-app" that explicitly targets all the worker VMs
  - Create a forwarding rule named "my-app" that targets the port of the NodePort service
  
The forwarding rule will be assigned an external IP.

Navigate to `http:/[LOAD_BALANCER_IP]:[HIGH_ORDER_PORT_NUMBER]`
