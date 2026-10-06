# AWS Learner Lab: parte 2, ALB y DNS

Continua desde [DEPLOY_LEARNER_LAB.md](DEPLOY_LEARNER_LAB.md), con EKS, RDS y la app disponibles. Esta parte crea un ALB manual en EC2; no necesita el AWS Load Balancer Controller ni roles IAM adicionales.

## 1. Exponer el gateway en el nodo

El servicio `gateway` usa el puerto NodePort `30080`. Aplique el manifiesto y compruebe el nodo:

```bash
kubectl apply -f k8s/app.yaml
kubectl get nodes -o wide
kubectl get service gateway
```

En EC2 > Security Groups, identifique el security group compartido del cluster/nodo. Agregue una regla de entrada TCP `30080` cuyo origen sea el security group que creara para el ALB en el paso siguiente. No abra `30080` a Internet.

## 2. Crear el ALB en la consola

Primero, EC2 > Security Groups > Create security group:

- Nombre: `eksapp-alb-sg`.
- VPC: la misma VPC del cluster.
- Entrada: HTTP/TCP/80 desde `0.0.0.0/0`.
- Salida: permitir todo.

Vuelva al security group del cluster/nodo y agregue HTTP personalizado/TCP/`30080` desde `eksapp-alb-sg`.

Luego EC2 > Target Groups > Create target group:

- Target type: Instances.
- Protocol/port: HTTP/`30080`.
- VPC: la del cluster.
- Health check path: `/health`.
- Registre la instancia EC2 del node group. Puede encontrar su ID en EC2 > Instances usando la IP privada que mostro `kubectl get nodes -o wide`.

Por ultimo EC2 > Load Balancers > Create Application Load Balancer:

- Scheme: Internet-facing; IP address type: IPv4.
- VPC: la del cluster; seleccione subredes publicas en dos zonas de disponibilidad.
- Security group: `eksapp-alb-sg`.
- Listener HTTP/80: forward al target group anterior.

Espere que el target figure `healthy`. Copie el DNS del ALB y pruebe `http://<DNS-DEL-ALB>`. El ALB manda el trafico al gateway; Nginx enruta `/` al frontend y `/api` al backend.

## 3. Route 53 (opcional)

Se necesita un dominio propio y una hosted zone publica. Sin dominio, use el DNS del ALB y omita este paso.

En Route 53 > Hosted zones > su zona > Create record, cree un registro `A`, active Alias y seleccione el Application Load Balancer creado. Consulte [Alias de Route 53 hacia un ELB](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/routing-to-elb-load-balancer.html).

## 4. HTTPS (opcional)

En ACM, solicite un certificado publico para el dominio, valide por DNS y espere `Issued`. Debe estar en la misma region que el ALB. En EC2 > Load Balancers, agregue el listener HTTPS/443 que reenvia al mismo target group y asocie el certificado. Agregue HTTPS/443 como entrada de `eksapp-alb-sg` y pruebe `https://su-dominio`.

## 5. Limpiar

Elimine el registro DNS si lo creo, luego el ALB y el target group desde EC2. Quite del security group del cluster la regla TCP/30080 y elimine `eksapp-alb-sg`. Despues siga la limpieza de la primera parte. El ALB factura mientras existe.

Esta version publica frontend y API detras de un ALB. No crea API Gateway ni ALB interno: requieren VPC Link, subredes privadas y permisos IAM que Learner Lab puede denegar. El NodePort fijo simplifica la demo, pero para produccion conviene instalar el controller y automatizar el registro de nodos.
