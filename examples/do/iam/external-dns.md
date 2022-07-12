## External DNS

DigitalOcean does not offer an IAM service. Instead of that, they use API tokens to communicate directly using the API.

To be able to communicate with the API, External DNS needs to know and use that token, so you must provide it as a 
Kubernetes Secret resource as follows:

```console
kubectl create secret generic external-dns-environment -n external-dns --from-literal DO_TOKEN="your-token-here"
```
