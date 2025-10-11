# Create secrets for external-dns, cert-manager, and other components

```bash
kubectl create ns external-dns
kubectl create secret generic azure-config-file -n external-dns --from-file=azure.json=/Users/alessandro/repos/scratch/kubeconfigs/external-dns.json
kubectl create configmap -n external-dns txt-owner-id --from-literal=txt-owner-id=susemeetup
```

```bash
az ad sp create-for-rbac --name extdns --role "DNS Zone Contributor" --scopes /subscriptions/<subscription_with_dns_zones>/resourceGroups/dns -o json
export CLIENT_SECRET="secret" 
kubectl create ns cert-manager
kubectl create secret generic azuredns-config -n cert-manager --from-literal=client-secret=${CLIENT_SECRET}
```
