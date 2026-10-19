# docker-Java-kubernetes-project
Deploying Java Applications with Docker and Kubernetes

Credit: https://github.com/danielbryantuk/oreilly-docker-java-shopping/

## Accessing the services through Ingress

`kubernetes/ingress.yaml` routes the three apps by hostname:

- `stockmanager.local` to stockmanager
- `productcatalogue.local` to productcatalogue
- `shopfront.local` to shopfront

The cluster must have an Ingress controller installed that uses the `nginx`
IngressClass. Apply the Kubernetes manifests with:

```powershell
kubectl apply -f kubernetes/
kubectl get ingressclass
kubectl get ingress shopping-ingress
```

For local testing, forward a local port to the ingress-nginx controller (not to
an individual app):

```powershell
kubectl port-forward -n ingress-nginx service/ingress-nginx-controller 8080:80
```

In another terminal, send requests with the desired hostname in the `Host`
header:

```powershell
curl.exe -i -H "Host: stockmanager.local" http://localhost:8080/stocks
curl.exe -i -H "Host: productcatalogue.local" http://localhost:8080/products
curl.exe -i -H "Host: shopfront.local" http://localhost:8080/
```

The stockmanager probe uses Spring Boot Actuator's `/actuator/health` endpoint;
the Pod must be Ready for Ingress traffic to reach it.
