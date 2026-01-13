# Guía de Despliegue: Integración CloudAltio en GCP

## Descripción general

Este documento describe cómo habilitar la integración de **CloudAltio** en Google Cloud Platform (GCP) mediante dos opciones de despliegue:

- **Opción 1 – Scripts `.sh`**:  
  - Creación de una Service Account con permisos de solo lectura sobre BigQuery y el proyecto  
  - Creación de un dataset de BigQuery donde se almacenarán las exportaciones de datos de costos para CloudAltio

- **Opción 2 – Terraform (IaC)**:  
  - Mismos recursos que la opción 1, pero gestionados como infraestructura como código

El objetivo es proporcionar a CloudAltio una identidad técnica (Service Account) y un dataset de BigQuery donde se puedan alojar los datos de costes que la aplicación utilizará para análisis FinOps.

---

## Requisitos previos

### Herramientas necesarias

Para la **Opción 1 – Scripts .sh**, necesitas:

- **gcloud CLI** instalada y configurada:
  - Autenticada (`gcloud auth login`)
  - Con un proyecto activo (`gcloud config set project YOUR_PROJECT_ID`) [web:91]
- **bq CLI** (parte de Google Cloud SDK), instalada y configurada [web:98]

Para la **Opción 2 – Terraform**, necesitas:

- **Terraform** 1.x instalado  
- **Proveedor Google** versión `~> 5.0` (se define en `providers.tf`) [web:99]  
- Credenciales GCP para Terraform (p.ej. variable de entorno `GOOGLE_APPLICATION_CREDENTIALS` o autenticación por `gcloud auth application-default login`) [web:96]

### Permisos mínimos en GCP

La identidad (usuario/SA) que ejecuta la configuración debe tener permisos suficientes para:

- Crear Service Accounts:
  - `roles/iam.serviceAccountAdmin` (o equivalente) [web:90]
- Asignar IAM roles a nivel de proyecto:
  - `roles/resourcemanager.projectIamAdmin` (o equivalente)
- Crear datasets de BigQuery:
  - `roles/bigquery.admin` o permisos específicos de creación de datasets [web:95]

### Información requerida

Antes de ejecutar cualquiera de las opciones, asegúrate de tener:

- **Project ID** de GCP donde se desplegará CloudAltio (p.ej. `mi-proyecto-prod`)
- **Ubicación de BigQuery**:
  - Por defecto en los scripts y Terraform: `us-central1` (Iowa) [web:95]

---

## Recursos que se van a crear

### Service Account: `cloudaltio`

**Nombre interno (account_id)**: `cloudaltio`  
**Display name**:  
- Script: `SA para exportacion cloudaltio`  
- Terraform: `Cloud Altio`  
**Descripción**: `Cloud altio, service account de orion` (Terraform)

**Roles asignados a nivel de proyecto**:

- `roles/bigquery.dataViewer`  
  - Permite leer datos de BigQuery (tablas/vistas) [web:95]
- `roles/bigquery.jobUser`  
  - Permite lanzar jobs de BigQuery, incluyendo consultas de solo lectura [web:95]
- `roles/viewer`  
  - Permiso de solo lectura sobre la mayoría de los recursos del proyecto (similar a un “read-only” global)

**Objetivo**:  
Esta Service Account será utilizada por CloudAltio para:

- Leer datasets/tablas de BigQuery que contengan los datos de costos  
- Ejecutar consultas (jobs) sobre esos datos  
- Inspeccionar recursos del proyecto con fines de análisis (solo lectura)

### BigQuery Dataset: `cloudaltioexport`

**ID del dataset**: `cloudaltioexport`  
**Ubicación**:
- Script `.sh`: `us-central1` (fijo)  
- Terraform: `var.bq_location` (por defecto `us-central1`)

**Descripción**:

- Script: `Dataset para exportaciones cloudaltio`  
- Terraform (labels):
  - `owner = "cloudaltio"`  
  - `provider = "orion"`

**Objetivo**:  
Dataset que almacenará las tablas de exportación de datos de costes y uso que CloudAltio consumirá para generar análisis FinOps, dashboards y reportes.

---

## Opción 1 – Despliegue mediante scripts `.sh`

Esta opción es ideal para despliegues rápidos desde consola, sin necesidad de mantener estado de Terraform. Crea los mismos recursos que la opción de IaC, pero de forma imperativa.

### 1. Creación de la Service Account

**Archivo**: `create-cloudaltio-sa.sh` (nombre sugerido)

```bash
#!/bin/bash

# Variables fijas según tu solicitud
SA_NAME="cloudaltio"
PROJECT_ID=$(gcloud config get-value project)
ROLES=(
  "roles/bigquery.dataViewer"
  "roles/bigquery.jobUser"
  "roles/viewer"
)

echo "Iniciando creación de Service Account: $SA_NAME en el proyecto: $PROJECT_ID"

# 1. Crear la Service Account
gcloud iam service-accounts create $SA_NAME \
    --display-name="SA para exportacion cloudaltio"

# 2. Asignar los roles solicitados
for ROLE in "${ROLES[@]}"; do
  echo "Asignando rol: $ROLE..."
  gcloud projects add-iam-policy-binding $PROJECT_ID \
      --member="serviceAccount:$SA_NAME@$PROJECT_ID.iam.gserviceaccount.com" \
      --role="$ROLE" \
      --quiet > /dev/null
done

echo "Configuración de IAM finalizada."

**1.1. Requisitos previos específicos**

- Haber configurado el proyecto activo:

bash

gcloud config set project YOUR_PROJECT_ID

- Tener permisos para crear Service Accounts y modificar IAM [web:90][web:91]

**1.2. Pasos de ejecución**

1. Guardar el script:
    - Crea un archivo create-cloudaltio-sa.sh
    - Pega el contenido anterior
2. Dar permisos de ejecución:

bash

chmod +x create-cloudaltio-sa.sh

3. Ejecutar el script:

bash

./create-cloudaltio-sa.sh

4. Salida esperada (ejemplo):

text

Iniciando creación de Service Account: cloudaltio en el proyecto: mi-proyecto-prod

Asignando rol: roles/bigquery.dataViewer...

Asignando rol: roles/bigquery.jobUser...

Asignando rol: roles/viewer...

Configuración de IAM finalizada.

**1.3. Validación**

- Listar Service Accounts:

bash

gcloud iam service-accounts list \


--filter="email:cloudaltio@$PROJECT_ID.iam.gserviceaccount.com"

- Ver roles asignados:

bash

gcloud projects get-iam-policy $PROJECT_ID \

--filter="bindings.members:cloudaltio@$PROJECT_ID.iam.gserviceaccount.com" \

--format="table(bindings.role)"

**2. Creación del Dataset de BigQuery**

**Archivo** : create-cloudaltio-dataset.sh (nombre sugerido)

bash

#!/bin/bash

_# Variables fijas_

DATASET_ID="cloudaltioexport"

LOCATION="us-central1"

echo "Creando dataset $DATASET_ID..."

bq --location=$LOCATION mk \

--dataset \

--description "Dataset para exportaciones cloudaltio" \

$DATASET_ID

echo "Dataset $DATASET_ID creado exitosamente."

**2.1. Requisitos previos específicos**

- bq (CLI de BigQuery) instalado y autenticado:

bash


gcloud auth login

gcloud auth application-default login

- Proyecto por defecto configurado:

bash

gcloud config set project YOUR_PROJECT_ID

**2.2. Pasos de ejecución**

1. Guardar el script:
    - Crea un archivo create-cloudaltio-dataset.sh
    - Pega el contenido anterior
2. Dar permisos de ejecución:

bash

chmod +x create-cloudaltio-dataset.sh

3. Ejecutar el script:

bash

./create-cloudaltio-dataset.sh

4. Salida esperada:

text

Creando dataset cloudaltioexport...

Dataset cloudaltioexport creado exitosamente.

**2.3. Validación**

- Listar datasets en el proyecto:

bash

bq ls

- Ver detalles del dataset:

bash

bq show --format=prettyjson cloudaltioexport


**Opción 2 – Despliegue mediante Terraform (IaC)**

Esta opción es ideal para entornos donde se requiere control de cambios, versionado
y despliegues repetibles en múltiples proyectos.

**Estructura de archivos**

Se recomiendan los siguientes archivos:

- main.tf – Definición de recursos (Service Account, roles IAM, dataset)
- providers.tf – Definición del proveedor de Google
- variables.tf – Parámetros de entrada (project_id, ubicación de BigQuery)
**1. providers.tf**

Configura el proveedor de GCP y la versión requerida.

text

terraform {

required_providers {

google = {

source = "hashicorp/google"

version = "~> 5.0"

}

}

}

provider "google" {

project = var.project_id

}

**2. variables.tf**

Define las variables de proyecto y ubicación de BigQuery.


text

variable "project_id" {

description = "ID del proyecto donde se crean los recursos"

type = string

}

variable "bq_location" {

description = "Ubicación del dataset de BigQuery"

type = string

default = "us-central1"

}

**3. main.tf**

Define los recursos principales para CloudAltio.

text

# Service Account

resource "google_service_account" "cloudaltio" {

project = var.project_id

account_id = "cloudaltio"

display_name = "Cloud Altio"

description = "Cloud altio, service account de orion"

}

# Project IAM Roles

resource "google_project_iam_member" "cloudaltio_roles" {


for_each = toset([

"roles/bigquery.dataViewer",

"roles/bigquery.jobUser",

"roles/viewer"

])

project = var.project_id

role = each.value

member = "serviceAccount:${google_service_account.cloudaltio.email}"

}

# BigQuery Dataset

resource "google_bigquery_dataset" "dataset" {

project = var.project_id

dataset_id = "cloudaltioexport"

location = var.bq_location

labels = {

owner = "cloudaltio"

provider = "orion"

}

}

**Pasos de despliegue con Terraform**

**1. Preparar el directorio de trabajo**


1. Crea un directorio, por ejemplo:

bash

mkdir cloudaltio-gcp-terraform

cd cloudaltio-gcp-terraform

2. Crea los archivos:
    - main.tf
    - providers.tf
    - variables.tf
3. Pega el contenido correspondiente en cada archivo.
**2. Configurar variables**

Puedes pasar project_id y bq_location de varias formas:

- Archivo terraform.tfvars:

text

project_id = "mi-proyecto-prod"

bq_location = "us-central1"

- O vía CLI (override puntual):

bash

terraform apply -var="project_id=mi-proyecto-prod" - var="bq_location=us-central1"

**3. Inicializar Terraform**

bash

terraform init

Esto:

- Descargará el proveedor hashicorp/google versión ~> 5.0 [web:99]
- Inicializará el backend local (por defecto)
**4. Revisar el plan**

bash


terraform plan -var="project_id=mi-proyecto-prod"

Salida esperada (resumen):

- google_service_account.cloudaltio – create
- google_project_iam_member.cloudaltio_roles["roles/bigquery.dataViewer"] – c
    reate
- google_project_iam_member.cloudaltio_roles["roles/bigquery.jobUser"] – crea
    te
- google_project_iam_member.cloudaltio_roles["roles/viewer"] – create
- google_bigquery_dataset.dataset – create

Revisa que los recursos a crear sean correctos y que el project_id sea el deseado.

**5. Aplicar cambios**

bash

terraform apply -var="project_id=mi-proyecto-prod"

Terraform mostrará un plan y pedirá confirmación:

text

Plan: 5 to add, 0 to change, 0 to destroy.

Do you want to perform these actions?

Terraform will perform the actions described above.

Only 'yes' will be accepted to approve.

Enter a value:

Escribe yes y presiona Enter.

**6. Validación post-deploy**

**6.1. Validar Service Account**

bash

gcloud iam service-accounts list \


--filter="email:cloudaltio@mi-proyecto-prod.iam.gserviceaccount.com"

También puedes usar Terraform outputs (si los defines). Por ejemplo, añade
en main.tf:

text

output "cloudaltio_service_account_email" {

value = google_service_account.cloudaltio.email

description = "Email de la Service Account de CloudAltio"

}

Luego:

bash

terraform output cloudaltio_service_account_email

**6.2. Validar roles IAM**

bash

gcloud projects get-iam-policy mi-proyecto-prod \

--filter="bindings.members:cloudaltio@mi-proyecto-
prod.iam.gserviceaccount.com" \

--format="table(bindings.role)"

Deberías ver:

- roles/bigquery.dataViewer
- roles/bigquery.jobUser
- roles/viewer

**6.3. Validar dataset de BigQuery**

bash

bq ls

bq show --format=prettyjson cloudaltioexport

También puedes usar Terraform:

bash


terraform state show google_bigquery_dataset.dataset

**Comparación de opciones**

```
Aspecto Opción 1 – Scripts .sh Opción 2 – Terraform
```
```
Tipo de
despliegue Imperativo (por comandos) Declarativo (IaC)
```
```
Herramientas gcloud, bq terraform, proveedor hashicorp/google
```
```
Repetibilidad Menor (scripts manuales) Alta (plan y state gestionados por Terraform)
```
```
Control de
cambios Manual (repositorio de scripts) Integrado (git + Terraform)
```
```
Ideal para
```
```
Pruebas rápidas, entornos
pequeños
```
```
Producción, multi-proyecto, equipos
FinOps/DevOps
```
```
Idempotencia
```
```
Limitada (puede requerir checks
adicionales) Alta (Terraform detecta estado actual)
```
**Buenas prácticas y consideraciones**

**Seguridad de la Service Account**

- No crees claves externas (JSON) para esta Service Account a menos que sea
    estrictamente necesario.
- Preferir **Workload Identity Federation** o uso directo de la SA desde servicios
    de GCP.
- Limitar su uso exclusivamente a CloudAltio.

**Gobernanza y etiquetado**


- En BigQuery, se usan labels owner = "cloudaltio", provider = "orion".
    Puedes extender esta práctica a otros recursos en Terraform (si añades más)
    para facilitar reporting y control de costes.

**Gestión de costes**

- El dataset cloudaltioexport almacenará datos de costes y uso. El volumen de
    datos dependerá de:
       - Cantidad de servicios GCP usados
       - Retención de los datos de exportación
- Revisa periódicamente:
    - Tamaño del dataset
    - Consultas ejecutadas por CloudAltio (BigQuery Jobs)
- Considera configurar políticas de retención de tablas/particiones en BigQuery
    según tus necesidades FinOps.

**Deshacer/Eliminar recursos**

**Opción 1 – Scripts**

Tendrás que eliminar los recursos manualmente.

**Eliminar roles de la Service Account** :

bash

PROJECT_ID=$(gcloud config get-value project)

SA_EMAIL="cloudaltio@$PROJECT_ID.iam.gserviceaccount.com"

**for** ROLE **in** "roles/bigquery.dataViewer" "roles/bigquery.jobUser" "roles/viewer"; **do**

gcloud projects remove-iam-policy-binding $PROJECT_ID \

--member="serviceAccount:$SA_EMAIL" \

--role="$ROLE"

**done**


**Eliminar la Service Account** :

bash

gcloud iam service-accounts delete $SA_EMAIL

**Eliminar el dataset** :

bash

bq rm - r -f cloudaltioexport

_- r: elimina tablas dentro del dataset, - f: evita prompt interactivo._

**Opción 2 – Terraform**

Desde el directorio del proyecto Terraform:

bash

terraform destroy -var="project_id=mi-proyecto-prod"

Terraform eliminará:

- Service Account cloudaltio
- Roles IAM asociados
- Dataset cloudaltioexport

**Recursos adicionales**

**Documentación oficial de GCP**

- **Service Accounts (IAM)** :
    - Crear Service Accounts (GUI y gcloud):
       https://cloud.google.com/iam/docs/service-accounts-create [web:90]
    - Referencia de gcloud iam service-accounts create:
       https://cloud.google.com/sdk/gcloud/reference/iam/service-
       accounts/create [web:100]
- **IAM & Policy Binding** :
    - Administración de IAM y políticas de acceso:
       https://cloud.google.com/iam/docs/overview [web:90]


- **BigQuery** :
    - Ubicaciones de BigQuery (regiones y multirregiones):
       https://cloud.google.com/bigquery/docs/locations [web:95]
    - Crear datasets en BigQuery (incluye ejemplo con bq mk):
       https://cloud.google.com/bigquery/docs/datasets [web:104]
- **CLI GCP (gcloud & bq)** :
    - Referencia completa de gcloud:
       https://cloud.google.com/sdk/gcloud/reference [web:91]
    - Referencia completa de bq:
       https://cloud.google.com/bigquery/docs/bq-command-line-
       tool [web:98]

**Documentación oficial de Terraform**

- **Proveedor Google** :
    - Página del proveedor hashicorp/google:
       https://registry.terraform.io/providers/hashicorp/google/latest [web:99]
    - Recurso google_bigquery_dataset:
       https://registry.terraform.io/providers/hashicorp/google/latest/docs/res
       ources/bigquery_dataset [web:99]
    - Recurso google_service_account:
       https://registry.terraform.io/providers/hashicorp/google/latest/docs/res
       ources/google_service_account [web:96]
- **Conceptos de Terraform** :
    - Bloque provider y configuración básica:
       https://developer.hashicorp.com/terraform/language/providers/configu
       ration [web:105]
    - Tutorial “Build infrastructure on GCP with Terraform”:
       https://developer.hashicorp.com/terraform/tutorials/gcp-get-
       started/google-cloud-platform-build [web:108]

**Recursos FinOps / CloudAltio**

- **Exportar datos de billing de GCP a BigQuery** (para alimentar CloudAltio):
    https://cloud.google.com/billing/docs/how-to/export-data-bigquery [web:104]


**Documento creado por** : FinOps by Orion
**Última actualización** : 13/01/
**Versión** : 1.0 (Integración CloudAltio en GCP – SA + Dataset)
**Plataforma** : Google Cloud Platform (GCP)
