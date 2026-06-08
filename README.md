# MLflow con AWS: EC2 + RDS PostgreSQL + S3

Esta guia resume el flujo completo para montar un servidor de MLflow en AWS usando:

- EC2 como maquina donde corre `mlflow server`
- RDS PostgreSQL como backend store para metadata
- S3 como artifact store para modelos y archivos

La arquitectura final queda asi:

```text
Notebook local
    |
    | mlflow.set_tracking_uri("http://EC2_PUBLIC_DNS:5000")
    v
EC2: mlflow server
    |
    | metadata: experiments, runs, params, metrics
    v
RDS PostgreSQL

EC2: mlflow server
    |
    | artifacts: modelos, archivos, outputs
    v
S3 bucket
```

## 0. Advertencias importantes

- EC2, RDS y S3 deben estar preferentemente en la misma region. Si conectas servicios entre regiones puedes generar latencia, complejidad y posibles costos adicionales de transferencia.
- No pegues secretos en notebooks ni en repositorios: contrasenas de RDS, AWS Secret Access Key, llaves `.pem`, etc.
- Para servidores en EC2, es mejor usar un IAM Role en lugar de `aws configure` con Access Key y Secret Access Key.
- No abras el puerto `5000` a todo internet si no es necesario. Para practicar, usa tu IP publica con `/32`.
- Al terminar de practicar, deten o elimina recursos si no quieres cargos: EC2, RDS, snapshots/backups y objetos en S3.
- Consulta siempre la pagina oficial de Free Tier y pricing. Los limites pueden cambiar.

Referencias utiles:

- AWS Free Tier para RDS: https://aws.amazon.com/rds/free/
- IAM Roles para EC2: https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/iam-roles-for-amazon-ec2.html
- Security groups de EC2: https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-network-security.html

## 1. Recursos necesarios en AWS

Necesitas tener creados:

1. Una instancia EC2 con Amazon Linux 2023.
2. Un bucket S3, por ejemplo:

```text
bucket-platzi-ejemplo
```

3. Una base RDS PostgreSQL, por ejemplo:

```text
database-2.c2tmeckmqn1j.us-east-1.rds.amazonaws.com
```

4. Un security group para EC2.
5. Un security group para RDS.
6. Un IAM Role para que EC2 pueda escribir en S3.

## 2. Security group de EC2

En AWS Console:

```text
EC2 -> Instances -> selecciona la instancia -> Security -> Security groups
```

En el security group de la EC2, configura reglas de entrada:

```text
Tipo              Protocolo  Puerto  Origen
SSH               TCP        22      TU_IP_PUBLICA/32
TCP personalizado TCP        5000    TU_IP_PUBLICA/32
```

Ejemplo:

```text
SSH               TCP        22      187.170.78.1/32
TCP personalizado TCP        5000    187.170.78.1/32
```

La regla del puerto `5000` debe estar en **reglas de entrada**, no en reglas de salida.

Las reglas de salida deben permitir que la EC2 descargue paquetes y hable con AWS. Para el ejercicio puedes dejar:

```text
Tipo             Protocolo  Puerto  Destino
Todo el trafico  All        All     0.0.0.0/0
```

Como minimo, para instalar paquetes necesitas salida HTTPS:

```text
HTTPS  TCP  443  0.0.0.0/0
```

## 3. Security group de RDS

RDS debe aceptar conexiones PostgreSQL desde la EC2.

En el security group de RDS, configura una regla de entrada:

```text
Tipo        Protocolo  Puerto  Origen
PostgreSQL  TCP        5432    SECURITY_GROUP_DE_EC2
```

Lo ideal es que el origen sea el security group de EC2, no `0.0.0.0/0`.

Esto permite:

```text
EC2 -> RDS PostgreSQL puerto 5432
```

## 4. IAM Role para S3

No uses `aws configure` con Access Key y Secret Key dentro de EC2 para este ejercicio. Es mas seguro asignar permisos a la EC2 con un IAM Role.

En AWS Console:

```text
IAM -> Roles -> Create role
```

Selecciona:

```text
Trusted entity: AWS service
Use case: EC2
```

Para practicar, puedes adjuntar:

```text
AmazonS3FullAccess
```

Nombre sugerido:

```text
EC2-MLflow-S3-Role
```

Luego asignalo a tu instancia:

```text
EC2 -> Instances -> selecciona instancia -> Actions -> Security -> Modify IAM role
```

Selecciona:

```text
EC2-MLflow-S3-Role
```

Guarda. No necesitas reiniciar la instancia.

## 5. Entrar a la EC2 por SSH

Desde PowerShell en tu PC local:

```powershell
ssh -i "G:/Escritorio/Cursos/MLOPS/ejemplo-v1.pem" ec2-user@EC2_PUBLIC_DNS
```

Ejemplo:

```powershell
ssh -i "G:/Escritorio/Cursos/MLOPS/ejemplo-v1.pem" ec2-user@ec2-100-31-39-153.compute-1.amazonaws.com
```

Si entras correctamente, veras algo parecido a:

```bash
[ec2-user@ip-172-31-37-99 ~]$
```

Eso significa que ya estas dentro de Linux en EC2.

No ejecutes comandos de Windows dentro de EC2. Por ejemplo, esto es incorrecto dentro de EC2:

```bash
Set-ExecutionPolicy
G:\Escritorio\Cursos\MLOPS\.venv\Scripts\Activate.ps1
```

Eso solo aplica a PowerShell local de Windows. La EC2 tiene su propio Python y sus propios paquetes.

## 6. Verificar credenciales AWS desde EC2

Dentro de EC2:

```bash
aws sts get-caller-identity
```

Si devuelve JSON con `UserId`, `Account` y `Arn`, el IAM Role funciona.

Luego prueba S3:

```bash
aws s3 ls s3://bucket-platzi-ejemplo
```

Tambien puedes probar:

```bash
aws s3 ls
```

Si `aws s3 ls` da AccessDenied pero `aws s3 ls s3://bucket-platzi-ejemplo` funciona, puede ser suficiente para MLflow. El primero lista todos los buckets de la cuenta; el segundo solo consulta el bucket que usaras.

## 7. Instalar dependencias en EC2

Dentro de EC2:

```bash
sudo dnf update -y
sudo dnf install -y python3-pip
python3 -m pip install --user mlflow boto3 psycopg2-binary
export PATH=$PATH:$HOME/.local/bin
mlflow --version
```

Si `dnf` falla con timeout como:

```text
Curl error (28): Timeout was reached
```

revisa las reglas de salida del security group de EC2. La instancia necesita salida a internet, normalmente por HTTPS `443`, para descargar paquetes.

## 8. Probar que MLflow abre el puerto 5000

Antes de conectar RDS y S3, prueba algo simple:

```bash
mlflow ui --host 0.0.0.0 --port 5000
```

La terminal debe quedarse ocupada. Eso es normal.

Desde tu navegador local abre:

```text
http://EC2_PUBLIC_DNS:5000
```

Ejemplo:

```text
http://ec2-100-31-39-153.compute-1.amazonaws.com:5000
```

Si esto carga, entonces:

```text
EC2 funciona
MLflow esta instalado
El puerto 5000 esta abierto
El security group de EC2 esta bien
```

Deten esta prueba con:

```text
Ctrl + C
```

## 9. Obtener datos de RDS

En AWS Console:

```text
RDS -> Databases -> selecciona tu database
```

Necesitas:

```text
DB_USER      master username
DB_PASSWORD  contrasena del usuario
DB_ENDPOINT  endpoint de RDS
DB_NAME      nombre de la base
```

Ejemplo de datos:

```text
DB_USER      postgres
DB_ENDPOINT  database-2.c2tmeckmqn1j.us-east-1.rds.amazonaws.com
DB_NAME      postgres
```

AWS no te muestra la contrasena actual. Si no la recuerdas:

```text
RDS -> Databases -> selecciona database -> Modify -> Master password
```

Cambia la contrasena y elige:

```text
Apply immediately
```

Para evitar problemas en la URI de MLflow, usa una contrasena solo con letras y numeros. Evita caracteres como:

```text
@ # / : ? & %
```

## 10. Probar conexion EC2 -> RDS con psql

Instala el cliente PostgreSQL:

```bash
sudo dnf install -y postgresql15
```

Prueba conexion:

```bash
psql "host=database-2.c2tmeckmqn1j.us-east-1.rds.amazonaws.com port=5432 dbname=postgres user=postgres sslmode=require"
```

Te pedira la contrasena:

```text
Password for user postgres:
```

Si conecta, veras:

```text
postgres=>
```

Eso significa que EC2 ya puede conectarse a RDS.

Para salir:

```sql
\q
```

Si aparece:

```text
FATAL: password authentication failed for user "postgres"
```

la contrasena esta mal.

Si aparece:

```text
connection timed out
```

normalmente falta abrir el puerto `5432` en el security group de RDS desde el security group de EC2.

## 11. Levantar MLflow server con RDS + S3

Dentro de EC2, ejecuta en una sola linea:

```bash
mlflow server --host 0.0.0.0 --port 5000 --backend-store-uri "postgresql://postgres:TU_PASSWORD@database-2.c2tmeckmqn1j.us-east-1.rds.amazonaws.com:5432/postgres?sslmode=require" --default-artifact-root "s3://bucket-platzi-ejemplo"
```

Reemplaza:

```text
TU_PASSWORD
```

por tu contrasena real de RDS. No la guardes en el repositorio.

La terminal debe quedarse ocupada. Eso significa que MLflow esta corriendo.

Luego abre desde tu navegador local:

```text
http://ec2-100-31-39-153.compute-1.amazonaws.com:5000
```

Si carga, ya tienes el montaje completo:

```text
MLflow server en EC2
Metadata en RDS PostgreSQL
Artifacts en S3
```

## 12. Usar el servidor desde un notebook local

En tu notebook:

```python
import mlflow

mlflow.set_tracking_uri("http://ec2-100-31-39-153.compute-1.amazonaws.com:5000")
mlflow.set_experiment("experimento_aws")
```

Luego registra un run:

```python
with mlflow.start_run(run_name="run_desde_notebook_local"):
    mlflow.log_param("modelo", "logistic_regression")
    mlflow.log_metric("accuracy", 0.95)
```

El notebook manda los logs a EC2:

```text
Notebook local -> EC2 MLflow server -> RDS + S3
```

## 13. Problemas comunes

### 13.1 Placeholder pegado literal en SSH

Error:

```text
Could not resolve hostname <ec2_public_dns>
```

Causa:

```text
<EC2_PUBLIC_DNS>
```

era un placeholder, no un valor real.

Solucion:

```powershell
ssh -i "G:/Escritorio/Cursos/MLOPS/ejemplo-v1.pem" ec2-user@ec2-100-31-39-153.compute-1.amazonaws.com
```

### 13.2 Faltaba usuario `ec2-user`

Si ejecutas:

```powershell
ssh -i "key.pem" ec2-100-31-39-153.compute-1.amazonaws.com
```

SSH puede intentar entrar con tu usuario local de Windows.

Usa:

```powershell
ssh -i "key.pem" ec2-user@ec2-100-31-39-153.compute-1.amazonaws.com
```

### 13.3 Permisos de la llave `.pem`

Error:

```text
UNPROTECTED PRIVATE KEY FILE
Permissions are too open
```

La llave `.pem` debe ser legible solo por tu usuario.

En PowerShell local:

```powershell
$path = "G:\Escritorio\Cursos\MLOPS\ejemplo-v1.pem"
$me = "$env:USERDOMAIN\$env:USERNAME"

icacls $path /inheritance:r
icacls $path /grant:r "${me}:R"
```

Si se bloquea demasiado:

```powershell
takeown /F $path
icacls $path /inheritance:r
icacls $path /grant:r "${me}:R"
```

### 13.4 `aws configure` con credenciales mal capturadas

Error:

```text
SignatureDoesNotMatch
```

En lugar de guardar Access Key y Secret Key en EC2, usamos IAM Role para EC2.

Si ya habia una configuracion mala:

```bash
mv ~/.aws ~/.aws_backup
```

Luego verificamos:

```bash
aws sts get-caller-identity
```

### 13.5 IAM Role no asignado a EC2

Error:

```text
Unable to locate credentials
```

Causa:

La EC2 no tenia IAM Role asignado.

Solucion:

```text
EC2 -> Instances -> Actions -> Security -> Modify IAM role
```

Asignar:

```text
EC2-MLflow-S3-Role
```

### 13.6 Puerto 5000 en reglas de salida en vez de entrada

Error en navegador:

```text
ERR_CONNECTION_TIMED_OUT
```

Causa:

El puerto `5000` estaba abierto en reglas de salida, no en reglas de entrada.

Solucion:

```text
Security group EC2 -> Reglas de entrada -> TCP personalizado -> 5000 -> TU_IP/32
```

### 13.7 `ERR_CONNECTION_REFUSED`

Error:

```text
ERR_CONNECTION_REFUSED
```

Interpretacion:

La red ya permite llegar a la EC2, pero no hay ningun proceso escuchando en `5000`, o MLflow fallo al arrancar.

Diagnostico:

```bash
ss -ltnp | grep 5000
```

Prueba simple:

```bash
mlflow ui --host 0.0.0.0 --port 5000
```

Si esto funciona, el problema no es el puerto `5000`; el problema esta en RDS/S3 o en el comando de `mlflow server`.

### 13.8 EC2 sin salida a internet

Error:

```text
Curl error (28): Timeout was reached
```

Causa:

La EC2 no podia descargar paquetes con `dnf`.

Solucion:

Revisar reglas de salida del security group. Para el ejercicio, permitir:

```text
Todo el trafico -> 0.0.0.0/0
```

o minimo:

```text
HTTPS TCP 443 -> 0.0.0.0/0
```

### 13.9 Contrasena incorrecta de RDS

Error:

```text
FATAL: password authentication failed for user "postgres"
```

Causa:

La contrasena de RDS era incorrecta.

Solucion:

Cambiarla en:

```text
RDS -> Databases -> database -> Modify -> Master password
```

Luego probar otra vez:

```bash
psql "host=database-2.c2tmeckmqn1j.us-east-1.rds.amazonaws.com port=5432 dbname=postgres user=postgres sslmode=require"
```

### 13.10 Endpoint incorrecto de RDS

Inicialmente se intento conectar a:

```text
database-1.c2tmeckmqn1j.us-east-1.rds.amazonaws.com
```

pero la conexion correcta quedo con la segunda base:

```text
database-2.c2tmeckmqn1j.us-east-1.rds.amazonaws.com
```

Siempre confirma el endpoint real en:

```text
RDS -> Databases -> selecciona database -> Connectivity & security -> Endpoint
```

## 14. Checklist final

Antes de correr MLflow completo, verifica:

```text
[ ] EC2 esta running.
[ ] RDS esta available.
[ ] EC2 y RDS estan en la misma region.
[ ] EC2 tiene IAM Role con permisos sobre S3.
[ ] aws sts get-caller-identity funciona en EC2.
[ ] aws s3 ls s3://bucket-platzi-ejemplo funciona en EC2.
[ ] Security group EC2 permite entrada 22 desde tu IP.
[ ] Security group EC2 permite entrada 5000 desde tu IP.
[ ] Security group RDS permite entrada 5432 desde el security group de EC2.
[ ] psql conecta desde EC2 hacia RDS.
[ ] mlflow ui --host 0.0.0.0 --port 5000 carga en el navegador.
[ ] mlflow server con postgresql + s3 arranca sin errores.
```

## 15. Apagar recursos al terminar

Para evitar cargos:

- Deten la instancia EC2 si no la usas.
- Deten o elimina RDS si ya no lo necesitas.
- Revisa snapshots/backups de RDS.
- Revisa objetos guardados en S3.
- Activa AWS Budgets o alertas de billing si vas a seguir practicando.

