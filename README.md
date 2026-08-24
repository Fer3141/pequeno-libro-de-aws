# Pequeño libro de AWS

Guía rápida de Amazon Web Services (AWS): conceptos clave, comandos de la AWS CLI más usados y cómo obtener las certificaciones oficiales. Todo explicado en español.

## Índice

- [¿Qué es AWS?](#qué-es-aws)
- [Configuración inicial](#configuración-inicial)
- [Regiones, zonas de disponibilidad y Free Tier](#regiones-zonas-de-disponibilidad-y-free-tier)
- [IAM (gestión de identidades y accesos)](#iam-gestión-de-identidades-y-accesos)
- [S3 (almacenamiento de objetos)](#s3-almacenamiento-de-objetos)
- [EC2 (cómputo)](#ec2-cómputo)
- [VPC (redes)](#vpc-redes)
- [Lambda (serverless)](#lambda-serverless)
- [RDS (bases de datos)](#rds-bases-de-datos)
- [CloudFormation (infraestructura como código)](#cloudformation-infraestructura-como-código)
- [CloudWatch (monitoreo y logs)](#cloudwatch-monitoreo-y-logs)
- [Costos y facturación](#costos-y-facturación)
- [Buenas prácticas de seguridad](#buenas-prácticas-de-seguridad)
- [Certificaciones de AWS](#certificaciones-de-aws)

---

## ¿Qué es AWS?

Amazon Web Services es la plataforma de servicios en la nube de Amazon: cómputo, almacenamiento, bases de datos, redes, machine learning, etc., que se contratan bajo demanda y se pagan solo por lo que se usa (modelo *pay-as-you-go*). Se puede administrar de tres formas: desde la **consola web**, desde la **AWS CLI** (línea de comandos) o mediante **SDKs**/infraestructura como código.

---

## Configuración inicial

### Instalar la AWS CLI

La AWS CLI (versión 2, la actual) se descarga desde la página oficial: https://aws.amazon.com/cli/ — ahí mismo están los instaladores para cada sistema operativo. También se puede instalar así:

**Windows**: descargar y ejecutar el instalador `.msi` de 64 bits: https://awscli.amazonaws.com/AWSCLIV2.msi

**macOS**:
```bash
curl "https://awscli.amazonaws.com/AWSCLIV2.pkg" -o "AWSCLIV2.pkg"
sudo installer -pkg AWSCLIV2.pkg -target /
```

**Linux**:
```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```

```bash
aws --version
```
Comprueba que la instalación funcionó y muestra la versión instalada (debería ser `aws-cli/2.x`).

### Crear las credenciales

`aws configure` pide un `Access Key ID` y un `Secret Access Key`. Estos se generan así:

1. Entrar a la consola de AWS → **IAM** → **Users** → seleccionar (o crear) el usuario.
2. Pestaña **Security credentials** → **Create access key**.
3. Elegir el caso de uso *Command Line Interface (CLI)* y confirmar.
4. Copiar el `Access Key ID` y el `Secret Access Key` que se muestran — el secret **solo se ve una vez**, hay que guardarlo en ese momento (o descargar el `.csv`).

> No uses las credenciales del usuario **root** para esto. Crea un usuario IAM (ver sección [IAM](#iam-gestión-de-identidades-y-accesos)) con los permisos que necesites y genera el access key sobre ese usuario.

```bash
aws configure
```
Pide de forma interactiva el `Access Key ID`, `Secret Access Key`, la región por defecto (ej. `us-east-1`) y el formato de salida (`json`, `text`, `table`). Guarda las credenciales en `~/.aws/credentials` y la configuración en `~/.aws/config`.

```bash
aws configure --profile <nombre>
```
Crea un perfil independiente (útil para manejar varias cuentas). Luego se usa añadiendo `--profile <nombre>` a cualquier comando, o exportando `AWS_PROFILE=<nombre>`.

```bash
aws sts get-caller-identity
```
Muestra qué usuario/rol está autenticado en este momento (cuenta, ARN, user id). Es el primer comando para comprobar que las credenciales funcionan.

> **Recomendación**: nunca uses el usuario **root** de la cuenta para el día a día. Crea un usuario IAM con permisos administrativos y activa MFA tanto en root como en ese usuario.

---

## Regiones, zonas de disponibilidad y Free Tier

AWS opera en **regiones** (áreas geográficas, ej. `us-east-1` en Virginia, `eu-west-1` en Irlanda) y cada región se divide en varias **zonas de disponibilidad** (AZ), que son centros de datos físicamente separados pero interconectados. Distribuir recursos entre AZs da tolerancia a fallos.

```bash
aws ec2 describe-regions --output table
```
Lista todas las regiones disponibles para la cuenta.

```bash
aws ec2 describe-availability-zones --region us-east-1
```
Lista las zonas de disponibilidad de una región concreta.

El **Free Tier** (capa gratuita) permite usar ciertos servicios sin costo durante los primeros 12 meses de la cuenta (ej. 750 horas/mes de EC2 `t2.micro`), más un puñado de servicios que son gratuitos "siempre" dentro de límites bajos (ej. Lambda, DynamoDB). Conviene revisar los límites exactos en la consola de facturación para no llevarse sorpresas.

---

## IAM (gestión de identidades y accesos)

IAM controla **quién** puede hacer **qué** dentro de la cuenta.

- **Usuario**: identidad para una persona o servicio, con credenciales propias.
- **Grupo**: conjunto de usuarios al que se le asignan permisos en bloque.
- **Rol**: identidad temporal sin credenciales fijas, pensada para que la asuman servicios de AWS (una EC2, una Lambda) u otras cuentas.
- **Política**: documento JSON que define permisos (qué acciones, sobre qué recursos, permitir o denegar).

```bash
aws iam create-user --user-name <nombre>
```
Crea un nuevo usuario IAM.

```bash
aws iam attach-user-policy --user-name <nombre> --policy-arn <arn-de-la-política>
```
Asocia una política (por ejemplo una gestionada por AWS como `arn:aws:iam::aws:policy/ReadOnlyAccess`) a un usuario.

```bash
aws iam list-users
aws iam list-roles
aws iam list-policies --scope Local
```
Listan usuarios, roles y políticas creadas por la cuenta (`--scope Local` filtra las políticas propias, excluyendo las administradas por AWS).

```bash
aws iam create-role --role-name <nombre> --assume-role-policy-document file://trust-policy.json
```
Crea un rol a partir de una *trust policy* (define quién puede asumir el rol: un servicio, otra cuenta, etc.).

**Buena práctica clave**: aplicar el **principio de mínimo privilegio** — dar solo los permisos estrictamente necesarios — y preferir roles temporales sobre credenciales de usuario de larga duración siempre que sea posible.

---

## S3 (almacenamiento de objetos)

S3 (Simple Storage Service) guarda archivos ("objetos") dentro de contenedores llamados **buckets**. No es un sistema de archivos tradicional: es almacenamiento clave-valor con altísima durabilidad (99.999999999%, "11 nueves").

```bash
aws s3 mb s3://<nombre-bucket>
```
Crea un bucket nuevo (`mb` = *make bucket*). El nombre debe ser único a nivel global.

```bash
aws s3 ls
aws s3 ls s3://<nombre-bucket>
```
Lista los buckets de la cuenta, o los objetos dentro de un bucket.

```bash
aws s3 cp archivo.txt s3://<nombre-bucket>/
aws s3 cp s3://<nombre-bucket>/archivo.txt ./
```
Sube o descarga un archivo. `cp` funciona en ambas direcciones según el orden origen/destino.

```bash
aws s3 sync ./carpeta-local s3://<nombre-bucket>/carpeta
```
Sincroniza una carpeta local con un bucket (o al revés), subiendo solo los archivos nuevos o modificados.

```bash
aws s3 rm s3://<nombre-bucket>/archivo.txt
aws s3 rb s3://<nombre-bucket> --force
```
Borra un objeto, o elimina un bucket completo (`rb` = *remove bucket*; `--force` borra también su contenido). **Cuidado**: es irreversible si el bucket no tiene versionado activado.

---

## EC2 (cómputo)

EC2 (Elastic Compute Cloud) ofrece servidores virtuales ("instancias") bajo demanda.

```bash
aws ec2 describe-instances
```
Lista las instancias existentes con su estado, tipo, IPs, etc.

```bash
aws ec2 run-instances --image-id <ami-id> --instance-type t2.micro --key-name <mi-clave> --security-group-ids <sg-id> --subnet-id <subnet-id>
```
Lanza una nueva instancia a partir de una AMI (imagen base), especificando tipo de instancia, par de claves SSH, grupo de seguridad y subred.

```bash
aws ec2 start-instances --instance-ids <id>
aws ec2 stop-instances --instance-ids <id>
aws ec2 terminate-instances --instance-ids <id>
```
Inicia, detiene (apaga, pero conserva el disco) o termina (elimina definitivamente) una instancia. Mientras está detenida solo se cobra el almacenamiento EBS asociado, no el cómputo.

```bash
aws ec2 describe-security-groups
```
Lista los grupos de seguridad (firewalls virtuales a nivel de instancia) y sus reglas de entrada/salida.

---

## VPC (redes)

Una VPC (Virtual Private Cloud) es una red privada y aislada dentro de AWS donde viven los recursos (instancias EC2, bases de datos, etc.). Se divide en **subredes** (públicas o privadas), tiene tablas de rutas, un *internet gateway* para salir a internet y grupos de seguridad/NACLs para controlar el tráfico.

```bash
aws ec2 create-vpc --cidr-block 10.0.0.0/16
```
Crea una VPC con el rango de direcciones IP indicado (CIDR).

```bash
aws ec2 create-subnet --vpc-id <vpc-id> --cidr-block 10.0.1.0/24 --availability-zone us-east-1a
```
Crea una subred dentro de una VPC, en una zona de disponibilidad concreta.

```bash
aws ec2 describe-vpcs
aws ec2 describe-subnets
```
Listan las VPCs y subredes existentes.

> Una subred es "pública" cuando su tabla de rutas apunta al *internet gateway* de la VPC; si no tiene esa ruta, es "privada" y solo se comunica dentro de la red (o a través de un NAT gateway para salir a internet sin ser accesible desde fuera).

---

## Lambda (serverless)

Lambda ejecuta código sin necesidad de gestionar servidores: se paga solo por el tiempo de ejecución. Cada función se dispara por un **evento** (una petición HTTP vía API Gateway, un archivo subido a S3, un mensaje en una cola SQS, un horario, etc.).

```bash
aws lambda create-function --function-name <nombre> --runtime python3.12 --role <arn-del-rol> --handler <archivo.funcion> --zip-file fileb://paquete.zip
```
Crea una función Lambda a partir de un paquete de código comprimido, indicando el runtime y el rol IAM que la función asumirá para acceder a otros servicios.

```bash
aws lambda invoke --function-name <nombre> --payload '{"clave":"valor"}' respuesta.json
```
Invoca la función manualmente con un payload de prueba y guarda la respuesta en un archivo local.

```bash
aws lambda update-function-code --function-name <nombre> --zip-file fileb://paquete.zip
```
Actualiza el código de una función ya existente sin recrearla.

---

## RDS (bases de datos)

RDS (Relational Database Service) ofrece bases de datos relacionales gestionadas (MySQL, PostgreSQL, MariaDB, SQL Server, Oracle) sin tener que administrar el servidor: AWS se encarga de parches, backups automáticos y replicación.

```bash
aws rds create-db-instance --db-instance-identifier <nombre> --db-instance-class db.t3.micro --engine postgres --master-username <usuario> --master-user-password <password> --allocated-storage 20
```
Crea una instancia de base de datos gestionada.

```bash
aws rds describe-db-instances
```
Lista las instancias de base de datos y su estado (endpoint, puerto, etc.).

```bash
aws rds create-db-snapshot --db-instance-identifier <nombre> --db-snapshot-identifier <snapshot-nombre>
```
Crea una copia de seguridad manual (snapshot) de la base de datos.

---

## CloudFormation (infraestructura como código)

CloudFormation permite definir toda la infraestructura (VPCs, instancias, buckets, roles...) en archivos de texto (YAML o JSON) llamados **plantillas**, y desplegarla de forma reproducible como una unidad llamada **stack**.

```bash
aws cloudformation deploy --template-file plantilla.yaml --stack-name <nombre-stack> --capabilities CAPABILITY_IAM
```
Crea o actualiza un stack a partir de una plantilla. `--capabilities CAPABILITY_IAM` es necesario si la plantilla crea roles o políticas IAM.

```bash
aws cloudformation describe-stacks --stack-name <nombre-stack>
```
Muestra el estado y los outputs del stack.

```bash
aws cloudformation delete-stack --stack-name <nombre-stack>
```
Elimina el stack completo, junto con todos los recursos que creó. **Cuidado**: es la forma más común de "limpiar" un entorno de prueba, pero también de borrar algo por error si se apunta al stack equivocado.

---

## CloudWatch (monitoreo y logs)

CloudWatch centraliza métricas (CPU, memoria, latencia, invocaciones...), logs y alarmas de los servicios de AWS.

```bash
aws logs describe-log-groups
```
Lista los grupos de logs existentes (por ejemplo, uno por cada función Lambda).

```bash
aws logs tail /aws/lambda/<nombre-funcion> --follow
```
Muestra los logs de un grupo en tiempo real, similar a `tail -f`.

```bash
aws cloudwatch put-metric-alarm --alarm-name <nombre> --metric-name CPUUtilization --namespace AWS/EC2 --statistic Average --period 300 --threshold 80 --comparison-operator GreaterThanThreshold --evaluation-periods 2 --alarm-actions <arn-sns>
```
Crea una alarma que se dispara (y notifica, por ejemplo, a un tema SNS) cuando el uso de CPU de una instancia supera el 80% durante dos periodos consecutivos de 5 minutos.

---

## Costos y facturación

```bash
aws ce get-cost-and-usage --time-period Start=2024-01-01,End=2024-02-01 --granularity MONTHLY --metrics "UnblendedCost"
```
Consulta el costo total facturado en un rango de fechas usando Cost Explorer (`ce`).

- **AWS Budgets**: permite crear presupuestos y recibir alertas por correo/SNS cuando el gasto (real o proyectado) supera un umbral. Se configura desde la consola de Billing.
- **Cost Explorer**: visualiza el gasto histórico desglosado por servicio, región o etiqueta (*tag*).
- **AWS Free Tier**: la propia consola de facturación muestra el consumo acumulado frente al límite gratuito de cada servicio.

> Recomendación básica para no llevarse sustos: crear un budget de alerta desde el primer día de la cuenta, aunque sea de $1 o $5.

---

## Buenas prácticas de seguridad

- Activar **MFA** (autenticación multifactor) en el usuario root y en los usuarios IAM con permisos elevados.
- No usar nunca el usuario **root** para tareas del día a día; crear usuarios IAM con el mínimo privilegio necesario.
- Usar **roles** en vez de credenciales de larga duración cuando un servicio de AWS necesita acceder a otro (ej. una EC2 que lee de S3).
- No hardcodear *Access Keys* en el código: usar variables de entorno, roles IAM o servicios como Secrets Manager / Parameter Store.
- Habilitar **AWS CloudTrail** para auditar quién hizo qué (queda registrado cada llamada a la API).
- Activar **AWS Config** o **Security Hub** si se necesita monitoreo continuo de cumplimiento y buenas prácticas.
- Cifrar datos en reposo (S3, RDS, EBS) y en tránsito (TLS) siempre que sea posible; en muchos servicios el cifrado por defecto ya viene activado o es gratuito de habilitar.

---

## Certificaciones de AWS

AWS ofrece certificaciones oficiales organizadas en cuatro niveles. Todos los exámenes se rinden en centros **Pearson VUE** o con supervisión online, y tienen una validez de **3 años**. La información y el registro se hacen desde el portal oficial:

- **Portal de certificaciones**: https://aws.amazon.com/certification/
- **Registro / agendar examen (AWS Certification Account, vía Pearson VUE o PSI)**: https://aws.amazon.com/certification/certification-prep/register/
- **Preparación gratuita y de pago (AWS Skill Builder)**: https://skillbuilder.aws/

### Nivel Fundamental (sin experiencia previa requerida)

| Certificación | Enlace oficial |
|---|---|
| AWS Certified Cloud Practitioner | https://aws.amazon.com/certification/certified-cloud-practitioner/ |
| AWS Certified AI Practitioner | https://aws.amazon.com/certification/certified-ai-practitioner/ |

### Nivel Asociado (se recomienda experiencia previa)

| Certificación | Enlace oficial |
|---|---|
| AWS Certified Solutions Architect – Associate | https://aws.amazon.com/certification/certified-solutions-architect-associate/ |
| AWS Certified Developer – Associate | https://aws.amazon.com/certification/certified-developer-associate/ |
| AWS Certified CloudOps Engineer – Associate *(antes "SysOps Administrator – Associate")* | https://aws.amazon.com/certification/certified-cloudops-engineer-associate/ |
| AWS Certified Data Engineer – Associate | https://aws.amazon.com/certification/certified-data-engineer-associate/ |
| AWS Certified Machine Learning Engineer – Associate | https://aws.amazon.com/certification/certified-machine-learning-engineer-associate/ |

### Nivel Profesional (2+ años de experiencia recomendados)

| Certificación | Enlace oficial |
|---|---|
| AWS Certified Solutions Architect – Professional | https://aws.amazon.com/certification/certified-solutions-architect-professional/ |
| AWS Certified DevOps Engineer – Professional | https://aws.amazon.com/certification/certified-devops-engineer-professional/ |
| AWS Certified Generative AI Developer – Professional | https://aws.amazon.com/certification/certified-generative-ai-developer-professional/ |

### Especialidades

| Certificación | Enlace oficial |
|---|---|
| AWS Certified Security – Specialty | https://aws.amazon.com/certification/certified-security-specialty/ |

> **Nota**: AWS actualiza periódicamente su catálogo de certificaciones (retira exámenes antiguos, lanza nuevos, o los renombra como pasó con SysOps → CloudOps). Antes de inscribirte, conviene revisar el portal oficial para confirmar que la certificación sigue vigente y ver la guía de examen (*exam guide*) más reciente, disponible en la propia página de cada certificación.

### Cómo prepararse

1. Elegir la certificación según el rol/objetivo (por ejemplo, Cloud Practitioner como punto de entrada si no se tiene experiencia previa en la nube).
2. Revisar la **guía de examen** (*exam guide*) oficial de la certificación elegida — enlazada en cada una de las páginas de arriba — para ver los dominios y su ponderación.
3. Usar **AWS Skill Builder** (https://skillbuilder.aws/) para los cursos de preparación gratuitos, sets de preguntas de práctica oficiales y, opcionalmente, los exámenes de práctica de pago.
4. Practicar en una cuenta real dentro del **Free Tier** (https://aws.amazon.com/free/) para afianzar los conceptos con la consola y la CLI.
5. Registrarse y agendar el examen desde https://aws.amazon.com/certification/certification-prep/register/.
