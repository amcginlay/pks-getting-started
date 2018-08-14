```bash
pks login -a api.pks.${PCF_SUBDOMAIN_NAME}.${PCF_DOMAIN_NAME} -u admin -p ${UAA_ADMIN_PASSWORD}
pks create-cluster k8s --plan small  --external-hostname k8s.pks.${PCF_SUBDOMAIN_NAME}.${PCF_DOMAIN_NAME} --num-nodes 1 --wait
```
