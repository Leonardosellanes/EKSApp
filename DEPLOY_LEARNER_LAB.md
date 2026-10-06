# AWS Learner Lab: despliegue minimo

Objetivo: crear PostgreSQL en RDS, un cluster EKS con un nodo y desplegar la app con un pod por servicio. No se crean ALB, API Gateway ni Route 53. Para probarla, se usa `kubectl port-forward`.

Use la misma region y VPC para RDS y EKS. Esta guia supone que puede usar la VPC por defecto. AWS CLI y `kubectl` pueden ejecutarse en CloudShell; construya las imagenes desde una terminal local con Docker y credenciales temporales del Learner Lab.

## 1. Comprobar permisos

Antes de crear recursos, revise en la consola de EKS que existan dos roles IAM distintos: uno para el cluster y otro para el node group. El de nodos necesita permitir EC2 y descargar imagenes de ECR. Si no aparecen o recibe `AccessDenied`/`iam:PassRole`, detengase y consulte al instructor; no cree recursos que luego no pueda usar.

En CloudShell o una terminal con las credenciales temporales del Learner Lab:

```bash
aws sts get-caller-identity
aws configure get region
```

## 2. Crear PostgreSQL en RDS

En AWS Console > RDS > Create database:

- Engine: PostgreSQL.
- Template: Dev/Test o la opcion de menor costo.
- Instancia: `db.t3.micro`, Single-AZ, 20 GiB, sin autoscaling.
- Nombre inicial de base: `eks_notes`; usuario master: `eks_user`.
- VPC: la VPC por defecto, que usara tambien EKS.
- Public access: No.
- Security group: cree `eksapp-db`, sin reglas de entrada por ahora.

Guarde endpoint, usuario, password y security group. Espere estado `Available`.

## 3. Crear EKS con un nodo

En AWS Console > EKS > Create cluster:

- Seleccione el rol de cluster comprobado en el paso 1.
- Use la VPC por defecto y dos subredes en zonas de disponibilidad distintas. En `us-east-1`, si EKS informa que una zona no esta disponible, seleccione otras dos; en esta cuenta `us-east-1e` no fue aceptada.
- Habilite los endpoints publico y privado. Limite el endpoint publico a su IP actual si la consola lo permite; el privado permite que los nodos lleguen al plano de control desde la VPC.
- Cree un managed node group con el otro rol existente.
- Use una instancia `t3.medium`, desired/min/max en `1`.

Cuando el node group este `Active`, abra su security group en EC2. En `eksapp-db`, agregue una regla PostgreSQL/TCP/5432 cuyo origen sea ese security group del nodo. RDS seguira sin acceso publico.

## 4. Crear ECR y subir las imagenes

Cree en ECR Console dos repositorios: `eksapp-frontend` y `eksapp-backend`. Desde una terminal con Docker y credenciales vigentes:

```bash
AWS_REGION=us-east-1
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
REGISTRY="$ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com"
FRONTEND_IMAGE="$REGISTRY/eksapp-frontend:latest"
BACKEND_IMAGE="$REGISTRY/eksapp-backend:latest"

aws ecr get-login-password --region "$AWS_REGION" | docker login --username AWS --password-stdin "$REGISTRY"
docker build -f frontend/dockerfile -t "$FRONTEND_IMAGE" ./frontend
docker build -f backend/dockerfile -t "$BACKEND_IMAGE" ./backend
docker push "$FRONTEND_IMAGE"
docker push "$BACKEND_IMAGE"
```

Use la region de AWS que selecciono. Copie las URI completas de las imagenes para el paso siguiente.

## 5. Aplicar Kubernetes

Desde la raiz del repositorio, en `k8s/app.yaml`, reemplace `REPLACE_WITH_RDS_ENDPOINT` y las dos URI `YOUR_*_IMAGE_URI`. El endpoint RDS va sin `:5432`.

Conecte `kubectl` al cluster y cree el secreto. Use el password de RDS; si contiene comilla simple, cambie el tipo de comillas en el comando:

```bash
aws eks update-kubeconfig --region "$AWS_REGION" --name NOMBRE_DEL_CLUSTER
APP_KEY=$(docker run --rm "$BACKEND_IMAGE" php artisan key:generate --show)
kubectl create secret generic eksapp-runtime \
  --from-literal=APP_KEY="$APP_KEY" \
  --from-literal=DB_PASSWORD='PASSWORD_DE_RDS'
kubectl create configmap gateway-config --from-file=nginx.conf=gateway/nginx.conf
kubectl apply -f k8s/app.yaml
kubectl wait --for=condition=complete job/eksapp-migrate --timeout=5m
kubectl get pods,services
```

El Job ejecuta las migraciones. Al terminar, los pods deberian estar `Running` y `Ready`.

## 6. Abrir y probar

```bash
kubectl port-forward service/gateway 8080:8080
```

Deje ese comando activo y abra `http://localhost:8080`. El gateway envia `/` al frontend y `/api` al backend.

Opcional: Adminer temporal para inspeccionar RDS. En otra terminal, reemplace el endpoint:

```bash
kubectl run adminer --image=adminer:4 --env="ADMINER_DEFAULT_SERVER=REPLACE_WITH_RDS_ENDPOINT" --port=8080
kubectl port-forward pod/adminer 8081:8080
```

Abra `http://localhost:8081` y use PostgreSQL, `eks_notes` y las credenciales de RDS. Al terminar: `kubectl delete pod adminer`.

## 7. Limpiar

```bash
kubectl delete -f k8s/app.yaml
```

En la consola, elimine node group, cluster EKS y base RDS. Cuando RDS termine de borrarse, elimine su security group y DB subnet group. Elimine tambien los dos repositorios ECR. Borrar RDS elimina sus datos.

Esta primera parte usa una sola cuenta/VPC y no crea balanceadores. Para publicar la app con un ALB y, opcionalmente, Route 53 y HTTPS, continue en [DEPLOY_LEARNER_LAB_PARTE_2.md](DEPLOY_LEARNER_LAB_PARTE_2.md).
