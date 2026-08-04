# eva-webservices-new

## Build

Build the jar:
```
mvn clean package -DskipTests
```

Build the Docker image:
```
docker build -t eva-webservices-new:local .
```

## Configuration

Non-secret defaults live in `src/main/resources/application.properties`.
Database connection details:

- In Kubernetes, from a Secret mounted at `/app/config/application.properties`
  (see `k8s-manifests/`).
- For local IDE / `mvn spring-boot:run`, via a git-ignored
  `./config/application.properties` at the repo root (Spring Boot loads
  `config/` relative to the working directory automatically).

Required keys:
- `eva.mongo.uri`
- `eva.mongo.accessioning-database`
- `spring.datasource.url`
- `spring.datasource.username`
- `spring.datasource.password`

## Deploying to dev or staging

Prerequisites: `KUBECONFIG` needs to be set to the target
cluster (dev or staging), and `k8s-manifests/overlays/<dev|staging>/application.properties` already exists
locally with real credentials (see [Deployment](#deployment) above.

Everything below is the same for both environments; just swap `dev` for
`staging` (namespace `eva-webservices-new-dev` / `eva-webservices-new-stage`,
overlay path `k8s-manifests/overlays/dev` / `k8s-manifests/overlays/staging`).
1. Login to the Docker registry 
   ```
   docker login dockerhub.ebi.ac.uk
   ```
2. Build the image (`--platform linux/amd64` only matters when building from a Mac):
   ```
   TAG=eva-webservices-new-manual-$(date +%Y%m%d-%H%M)
   docker build --platform linux/amd64 -t dockerhub.ebi.ac.uk/ebivariation/eva-seqcol:$TAG .
   ```
   (Borrowing the `ebivariation/eva-seqcol` repo as scratch space - see
   the `FIXME` in the overlay `kustomization.yaml` files.)
3. Push it:
   ```
   docker push dockerhub.ebi.ac.uk/ebivariation/eva-seqcol:$TAG
   ```
4. Bump the tag in the target overlay:
   ```
   sed -i '' "s/newTag: .*/newTag: $TAG/" k8s-manifests/overlays/dev/kustomization.yaml
   ```
   (or edit the `newTag:` line by hand.)
5. Apply:
   ```
   kubectl apply -k k8s-manifests/overlays/dev
   ```
6. Wait for the rollout and confirm pods are healthy:
   ```
   kubectl rollout status deployment/eva-webservices-new -n eva-webservices-new-dev --timeout=180s
   kubectl get pods -n eva-webservices-new-dev 
   ```
7. Exercise a real endpoint:
   ```
   curl -i "https://wwwdev.ebi.ac.uk/eva/webservices/eva-webservices-new/v3/rsids"
   curl -i "https://wwwint.ebi.ac.uk/eva/webservices/eva-webservices-new/v3/rsids"
   ```
8. If anything looks wrong, check logs and pod events:
   ```
   kubectl logs -n eva-webservices-new-dev deployment/eva-webservices-new --tail=200 -f
   kubectl describe pod -n eva-webservices-new-dev <pod-name>
   ```
