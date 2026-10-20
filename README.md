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

The stockmanager startup/liveness probes use Actuator's `/actuator/health/liveness`
endpoint, and readiness uses `/actuator/health/readiness`, which also checks the
database connection.

## Configure stockmanager with external PostgreSQL

Stockmanager uses PostgreSQL for Kubernetes deployments. Update
`kubernetes/stockmanager-configmap.yaml` with the managed database hostname,
port, database name, and the SSL mode required by your provider. The committed
values are placeholders and must be replaced before applying the manifest.

The local file `kubernetes/stockmanager-secret.yaml` is Git-ignored. Edit it
and set `DB_USERNAME` and `DB_PASSWORD` to your database credentials. Do not
commit or share this file. Apply it before the Deployment:

```powershell
notepad .\kubernetes\stockmanager-secret.yaml
kubectl apply -f kubernetes/stockmanager-secret.yaml
```

Build the updated stockmanager image and load it into the local kind cluster:

```powershell
mvn -f stockmanager\pom.xml package -DskipTests
docker build -t stockmanager:dev .\stockmanager
kind load docker-image stockmanager:dev
```

Apply the database configuration and stockmanager resources, then check the
Pod and its logs:

```powershell
kubectl apply -f kubernetes/stockmanager-configmap.yaml
kubectl apply -f kubernetes/stockmanager-secret.yaml
kubectl apply -f kubernetes/stockmanager-service.yaml
kubectl rollout restart deployment/stockmanager
kubectl rollout status deployment/stockmanager
kubectl logs deployment/stockmanager
```

Repeat the rollout restart after changing ConfigMap values because environment
variables sourced from a ConfigMap are read when the Pod starts.

The managed database must accept network connections from the Kubernetes
cluster and have the requested database created. `DB_SSL_MODE=require`
encrypts the connection; follow your provider's guidance for certificate
verification in production. The application uses Hibernate `ddl-auto=update`
to create/update tables for this demo; production deployments should use
versioned database migrations. Demo stock records are inserted only when the
database has no stock rows, so restarting the Pod does not overwrite existing
inventory.

Kubernetes Secrets are not a substitute for a secrets manager: by default,
their values are base64-encoded, not encrypted. Restrict cluster access and
enable encryption at rest or use your platform's secret-management integration
for production.
