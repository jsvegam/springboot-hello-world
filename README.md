# springboot-hello-world

This project is a simple Spring Boot microservice designed for deployment in an AWS EKS (Elastic Kubernetes Service) environment.  
It exposes a REST API endpoint that returns a greeting message.

## Features

- REST API endpoint at `/` that returns "Hello from Spring Boot!"
- Dockerfile for containerization
- Kubernetes manifests for deployment and service configuration

## Technologies

- Java 17
- Spring Boot 3.2.x
- Docker
- Kubernetes

## Usage

1. Build the project with Maven.
2. Build and push the Docker image to your container registry.
3. Deploy to Kubernetes using the manifests in the `k8s/` directory.

## Endpoints

- `GET /` → Returns greeting message




## Para desplegar esta pieza en el Cluster EKS
## Validar el funcionamiento de docker en el equipo local

1) ¿El daemon responde?
docker system info >/dev/null 2>&1 && echo "OK: daemon reachable" || echo "ERROR: daemon not reachable"


OK → Docker está levantado.

ERROR → no está corriendo o tu CLI no está conectado al daemon.

2) Versión (debe mostrar Client y Server)
docker version


Si solo ves “Client” o te dice Cannot connect to the Docker daemon… → no está corriendo.

3) Contexto activo (elige el correcto)
docker context ls
docker context use default   # o 'desktop-linux' si ese aparece como el de Docker Desktop

4) Prueba mínima de ejecución (sin depender de Docker Hub)
docker run --rm public.ecr.aws/docker/library/hello-world:latest

Si imprime “Hello from Docker!”, todo OK.

Si se queda colgado aquí, suele ser red/proxy bloqueando el registry.

## Para desplegar en el Cluster EKS #### (Obligatorio)

1) Variables
export AWS_PROFILE=eks-operator
export AWS_REGION=us-east-1
export CLUSTER_NAME=my-eks-cluster
export NAMESPACE=hello
export ECR_REPO=springboot-hello-world

ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
IMAGE_TAG=$(git rev-parse --short HEAD 2>/dev/null || date +%Y%m%d%H%M%S)
IMAGE_URI="$ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/$ECR_REPO:$IMAGE_TAG"
echo "IMAGE_URI=$IMAGE_URI"


## ECR: crear repo (si no existe) + login #### (Obligatorio)
aws ecr describe-repositories --repository-names "$ECR_REPO" --region "$AWS_REGION" >/dev/null 2>&1 \
 || aws ecr create-repository --repository-name "$ECR_REPO" --image-scanning-configuration scanOnPush=true --region "$AWS_REGION"

aws ecr get-login-password --region "$AWS_REGION" \
 | docker login --username AWS --password-stdin "$ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com"


# 3) Build multi-arch (amd64) y push #### (Obligatorio)
Opción A (recomendada): buildx
docker buildx create --use >/dev/null 2>&1 || true
docker buildx build --platform linux/amd64 -t "$IMAGE_URI" --push .


# 4) Kubeconfig y manifiesto listo para aplicar #### (Obligatorio)
aws eks update-kubeconfig --name "$CLUSTER_NAME" --region "$AWS_REGION" --profile "$AWS_PROFILE"

kubectl get ns "$NAMESPACE" >/dev/null 2>&1 || kubectl create ns "$NAMESPACE"

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


## Verificación (Opcional)
5) Verificación y prueba
kubectl -n "$NAMESPACE" rollout status deploy/hello
kubectl -n "$NAMESPACE" get pods -o wide
kubectl -n "$NAMESPACE" get svc hello

Probar local (rápido)
kubectl -n "$NAMESPACE" port-forward deploy/hello 8080:8080
# en otra terminal:
curl -s http://localhost:8080/

Exponer público (si tus subnets permiten LB)
kubectl -n "$NAMESPACE" patch svc hello -p '{"spec":{"type":"LoadBalancer"}}'
kubectl -n "$NAMESPACE" get svc hello -w    # espera EXTERNAL-IP

6) Actualizar versión (nuevo push)
# tras construir y pushear una nueva tag a ECR:
kubectl -n "$NAMESPACE" set image deploy/hello app="$IMAGE_URI"
kubectl -n "$NAMESPACE" rollout status deploy/hello

7) (Opcional) Autoescalado rápido
kubectl -n "$NAMESPACE" autoscale deployment hello --min=2 --max=5 --cpu-percent=70
kubectl -n "$NAMESPACE" get hpa



## To destroy

# see what would block VPC: LBs created by Services/Ingress
kubectl get svc -A | grep -i LoadBalancer || true
kubectl get ingress -A || true


# 0) Ensure kube context is valid (if cluster still exists)
aws eks update-kubeconfig --region us-east-1 --name <cluster> --profile <profile> || true

# 1) Remove LB creators in K8s
kubectl delete ingress --all -A || true
kubectl delete svc -A --field-selector spec.type=LoadBalancer || true

# 2) (Optional) Delete app namespaces
kubectl delete ns <your-ns> || true

# 3) Empty ECR if you didn’t set force_delete
aws ecr list-images --repository-name deepseek-app --region us-east-1 \
  --query 'imageIds[*]' --output json > /tmp/ecr_imgs.json

aws ecr batch-delete-image --repository-name deepseek-app \
  --image-ids file:///tmp/ecr_imgs.json --region us-east-1 || true


aws ecr list-images --repository-name springboot-hello-world --region us-east-1 \
  --query 'imageIds[*]' --output json > /tmp/ecr_imgs.json

aws ecr batch-delete-image --repository-name springboot-hello-world \
  --image-ids file:///tmp/ecr_imgs.json --region us-east-1 || true

 # Opción A) Borrar todo el repo de una (incluye todas las imágenes)

aws ecr delete-repository \
  --repository-name springboot-hello-world \
  --force --region us-east-1




# 4) Destroy in order
terraform destroy -target=module.eks -auto-approve || true
terraform destroy -target=module.vpc -auto-approve || true
terraform destroy -auto-approve


