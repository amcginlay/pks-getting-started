# To create a cluster

```bash
pks login \
  -a api.pks.${PCF_SUBDOMAIN_NAME}.${PCF_DOMAIN_NAME} \
  -u admin \
  -p ${UAA_ADMIN_PASSWORD}
  
pks create-cluster k8s \
  --external-hostname k8s.pks.${PCF_SUBDOMAIN_NAME}.${PCF_DOMAIN_NAME} \
  --plan small  \
  --num-nodes 1 \
  --wait
```

# To expose a cluster

- Add k8s.pks.${PCF_SUBDOMAIN_NAME}.${PCF_DOMAIN_NAME} entry in Cloud DNS pointing at master node

# To expose a deployment

- Deployed app must have NodePort service. Note its high-order port number
- Add a firewall named "my-app" for that port with target set to "worker"
- In Load Balancing (advanced):
  - Add a target pool named "my-app" that explicitly targets all the worker VMs
  - Create a forwarding rule named "my-app" that targets the port of the NodePort service
  
The forwarding rule will be assigned an external IP.

Navigate to `http:/[LOAD_BALANCER_IP]:[HIGH_ORDER_PORT_NUMBER]`
