# Recuperación de EKSApp en AWS Learner Lab

Esta guía fue validada el 9 de octubre de 2026 contra el despliegue real de esta cuenta. No guarda ni requiere pegar credenciales en archivos del repositorio.

## Causa que se corrigió

El node group original `eksapp-workers-20261005` usaba Amazon Linux 2023 para Kubernetes 1.37. Su proceso de arranque (`nodeadm`) necesita llamar a `ec2:DescribeInstances`, pero el SCP del Learner Lab lo bloquea explícitamente. Por eso se creaba una instancia EC2 aparentemente sana, pero nunca se registraba como nodo de Kubernetes; todos los pods quedaban `Pending`.

Se aplicaron estos cambios permanentes:

- Se creó el node group `eksapp-workers-br`, con `BOTTLEROCKET_x86_64`, tamaño mínimo, deseado y máximo de 1.
- Se asociaron los target groups público e interno de los ALB al Auto Scaling Group de ese node group. El nodo que reemplace AWS se registra y desregistra automáticamente; ya no hay que ejecutar `register-targets` manualmente.
- Se inició la eliminación del node group roto para evitar una instancia adicional inútil.

## Pasos al comenzar una nueva sesión

Abrí **CloudShell** dentro del Learner Lab y comprobá que está usando las credenciales de la sesión actual:

```bash
export AWS_REGION=us-east-1
export CLUSTER_NAME=eksapp-lab-20261005
export NODEGROUP=eksapp-workers-br

aws sts get-caller-identity
```

### 1. Autorizar la IP de CloudShell para administrar EKS

El endpoint de Kubernetes está restringido por IP. Cada sesión de CloudShell puede salir con una IP distinta. Este comando preserva acceso privado para los nodos y actualiza la allowlist pública a la IP de CloudShell actual:

```bash
CLOUDSHELL_IP=$(curl -s https://checkip.amazonaws.com | tr -d '\n')

aws eks update-cluster-config \
  --region "$AWS_REGION" \
  --name "$CLUSTER_NAME" \
  --resources-vpc-config "endpointPublicAccess=true,endpointPrivateAccess=true,publicAccessCidrs=$CLOUDSHELL_IP/32"

aws eks wait cluster-active \
  --region "$AWS_REGION" \
  --name "$CLUSTER_NAME"
```

### 2. Configurar kubectl y comprobar el nodo

```bash
aws eks update-kubeconfig \
  --region "$AWS_REGION" \
  --name "$CLUSTER_NAME"

kubectl get nodes -o wide
kubectl get pods -o wide
```

El resultado esperado es un nodo `Ready` con `Bottlerocket OS` y tres pods `1/1 Running`: `backend`, `frontend` y `gateway`.

Si el nodo todavía se está creando, comprobá el estado y esperá a que sea `ACTIVE`:

```bash
aws eks describe-nodegroup \
  --region "$AWS_REGION" \
  --cluster-name "$CLUSTER_NAME" \
  --nodegroup-name "$NODEGROUP" \
  --query 'nodegroup.[status,health.issues,scalingConfig]' \
  --output table
```

El node group ya está configurado con `minSize=1`, `desiredSize=1` y `maxSize=1`. No hay que recrearlo ni reiniciar deployments: Kubernetes vuelve a programar los pods cuando el nodo aparece.

### 3. Verificar que los ALB registraron el nodo automáticamente

Obtené el Auto Scaling Group y sus target groups:

```bash
ASG=$(aws eks describe-nodegroup \
  --region "$AWS_REGION" \
  --cluster-name "$CLUSTER_NAME" \
  --nodegroup-name "$NODEGROUP" \
  --query 'nodegroup.resources.autoScalingGroups[0].name' \
  --output text)

aws autoscaling describe-auto-scaling-groups \
  --region "$AWS_REGION" \
  --auto-scaling-group-names "$ASG" \
  --query 'AutoScalingGroups[0].[AutoScalingGroupName,TargetGroupARNs]' \
  --output table
```

La salida debe mostrar dos target groups. Si el nodo y los pods ya están listos, los health checks pasan a `healthy` en poco tiempo. Consultalos así:

```bash
PUBLIC_TG=$(aws elbv2 describe-target-groups \
  --region "$AWS_REGION" \
  --names eksapp-tg-20261005 \
  --query 'TargetGroups[0].TargetGroupArn' \
  --output text)

INTERNAL_TG=$(aws elbv2 describe-target-groups \
  --region "$AWS_REGION" \
  --names eksapp-internal-tg-20261006 \
  --query 'TargetGroups[0].TargetGroupArn' \
  --output text)

aws elbv2 describe-target-health --region "$AWS_REGION" --target-group-arn "$PUBLIC_TG"
aws elbv2 describe-target-health --region "$AWS_REGION" --target-group-arn "$INTERNAL_TG"
```

No ejecutes `register-targets`: el Auto Scaling Group se ocupa de ello.

### 4. Prueba final

```bash
curl -i http://eksapp-lb-20261005-1050734818.us-east-1.elb.amazonaws.com/health
curl -i https://uminyfbltl.execute-api.us-east-1.amazonaws.com/health
curl -i https://uminyfbltl.execute-api.us-east-1.amazonaws.com/api/notes
```

Los tres comandos deben devolver `HTTP/1.1 200` cuando la app esté recuperada.

## Si vuelve a fallar

Primero ejecutá los pasos 1 y 2. Si no aparece un nodo `Ready`, revisá el diagnóstico de `describe-nodegroup` y los eventos de Kubernetes:

```bash
kubectl get events --sort-by=.lastTimestamp | tail -n 30
```

No vuelvas a crear un node group Amazon Linux 2023 con este Learner Lab: el SCP que bloquea `ec2:DescribeInstances` volvería a impedir que el nodo se una al cluster.
