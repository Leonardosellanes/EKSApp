# AWS Learner Lab: parte 3, API Gateway y ALB interno

Continúa desde [DEPLOY_LEARNER_LAB_PARTE_2.md](DEPLOY_LEARNER_LAB_PARTE_2.md). La parte 2 deja un ALB público para el frontend. Esta parte agrega un HTTP API público, un VPC Link y un ALB interno para la API:

`Navegador -> API Gateway -> VPC Link -> ALB interno -> NodePort 30080 -> gateway -> backend -> RDS`

El frontend seguirá saliendo por el ALB público, pero sus llamadas `/api/notes` irán a API Gateway. Esta guía usa AWS CLI desde CloudShell o una terminal con credenciales vigentes. No guardes credenciales temporales ni claves SSH en el repositorio.

## 1. Variables y subredes

Usa la región y el nombre de cluster que elegiste en la parte 1. El siguiente nombre es el de esta ejecución; reemplázalo si el tuyo es distinto. Encuentra la VPC, el security group del cluster y la instancia EC2 del nodo:

```bash
export AWS_REGION=us-east-1
export CLUSTER_NAME=eksapp-lab-20261005

aws sts get-caller-identity
export VPC_ID=$(aws eks describe-cluster --name "$CLUSTER_NAME" --region "$AWS_REGION" --query 'cluster.resourcesVpcConfig.vpcId' --output text)
export NODE_SG=$(aws eks describe-cluster --name "$CLUSTER_NAME" --region "$AWS_REGION" --query 'cluster.resourcesVpcConfig.clusterSecurityGroupId' --output text)

kubectl get nodes -o wide
export NODE_IP=$(kubectl get nodes -o jsonpath='{.items[0].status.addresses[?(@.type=="InternalIP")].address}')
export INSTANCE_ID=$(aws ec2 describe-instances --region "$AWS_REGION" --filters "Name=private-ip-address,Values=$NODE_IP" "Name=instance-state-name,Values=running" --query 'Reservations[0].Instances[0].InstanceId' --output text)
printf 'VPC=%s\nNode SG=%s\nNode=%s (%s)\n' "$VPC_ID" "$NODE_SG" "$INSTANCE_ID" "$NODE_IP"
```

Lista las subredes de esa VPC y elige dos de zonas de disponibilidad distintas. El ALB interno y el VPC Link deben usar subredes de la VPC del cluster. En producción se prefieren subredes privadas; en esta prueba con la VPC por defecto se usaron dos subredes distintas y el ALB se creó con esquema `internal`.

```bash
aws ec2 describe-subnets --region "$AWS_REGION" --filters "Name=vpc-id,Values=$VPC_ID" --query 'Subnets[*].[SubnetId,AvailabilityZone,CidrBlock]' --output table
export SUBNET_A=subnet-REEMPLAZAR_A
export SUBNET_B=subnet-REEMPLAZAR_B
```

## 2. Security groups

El ALB interno acepta HTTP solo desde las interfaces del VPC Link. El nodo acepta el NodePort `30080` solo desde ese ALB; no abras ese puerto a Internet.

```bash
export ALB_SG=$(aws ec2 create-security-group --region "$AWS_REGION" --group-name eksapp-internal-alb-sg --description 'Internal ALB for API Gateway' --vpc-id "$VPC_ID" --query GroupId --output text)
export VPCLINK_SG=$(aws ec2 create-security-group --region "$AWS_REGION" --group-name eksapp-vpclink-sg --description 'API Gateway VPC Link ENIs' --vpc-id "$VPC_ID" --query GroupId --output text)

aws ec2 authorize-security-group-ingress --region "$AWS_REGION" --group-id "$ALB_SG" --protocol tcp --port 80 --source-group "$VPCLINK_SG"
aws ec2 authorize-security-group-ingress --region "$AWS_REGION" --group-id "$NODE_SG" --protocol tcp --port 30080 --source-group "$ALB_SG"
```

Los security groups nuevos conservan la salida predeterminada permitida. Si se repite el paso, AWS puede responder `InvalidPermission.Duplicate`; no agregues la misma regla otra vez.

## 3. ALB interno y target group

El target type es `instance`: se registra el nodo EC2 en el NodePort fijo del Service `gateway`.

```bash
export LB_ARN=$(aws elbv2 create-load-balancer --region "$AWS_REGION" --name eksapp-internal-lb --scheme internal --type application --subnets "$SUBNET_A" "$SUBNET_B" --security-groups "$ALB_SG" --query 'LoadBalancers[0].LoadBalancerArn' --output text)
aws elbv2 wait load-balancer-available --region "$AWS_REGION" --load-balancer-arns "$LB_ARN"

export TG_ARN=$(aws elbv2 create-target-group --region "$AWS_REGION" --name eksapp-internal-tg --protocol HTTP --port 30080 --vpc-id "$VPC_ID" --target-type instance --health-check-protocol HTTP --health-check-path /health --matcher HttpCode=200-399 --query 'TargetGroups[0].TargetGroupArn' --output text)
aws elbv2 register-targets --region "$AWS_REGION" --target-group-arn "$TG_ARN" --targets "Id=$INSTANCE_ID,Port=30080"

export LISTENER_ARN=$(aws elbv2 create-listener --region "$AWS_REGION" --load-balancer-arn "$LB_ARN" --protocol HTTP --port 80 --default-actions "Type=forward,TargetGroupArn=$TG_ARN" --query 'Listeners[0].ListenerArn' --output text)
aws elbv2 describe-target-health --region "$AWS_REGION" --target-group-arn "$TG_ARN" --output table
```

Espera `healthy`. El DNS de un ALB `internal` solo apunta a direcciones privadas; no se usa directamente desde el navegador fuera de la VPC.

## 4. VPC Link y HTTP API

El VPC Link crea interfaces de red en las dos subredes y conecta API Gateway con el listener HTTP del ALB. Puede tardar varios minutos en quedar `AVAILABLE`.

```bash
export VPC_LINK_ID=$(aws apigatewayv2 create-vpc-link --region "$AWS_REGION" --name eksapp-vpclink --subnet-ids "$SUBNET_A" "$SUBNET_B" --security-group-ids "$VPCLINK_SG" --query VpcLinkId --output text)

aws apigatewayv2 get-vpc-link --region "$AWS_REGION" --vpc-link-id "$VPC_LINK_ID" --query VpcLinkStatus --output text
```

Repite `get-vpc-link` hasta que muestre `AVAILABLE`. Si queda `FAILED`, revisa subredes, security groups y permisos del Learner Lab antes de continuar.

```bash
export API_ID=$(aws apigatewayv2 create-api --region "$AWS_REGION" --name eksapp-api --protocol-type HTTP --query ApiId --output text)

export INTEGRATION_ID=$(aws apigatewayv2 create-integration --region "$AWS_REGION" --api-id "$API_ID" --integration-type HTTP_PROXY --integration-method ANY --integration-uri "$LISTENER_ARN" --connection-type VPC_LINK --connection-id "$VPC_LINK_ID" --payload-format-version 1.0 --query IntegrationId --output text)

aws apigatewayv2 create-route --region "$AWS_REGION" --api-id "$API_ID" --route-key '$default' --target "integrations/$INTEGRATION_ID"
aws apigatewayv2 create-stage --region "$AWS_REGION" --api-id "$API_ID" --stage-name '$default' --auto-deploy

export API_URL="https://${API_ID}.execute-api.${AWS_REGION}.amazonaws.com"
printf 'API_URL=%s\n' "$API_URL"
```

## 5. CORS y frontend

Como la página está en el ALB público y la API en otro dominio, el navegador exige CORS. Permite solo el origen exacto del ALB público, no `*`:

```bash
export PUBLIC_ALB_DNS=DNS_COPIADO_DE_PARTE_2
export FRONTEND_ORIGIN="http://$PUBLIC_ALB_DNS"

aws apigatewayv2 update-api --region "$AWS_REGION" --api-id "$API_ID" --cors-configuration "{\"AllowOrigins\":[\"$FRONTEND_ORIGIN\"],\"AllowMethods\":[\"GET\",\"POST\",\"OPTIONS\"],\"AllowHeaders\":[\"content-type\",\"accept\"],\"MaxAge\":300}"
```

El frontend concatena `/notes` a `NUXT_PUBLIC_API_BASE`, por eso la base termina en `/api`. El manifiesto inicial usa `/api` para el ALB único; al completar esta parte, cambia la variable del Deployment:

```bash
kubectl set env deployment/frontend "NUXT_PUBLIC_API_BASE=${API_URL}/api"
kubectl rollout status deployment/frontend
```

Después de volver a aplicar `k8s/app.yaml`, ejecuta otra vez `kubectl set env`, porque el manifiesto base restaura `/api`.

Prueba la API y su preflight CORS:

```bash
curl -i "$API_URL/health"
curl -i "$API_URL/api/notes" -H "Origin: $FRONTEND_ORIGIN"
curl -i -X OPTIONS "$API_URL/api/notes" \
  -H "Origin: $FRONTEND_ORIGIN" \
  -H 'Access-Control-Request-Method: POST' \
  -H 'Access-Control-Request-Headers: content-type'
```

`/health` y `/api/notes` deben responder `200`; el preflight debe responder `204` e incluir `Access-Control-Allow-Origin` con el origen del ALB. Recarga la aplicación y comprueba que aparecen las notas.

## 6. Qué significa "sin autorización"

La ruta HTTP API se creó con `AuthorizationType: NONE`: cualquier persona que conozca la URL puede llamar a `GET /api/notes` y `POST /api/notes`; no se pide usuario, contraseña ni token. API Gateway aquí enruta el tráfico, pero no autentica usuarios. CORS limita qué páginas pueden leer la respuesta desde un navegador; **no** bloquea llamadas desde `curl` u otros clientes y no reemplaza autenticación. Es suficiente para una demo de notas no sensibles, no para datos privados.

Route 53, dominio propio, HTTPS para el ALB público y autenticación no están incluidos. API Gateway entrega una URL HTTPS propia; para usar `api.tudominio`, se necesita un dominio y un certificado regional de ACM. Para proteger la API en un ejercicio posterior, agrega un authorizer y prueba acceso con y sin token.

## 7. Valores de cada laboratorio

Los comandos generan IDs y DNS propios en cada cuenta. Usa los valores que imprimen y no reutilices URLs, IDs, endpoints de RDS ni credenciales de otra ejecución. El API Gateway resultante es público; esta práctica no configura autenticación.

## 8. Costos y limpieza

Un segundo ALB aumenta el consumo mientras exista. Elimina los recursos cuando termine la práctica. En el mismo shell donde guardaste las variables:

```bash
aws apigatewayv2 delete-api --region "$AWS_REGION" --api-id "$API_ID"
aws apigatewayv2 delete-vpc-link --region "$AWS_REGION" --vpc-link-id "$VPC_LINK_ID"
aws elbv2 delete-load-balancer --region "$AWS_REGION" --load-balancer-arn "$LB_ARN"
aws elbv2 wait load-balancers-deleted --region "$AWS_REGION" --load-balancer-arns "$LB_ARN"
aws elbv2 delete-target-group --region "$AWS_REGION" --target-group-arn "$TG_ARN"
aws ec2 revoke-security-group-ingress --region "$AWS_REGION" --group-id "$NODE_SG" --protocol tcp --port 30080 --source-group "$ALB_SG"
aws ec2 delete-security-group --region "$AWS_REGION" --group-id "$ALB_SG"
aws ec2 delete-security-group --region "$AWS_REGION" --group-id "$VPCLINK_SG"
kubectl set env deployment/frontend NUXT_PUBLIC_API_BASE=/api
```

La eliminación del VPC Link puede tardar; si AWS todavía informa que el security group está en uso, espera unos minutos y vuelve a borrarlo. La limpieza de esta parte no borra el ALB público, EKS ni RDS.
