# springboot-hello-world

Este proyecto es un microservicio simple de Spring Boot que expone un endpoint REST de saludo. Está preparado para ser desplegado en AWS EKS (Elastic Kubernetes Service) usando Docker y manifiestos de Kubernetes.

---

## Características

- Endpoint REST en `/` que retorna "Hello from Spring Boot!"
- Dockerfile para contenerización
- Manifiestos de Kubernetes para despliegue y servicio

---

## Tecnologías

- Java 17
- Spring Boot 3.2.x
- Docker
- Kubernetes

---

## Uso Local

### 1. Construir y ejecutar localmente con Maven

```sh
mvn clean spring-boot:repackage -DskipTests
java -jar target/demo-0.0.1-SNAPSHOT.jar
```

### 2. Construir y ejecutar con Docker

```sh
docker build -t springboot-hello-world:latest .
docker run -p 8080:8080 springboot-hello-world:latest
```

Accede a [http://localhost:8080](http://localhost:8080) para probar el endpoint.

---

## Endpoints

- `GET /` → Devuelve mensaje de saludo

---

## Validar Docker en tu equipo local

1. **¿El daemon responde?**
   ```sh
   docker system info >/dev/null 2>&1 && echo "OK: daemon reachable" || echo "ERROR: daemon not reachable"
   ```
   - OK → Docker está levantado.
   - ERROR → Docker no está corriendo o tu CLI no está conectado al daemon.

2. **Verifica versión (debe mostrar Client y Server):**
   ```sh
   docker version
   ```

3. **Contexto activo (elige el correcto):**
   ```sh
   docker context ls
   docker context use default   # o 'desktop-linux' si ese aparece como el de Docker Desktop
   ```

4. **Prueba mínima de ejecución:**
   ```sh
   docker run --rm public.ecr.aws/docker/library/hello-world:latest
   ```
   Si imprime “Hello from Docker!”, todo OK.

---

## Despliegue en AWS EKS

### 1. Variables de entorno

```sh
export AWS_PROFILE=eks-operator
export AWS_REGION=us-east-1
export CLUSTER_NAME=my-eks-cluster
export NAMESPACE=hello
export ECR_REPO=springboot-hello-world

ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
IMAGE_TAG=$(git rev-parse --short HEAD 2>/dev/null || date +%Y%m%d%H%M%S)
IMAGE_URI="$ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/$ECR_REPO:$IMAGE_TAG"
echo "IMAGE_URI=$IMAGE_URI"
```

### 2. Crear repositorio ECR y login

```sh
aws ecr describe-repositories --repository-names "$ECR_REPO" --region "$AWS_REGION" >/dev/null 2>&1 \
 || aws ecr create-repository --repository-name "$ECR_REPO" --image-scanning-configuration scanOnPush=true --region "$AWS_REGION"

aws ecr get-login-password --region "$AWS_REGION" \
 | docker login --username AWS --password-stdin "$ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com"
```

### 3. Build multi-arch y push

```sh
docker buildx create --use >/dev/null 2>&1 || true
docker buildx build --platform linux/amd64 -t "$IMAGE_URI" --push .
```

### 4. Configurar kubeconfig y aplicar manifiestos

```sh
aws eks update-kubeconfig --name "$CLUSTER_NAME" --region "$AWS_REGION" --profile "$AWS_PROFILE"
kubectl get ns "$NAMESPACE" >/dev/null 2>&1 || kubectl create ns "$NAMESPACE"
```

Aplica el siguiente manifiesto (ajustando las variables):

```sh
cat <<'YAML' | IMAGE_URI_REPLACE="$IMAGE_URI" NAMESPACE_REPLACE="$NAMESPACE" envsubst | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello
  namespace: ${NAMESPACE_REPLACE}
spec:
  replicas: 2
  selector:
    matchLabels: { app: hello }
  template:
    metadata:
      labels: { app: hello }
    spec:
      containers:
      - name: app
        image: ${IMAGE_URI_REPLACE}
        ports:
        - containerPort: 8080
        readinessProbe:
          httpGet: { path: "/", port: 8080 }
          initialDelaySeconds: 5
          periodSeconds: 5
        livenessProbe:
          httpGet: { path: "/", port: 8080 }
          initialDelaySeconds: 10
          periodSeconds: 10
        resources:
          requests: { cpu: 100m, memory: 128Mi }
          limits:   { memory: 256Mi }
---
apiVersion: v1
kind: Service
metadata:
  name: hello
  namespace: ${NAMESPACE_REPLACE}
spec:
  type: ClusterIP
  selector:
    app: hello
  ports:
  - name: http
    port: 80
    targetPort: 8080
YAML
```

---

## Verificación y pruebas

```sh
kubectl -n "$NAMESPACE" rollout status deploy/hello
kubectl -n "$NAMESPACE" get pods -o wide
kubectl -n "$NAMESPACE" get svc hello

# Prueba local rápida
kubectl -n "$NAMESPACE" port-forward deploy/hello 8080:8080
curl -s http://localhost:8080/
```

**Exponer públicamente (si tus subnets lo permiten):**

```sh
kubectl -n "$NAMESPACE" patch svc hello -p '{"spec":{"type":"LoadBalancer"}}'
kubectl -n "$NAMESPACE" get svc hello -w    # espera EXTERNAL-IP
```

---

## Actualizar versión de la app

```sh
# Tras construir y pushear una nueva tag a ECR:
kubectl -n "$NAMESPACE" set image deploy/hello app="$IMAGE_URI"
kubectl -n "$NAMESPACE" rollout status deploy/hello
```

---

## Autoescalado (opcional)

```sh
kubectl -n "$NAMESPACE" autoscale deployment hello --min=2 --max=5 --cpu-percent=70
kubectl -n "$NAMESPACE" get hpa
```

---

## Eliminación de recursos

1. **Eliminar LoadBalancers e Ingress:**
   ```sh
   kubectl get svc -A | grep -i LoadBalancer || true
   kubectl get ingress -A || true
   kubectl delete ingress --all -A || true
   kubectl delete svc -A --field-selector spec.type=LoadBalancer || true
   ```

2. **Eliminar namespaces de la app (opcional):**
   ```sh
   kubectl delete ns <your-ns> || true
   ```

3. **Vaciar ECR:**
   ```sh
   aws ecr list-images --repository-name springboot-hello-world --region us-east-1 \
     --query 'imageIds[*]' --output json > /tmp/ecr_imgs.json

   aws ecr batch-delete-image --repository-name springboot-hello-world \
     --image-ids file:///tmp/ecr_imgs.json --region us-east-1 || true

   aws ecr delete-repository \
     --repository-name springboot-hello-world \
     --force --region us-east-1
   ```

4. **Destruir infraestructura (si usas Terraform):**
   ```sh
   terraform destroy -target=module.eks -auto-approve || true
   terraform destroy -target=module.vpc -auto-approve || true
   terraform destroy -auto-approve
   ```

---

## Notas

- El Dockerfile usa build multi-stage para generar un JAR ejecutable y crear una imagen ligera.
- El manifiesto de Kubernetes incluye readiness y liveness probes.
- Puedes adaptar los comandos y manifiestos según tu entorno