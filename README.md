# AWS - Guía de Despliegue: CloudAltio Integration Stack v1.3

## Descripción General

Este template de CloudFormation despliega la infraestructura completa necesaria para integrar la plataforma CloudAltio con tu cuenta de AWS. La solución automatiza la creación de un rol IAM cross-account con permisos de solo lectura, un bucket S3 para almacenar datos de costos, y una exportación automática en formato FOCUS 1.2 (FinOps Open Cost and Usage Specification) para análisis avanzado de costos en la nube.

## ¿Qué es CloudAltio?

CloudAltio es una plataforma de FinOps que permite visualizar, analizar y optimizar los costos de AWS. Esta integración proporciona a CloudAltio acceso de solo lectura a tus recursos y datos de facturación sin comprometer la seguridad de tu cuenta.

## Requisitos Previos

### Permisos Necesarios

Tu usuario o rol IAM debe tener los siguientes permisos:

- **IAM**: `iam:CreateRole`, `iam:PutRolePolicy`, `iam:AttachRolePolicy`, `iam:CreatePolicy`
- **S3**: `s3:CreateBucket`, `s3:PutBucketPolicy`, `s3:PutLifecycleConfiguration`
- **BCM Data Exports**: `bcm-data-exports:CreateExport`
- **CloudFormation**: `cloudformation:CreateStack`, `cloudformation:DescribeStacks`

### Región Recomendada

**us-east-1 (N. Virginia)** - Esta es la región recomendada porque:
- AWS Billing and Cost Management opera principalmente desde us-east-1
- AWS Data Exports tiene mejor disponibilidad en esta región
- Los costos de almacenamiento S3 son competitivos

### Información Requerida

Antes de iniciar el despliegue, necesitas:

1. **Account ID de CloudAltio**: Proporcionado por el equipo de CloudAltio (reemplazar el valor por defecto `123456789012`)
2. **Acceso a AWS Management Console** o **AWS CLI configurado**
3. **Permisos de administrador** o equivalentes en la cuenta AWS

## Parámetros del Template

| Parámetro | Tipo | Descripción | Valor por Defecto | Acción Requerida |
|-----------|------|-------------|-------------------|------------------|
| `AccountIdForCloudAltio` | String | AWS Account ID de la cuenta de CloudAltio que asumirá el rol para acceder a tus datos | `123456789012` | **Actualizar obligatoriamente** con el ID proporcionado por CloudAltio |

## Recursos Desplegados

### 1. CloudAltioRole (IAM Role)

**Tipo**: `AWS::IAM::Role`  
**Nombre**: `CloudAltioRole`  
**Tag**: `Env:CloudAltio`

Rol IAM cross-account con tres capas de permisos de solo lectura:

#### Política AWS Administrada
- **ViewOnlyAccess**: Proporciona acceso de solo lectura a metadatos de configuración de recursos AWS sin permitir acceso al contenido de los datos almacenados

#### Políticas Inline Personalizadas

**CloudAltioAdditionalResourceReadOnly**:
Permisos de lectura extendidos para más de 150 servicios AWS, incluyendo:
- **Compute**: EC2, ECS, EKS, Lambda, Batch
- **Storage**: S3 (listado y metadatos), EFS, FSx, Glacier
- **Database**: RDS, DynamoDB, Redshift, ElastiCache
- **Networking**: VPC, ELB, CloudFront, Route 53
- **Analytics**: Athena, Glue, Kinesis, EMR
- **Monitoring**: CloudWatch, CloudTrail, X-Ray
- **Security**: IAM (Get*), KMS, Secrets Manager, Security Hub
- **Developer Tools**: CodeBuild, CodePipeline, CodeCommit
- **Machine Learning**: SageMaker (implícito en ViewOnly), Personalize

**CloudAltioCURReadOnly**:
Permisos específicos para datos de facturación y gestión de costos:
- Cost Explorer: Todas las operaciones de lectura (`ce:Get*`, `ce:Describe*`, `ce:List*`)
- AWS Budgets: Lectura de presupuestos y alertas
- AWS Organizations: Información de cuentas y estructura organizacional
- Pricing API: Acceso completo para consultas de precios
- Savings Plans: Lectura de planes de ahorro y recomendaciones
- Cost and Usage Reports: Lectura de definiciones de reportes

**Trust Policy**:
El rol solo puede ser asumido por la cuenta de CloudAltio especificada en el parámetro, previniendo accesos no autorizados.

### 2. CostandUsageReportBucket (S3 Bucket)

**Tipo**: `AWS::S3::Bucket`  
**Nombre**: `cloudaltio-{account-id}` (ejemplo: `cloudaltio-987654321098`)  
**Tag**: `Env:CloudAltio`

Bucket S3 configurado con las siguientes características de seguridad y gestión:

**Seguridad**:
- Access Control: Private
- Bloqueo de acceso público activado en todos los niveles:
  - BlockPublicAcls: true
  - BlockPublicPolicy: true
  - IgnorePublicAcls: true
  - RestrictPublicBuckets: true

**Gestión de Ciclo de Vida**:
- Regla: `ExpireOldObjects`
- Los objetos se eliminan automáticamente después de **200 días**
- Esto ayuda a controlar costos de almacenamiento manteniendo solo datos relevantes

**Propósito**: Almacenar las exportaciones diarias de datos de costos en formato FOCUS Parquet generadas por AWS Data Exports.

### 3. CostandUsageReportBucketPolicy (S3 Bucket Policy)

**Tipo**: `AWS::S3::BucketPolicy`

Política de bucket que define tres permisos específicos:

| Principal | Acciones | Recursos | Propósito |
|-----------|----------|----------|-----------|
| `billingreports.amazonaws.com` | `s3:GetBucketAcl`<br>`s3:GetBucketPolicy` | Bucket raíz | Permite al servicio de billing verificar permisos del bucket |
| `billingreports.amazonaws.com` | `s3:PutObject` | `bucket/*` | Permite escribir los reportes de costos exportados |
| CloudAltioRole (ARN dinámico) | `s3:GetObject`<br>`s3:GetObjectAcl` | `bucket/*` | Permite a CloudAltio leer los reportes almacenados |

### 4. FOCUSDataExport (BCM Data Export)

**Tipo**: `AWS::BCMDataExports::Export`  
**Nombre**: `focus-cloudaltio-{account-id}`  
**Tag**: `Env:CloudAltio`

Exportación automática de datos de costos configurada con:

**Especificación FOCUS**:
- Versión: FOCUS 1.2 (última versión estable)
- Query: `SELECT * FROM FOCUS_1_2`
- Estándar abierto de FinOps Foundation para datos de costos en la nube

**Formato de Archivo**:
- Tipo: Parquet (formato columnar optimizado)
- Compresión: Snappy (balance entre velocidad y tamaño)
- Modo de escritura: Sobrescribe reportes existentes

**Configuración de Entrega**:
- Bucket destino: El bucket creado por este stack
- Prefijo S3: `daily-v1/`
- Región: us-east-1
- Frecuencia: SYNCHRONOUS (actualizaciones diarias automáticas)

**Ventajas de FOCUS 1.2**:
- Esquema estandarizado y consistente
- Compatible con múltiples proveedores cloud
- Incluye métricas avanzadas de FinOps
- Soporte para análisis de compromisos (Reserved Instances, Savings Plans)

### 5. ObjectGetIAMPolicy (IAM Policy)

**Tipo**: `AWS::IAM::Policy`  
**Nombre**: `ObjectGetCostandUsageReports`

Política IAM adicional que se adjunta directamente al CloudAltioRole, garantizando permisos explícitos de lectura sobre los objetos del bucket S3 de reportes. Esta política complementa la bucket policy para asegurar acceso bidireccional.

## Arquitectura de la Solución

```
┌─────────────────────────────────────────────────────────────┐
│              Tu Cuenta AWS (Management/Payer)               │
│                                                             │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  CloudAltioRole (IAM Role)                            │  │
│  │  Tag: Env:CloudAltio                                  │  │
│  │  ┌─────────────────────────────────────────────────┐  │  │
│  │  │ Managed Policy: ViewOnlyAccess                  │  │  │
│  │  └─────────────────────────────────────────────────┘  │  │
│  │  ┌─────────────────────────────────────────────────┐  │  │
│  │  │ Inline: CloudAltioAdditionalResourceReadOnly    │  │  │
│  │  │ (150+ servicios AWS)                            │  │  │
│  │  └─────────────────────────────────────────────────┘  │  │
│  │  ┌─────────────────────────────────────────────────┐  │  │
│  │  │ Inline: CloudAltioCURReadOnly                   │  │  │
│  │  │ (Billing, CE, Organizations, Pricing)           │  │  │
│  │  └─────────────────────────────────────────────────┘  │  │
│  │  ┌─────────────────────────────────────────────────┐  │  │
│  │  │ Attached: ObjectGetCostandUsageReports          │  │  │
│  │  │ (S3 GetObject)                                  │  │  │
│  │  └─────────────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────────────┘  │
│                            │                                │
│                            │ AssumeRole                     │
│                            ▼                                │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  S3 Bucket: cloudaltio-{account-id}                   │  │
│  │  Tag: Env:CloudAltio                                  │  │
│  │  - Private Access + Public Block                      │  │
│  │  - Lifecycle: 200 días                                │  │
│  │  - Prefijo: daily-v1/                                 │  │
│  └───────────────────────────────────────────────────────┘  │
│                            ▲                                │
│                            │ PutObject (daily)              │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  BCM Data Exports                                     │  │
│  │  Tag: Env:CloudAltio                                  │  │
│  │  - Export: focus-cloudaltio-{account-id}             │  │
│  │  - Format: FOCUS 1.2 → Parquet (Snappy)              │  │
│  │  - Frequency: Daily (Synchronous)                     │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
                            │
                            │ AssumeRole + GetObject
                            ▼
              ┌──────────────────────────────┐
              │  Cuenta CloudAltio           │
              │  (Plataforma FinOps)         │
              │  - Análisis de costos        │
              │  - Optimización              │
              │  - Dashboards                │
              │  - Recomendaciones           │
              └──────────────────────────────┘
```

## Pasos de Implementación

### Opción 1: AWS Management Console (Interfaz Web)

#### Paso 1: Preparar el Template

1. Copia el contenido JSON completo del template
2. Guárdalo en un archivo local con nombre descriptivo: `cloudaltio-onboarding-v1.3.json`
3. Valida la sintaxis usando un validador JSON online (opcional pero recomendado)

#### Paso 2: Acceder a CloudFormation

1. Inicia sesión en **AWS Management Console**
2. En la barra de búsqueda superior, escribe "CloudFormation" y selecciónalo
3. **Importante**: Cambia la región a **us-east-1** (N. Virginia) en el selector de región superior derecho
4. Verifica que estés en la región correcta antes de continuar

#### Paso 3: Crear el Stack

1. Haz clic en el botón naranja **"Create stack"**
2. Selecciona **"With new resources (standard)"**
3. En "Prepare template", selecciona **"Template is ready"**
4. En "Template source", selecciona **"Upload a template file"**
5. Haz clic en **"Choose file"** y selecciona tu archivo `cloudaltio-onboarding-v1.3.json`
6. Haz clic en **"Next"**

#### Paso 4: Especificar Detalles del Stack

1. **Stack name**: Ingresa `cloudaltio-integration`
   - Usa solo letras, números y guiones
   - Debe ser único en tu cuenta y región

2. **Parameters**:
   - **AccountIdForCloudAltio**: Reemplaza `123456789012` con el Account ID real proporcionado por CloudAltio
   - Este ID es crítico - verifica que sea correcto

3. Haz clic en **"Next"**

#### Paso 5: Configurar Opciones del Stack

1. **Tags** (recomendado - agrega etiquetas para organización):
   ```
   Key: Environment     | Value: Production
   Key: ManagedBy       | Value: CloudFormation
   Key: Purpose         | Value: CloudAltio-Integration
   Key: Owner           | Value: FinOps-Team
   Key: CostCenter      | Value: IT-Operations
   ```

2. **Permissions**: 
   - Deja en blanco para usar permisos de tu usuario actual
   - O selecciona un rol IAM específico si tu organización lo requiere

3. **Stack failure options**: 
   - Selecciona **"Roll back all stack resources"**
   - Esto asegura una limpieza automática si algo falla

4. **Advanced options**: Deja los valores por defecto

5. Haz clic en **"Next"**

#### Paso 6: Revisar y Crear

1. Revisa cuidadosamente toda la configuración:
   - Nombre del stack
   - Parámetros (especialmente el Account ID)
   - Tags configurados

2. Desplázate hasta el final de la página

3. **CRÍTICO**: Marca la casilla de reconocimiento:
   - ☑ **"I acknowledge that AWS CloudFormation might create IAM resources with custom names"**
   - Sin marcar esta casilla, el stack fallará.

4. Haz clic en el botón azul **"Submit"**

#### Paso 7: Monitorear la Creación

1. Serás redirigido a la página de detalles del stack
2. El estado inicial será **"CREATE_IN_PROGRESS"**
3. Haz clic en la pestaña **"Events"** para ver el progreso en tiempo real
4. Los eventos se muestran en orden cronológico inverso (los más recientes arriba)

**Tiempo estimado**: 3-5 minutos

**Eventos esperados** (en orden):
```
CREATE_IN_PROGRESS    AWS::CloudFormation::Stack          cloudaltio-integration
CREATE_IN_PROGRESS    AWS::IAM::Role                      CloudAltioRole
CREATE_COMPLETE       AWS::IAM::Role                      CloudAltioRole
CREATE_IN_PROGRESS    AWS::S3::Bucket                     CostandUsageReportBucket
CREATE_COMPLETE       AWS::S3::Bucket                     CostandUsageReportBucket
CREATE_IN_PROGRESS    AWS::S3::BucketPolicy               CostandUsageReportBucketPolicy
CREATE_COMPLETE       AWS::S3::BucketPolicy               CostandUsageReportBucketPolicy
CREATE_IN_PROGRESS    AWS::BCMDataExports::Export         FOCUSDataExport
CREATE_COMPLETE       AWS::BCMDataExports::Export         FOCUSDataExport
CREATE_IN_PROGRESS    AWS::IAM::Policy                    ObjectGetIAMPolicy
CREATE_COMPLETE       AWS::IAM::Policy                    ObjectGetIAMPolicy
CREATE_COMPLETE       AWS::CloudFormation::Stack          cloudaltio-integration
```

5. Espera hasta que el estado cambie a **"CREATE_COMPLETE"** (color verde)

### Opción 2: AWS CLI (Línea de Comandos)

#### Prerrequisitos CLI

Verifica que tienes AWS CLI instalado y configurado:

```bash
# Verificar versión de AWS CLI
aws --version

# Verificar credenciales configuradas
aws sts get-caller-identity

# Verificar región configurada
aws configure get region
```

#### Crear el Stack con CLI

```bash
# Navegar al directorio donde está el template
cd /ruta/a/tu/template

# Crear el stack
aws cloudformation create-stack \
  --stack-name cloudaltio-integration \
  --template-body file://cloudaltio-onboarding-v1.3.json \
  --parameters ParameterKey=AccountIdForCloudAltio,ParameterValue=123456789012 \
  --capabilities CAPABILITY_NAMED_IAM \
  --region us-east-1 \
  --tags \
    Key=Environment,Value=Production \
    Key=ManagedBy,Value=CloudFormation \
    Key=Purpose,Value=CloudAltio-Integration \
    Key=Owner,Value=FinOps-Team

# Respuesta esperada:
# {
#     "StackId": "arn:aws:cloudformation:us-east-1:123456789012:stack/cloudaltio-integration/..."
# }
```

**Nota importante**: Reemplaza `123456789012` con el Account ID real de CloudAltio.

#### Monitorear el Progreso con CLI

```bash
# Opción 1: Esperar automáticamente hasta que complete
aws cloudformation wait stack-create-complete \
  --stack-name cloudaltio-integration \
  --region us-east-1

# Opción 2: Consultar estado manualmente
aws cloudformation describe-stacks \
  --stack-name cloudaltio-integration \
  --region us-east-1 \
  --query 'Stacks[0].StackStatus' \
  --output text

# Opción 3: Ver eventos en tiempo real
aws cloudformation describe-stack-events \
  --stack-name cloudaltio-integration \
  --region us-east-1 \
  --query 'StackEvents[].[Timestamp,ResourceType,ResourceStatus,ResourceStatusReason]' \
  --output table
```

#### Script de Despliegue Completo

```bash
#!/bin/bash
# deploy-cloudaltio.sh

set -e  # Salir si hay errores

STACK_NAME="cloudaltio-integration"
TEMPLATE_FILE="cloudaltio-onboarding-v1.3.json"
CLOUDALTIO_ACCOUNT_ID="123456789012"  # CAMBIAR ESTE VALOR
REGION="us-east-1"

echo "Desplegando CloudAltio Integration Stack..."
echo "Stack Name: $STACK_NAME"
echo "Region: $REGION"
echo "CloudAltio Account ID: $CLOUDALTIO_ACCOUNT_ID"

# Validar template
echo "Validando template..."
aws cloudformation validate-template \
  --template-body file://$TEMPLATE_FILE \
  --region $REGION

# Crear stack
echo "Creando stack..."
STACK_ID=$(aws cloudformation create-stack \
  --stack-name $STACK_NAME \
  --template-body file://$TEMPLATE_FILE \
  --parameters ParameterKey=AccountIdForCloudAltio,ParameterValue=$CLOUDALTIO_ACCOUNT_ID \
  --capabilities CAPABILITY_NAMED_IAM \
  --region $REGION \
  --tags \
    Key=Environment,Value=Production \
    Key=ManagedBy,Value=CloudFormation \
    Key=Purpose,Value=CloudAltio-Integration \
  --query 'StackId' \
  --output text)

echo "Stack ID: $STACK_ID"
echo "Esperando a que el stack se complete..."

# Esperar a que complete
aws cloudformation wait stack-create-complete \
  --stack-name $STACK_NAME \
  --region $REGION

echo "✓ Stack creado exitosamente"

# Obtener outputs
echo ""
echo "Obteniendo Role ARN..."
ROLE_ARN=$(aws cloudformation describe-stacks \
  --stack-name $STACK_NAME \
  --region $REGION \
  --query 'Stacks[0].Outputs[?OutputKey==`RoleArn`].OutputValue' \
  --output text)

echo ""
echo "═══════════════════════════════════════════════════"
echo "  DESPLIEGUE COMPLETADO"
echo "═══════════════════════════════════════════════════"
echo ""
echo "Role ARN (proporcionar a CloudAltio):"
echo "$ROLE_ARN"
echo ""
echo "Bucket S3 creado:"
echo "cloudaltio-$(aws sts get-caller-identity --query Account --output text)"
echo ""
echo "Export FOCUS creado:"
echo "focus-cloudaltio-$(aws sts get-caller-identity --query Account --output text)"
echo ""
echo "═══════════════════════════════════════════════════"
```

Guarda este script como `deploy-cloudaltio.sh`, hazlo ejecutable y ejecútalo:

```bash
chmod +x deploy-cloudaltio.sh
./deploy-cloudaltio.sh
```

## Validación Post-Despliegue

### 1. Verificar Estado del Stack

**Consola**:
1. CloudFormation → Stacks → `cloudaltio-integration`
2. Verifica que el estado sea **CREATE_COMPLETE** (verde)
3. La pestaña "Events" no debe mostrar errores

**CLI**:
```bash
aws cloudformation describe-stacks \
  --stack-name cloudaltio-integration \
  --region us-east-1 \
  --query 'Stacks[0].StackStatus' \
  --output text

# Salida esperada: CREATE_COMPLETE
```

### 2. Obtener el Role ARN

**Consola**:
1. CloudFormation → Stacks → `cloudaltio-integration`
2. Pestaña **"Outputs"**
3. Copia el valor de **RoleArn**

**CLI**:
```bash
aws cloudformation describe-stacks \
  --stack-name cloudaltio-integration \
  --region us-east-1 \
  --query 'Stacks[0].Outputs[?OutputKey==`RoleArn`].OutputValue' \
  --output text

# Salida esperada: arn:aws:iam::987654321098:role/CloudAltioRole
```

**ACCIÓN REQUERIDA**: Proporciona este ARN al equipo de CloudAltio para completar la integración.

### 3. Verificar el Rol IAM

**Consola**:
1. IAM → Roles → Buscar `CloudAltioRole`
2. Verifica:
   - Trust relationships: Debe mostrar el Account ID de CloudAltio
   - Permissions: Debe tener 1 managed policy y 2 inline policies
   - Tags: Debe tener `Env:CloudAltio`

**CLI**:
```bash
# Ver información del rol
aws iam get-role \
  --role-name CloudAltioRole \
  --query 'Role.[RoleName,Arn,Tags]' \
  --output table

# Listar políticas administradas
aws iam list-attached-role-policies \
  --role-name CloudAltioRole

# Salida esperada:
# - ViewOnlyAccess

# Listar políticas inline
aws iam list-role-policies \
  --role-name CloudAltioRole

# Salida esperada:
# - CloudAltioAdditionalResourceReadOnly
# - CloudAltioCURReadOnly
```

### 4. Verificar el Bucket S3

**Obtener nombre del bucket**:
```bash
BUCKET_NAME="cloudaltio-$(aws sts get-caller-identity --query Account --output text)"
echo $BUCKET_NAME
```

**Verificar existencia y configuración**:
```bash
# Verificar que el bucket existe
aws s3 ls | grep cloudaltio

# Ver configuración de bloqueo de acceso público
aws s3api get-public-access-block --bucket $BUCKET_NAME

# Ver lifecycle configuration
aws s3api get-bucket-lifecycle-configuration --bucket $BUCKET_NAME

# Ver tags
aws s3api get-bucket-tagging --bucket $BUCKET_NAME

# Ver bucket policy
aws s3api get-bucket-policy --bucket $BUCKET_NAME --query Policy --output text | jq .
```

**Verificaciones esperadas**:
- ✓ Bucket existe y es accesible
- ✓ Public Access Block está completamente activado
- ✓ Lifecycle policy expira objetos en 200 días
- ✓ Tag `Env:CloudAltio` está presente
- ✓ Bucket policy permite acceso a `billingreports.amazonaws.com` y CloudAltioRole

### 5. Verificar la Exportación FOCUS

**Consola**:
1. Navega a **Billing and Cost Management**
2. En el menú lateral, selecciona **Data Exports**
3. Busca el export: `focus-cloudaltio-{tu-account-id}`
4. Estado esperado: **Active**
5. Verifica que la columna "Last refresh" tenga una fecha (puede tardar 24-48h en la primera ejecución)

**CLI**:
```bash
# Listar exports
aws bcm-data-exports list-exports \
  --region us-east-1 \
  --query 'Exports[?contains(ExportArn, `cloudaltio`)]' \
  --output table

# Ver detalles del export específico
aws bcm-data-exports get-export \
  --export-arn "arn:aws:bcm-data-exports:us-east-1:$(aws sts get-caller-identity --query Account --output text):export/focus-cloudaltio-$(aws sts get-caller-identity --query Account --output text)" \
  --region us-east-1
```

### 6. Verificar Tags en los Recursos

```bash
# Verificar tag en el rol IAM
aws iam list-role-tags --role-name CloudAltioRole

# Verificar tag en el bucket S3
aws s3api get-bucket-tagging \
  --bucket cloudaltio-$(aws sts get-caller-identity --query Account --output text)

# Verificar tag en el export
aws bcm-data-exports list-tags-for-resource \
  --resource-arn "arn:aws:bcm-data-exports:us-east-1:$(aws sts get-caller-identity --query Account --output text):export/focus-cloudaltio-$(aws sts get-caller-identity --query Account --output text)" \
  --region us-east-1
```

Todos los recursos deben mostrar el tag `Env:CloudAltio`.

### 7. Prueba de Conectividad (Opcional - Para Usuarios Avanzados)

Si tienes acceso a la cuenta de CloudAltio, puedes probar asumir el rol:

```bash
# Desde la cuenta de CloudAltio
aws sts assume-role \
  --role-arn "arn:aws:iam::TU_ACCOUNT_ID:role/CloudAltioRole" \
  --role-session-name "CloudAltioTest" \
  --duration-seconds 3600

# Si es exitoso, configurar credenciales temporales y probar acceso
aws s3 ls s3://cloudaltio-TU_ACCOUNT_ID/ --profile cloudaltio-assumed
```

## Configuración en CloudAltio

### Información a Proporcionar

Una vez que el stack esté completamente desplegado (`CREATE_COMPLETE`), proporciona la siguiente información al equipo de CloudAltio:

| Campo | Valor | Dónde Obtenerlo |
|-------|-------|-----------------|
| **Role ARN** | `arn:aws:iam::{account-id}:role/CloudAltioRole` | CloudFormation Outputs |
| **Bucket Name** | `cloudaltio-{account-id}` | S3 Console o CLI |
| **Export Name** | `focus-cloudaltio-{account-id}` | Data Exports Console |
| **AWS Account ID** | Tu Account ID de 12 dígitos | IAM Dashboard o CLI: `aws sts get-caller-identity` |
| **AWS Region** | `us-east-1` | Región donde desplegaste el stack |
| **Export Format** | `FOCUS 1.2 Parquet` | Configurado en el template |

### Plantilla de Email para CloudAltio

```
Asunto: Integración AWS completada - [NOMBRE_EMPRESA]

Hola equipo de CloudAltio,

Hemos completado exitosamente el despliegue de la infraestructura de integración en nuestra cuenta AWS.

Detalles de la integración:

- Role ARN: arn:aws:iam::123456789012:role/CloudAltioRole
- AWS Account ID: 123456789012
- Bucket Name: cloudaltio-123456789012
- Export Name: focus-cloudaltio-123456789012
- Region: us-east-1
- Export Format: FOCUS 1.2 Parquet
- Fecha de despliegue: 13/01/2026

El stack de CloudFormation está en estado CREATE_COMPLETE y todos los recursos han sido validados.

Por favor, procede con la configuración en la plataforma CloudAltio.

Saludos,
[TU NOMBRE]
[EQUIPO FINOPS]
```

### Tiempo de Disponibilidad de Datos

**Primera exportación**: 24-48 horas  
Durante este período inicial:
- AWS Data Exports está procesando el historial de costos
- El bucket S3 puede estar vacío o contener datos parciales
- CloudAltio podrá visualizar datos históricos una vez que la primera exportación complete

**Actualizaciones posteriores**: Diarias  
- Las exportaciones se actualizan automáticamente cada 24 horas
- Típicamente se completan en la madrugada (hora UTC)
- Los datos del día anterior estarán disponibles al día siguiente

## Consideraciones de Seguridad

### Principio de Mínimo Privilegio

El rol CloudAltioRole está diseñado siguiendo las mejores prácticas de seguridad:

1. **Solo lectura**: Ningún permiso de escritura, modificación o eliminación de recursos
2. **Sin acceso a datos sensibles**: ViewOnlyAccess no permite leer contenido de:
   - Objetos en buckets S3
   - Tablas de DynamoDB
   - Código de funciones Lambda
   - Secretos en Secrets Manager
   - Parámetros encriptados en Systems Manager

3. **Trust policy restrictivo**: Solo la cuenta específica de CloudAltio puede asumir el rol
4. **Sin permisos administrativos**: No puede crear, modificar o eliminar recursos de IAM

### Auditoría y Compliance

**CloudTrail**:
Todas las operaciones de AssumeRole quedan registradas en CloudTrail:

```bash
# Ver intentos de asumir el rol
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=ResourceName,AttributeValue=CloudAltioRole \
  --region us-east-1 \
  --max-results 50
```

**S3 Access Logging** (opcional):
Para auditar accesos al bucket de reportes:

```bash
# Crear bucket para logs
aws s3api create-bucket \
  --bucket cloudaltio-access-logs-$(aws sts get-caller-identity --query Account --output text) \
  --region us-east-1

# Habilitar logging en el bucket de reportes
aws s3api put-bucket-logging \
  --bucket cloudaltio-$(aws sts get-caller-identity --query Account --output text) \
  --bucket-logging-status '{
    "LoggingEnabled": {
      "TargetBucket": "cloudaltio-access-logs-'$(aws sts get-caller-identity --query Account --output text)'",
      "TargetPrefix": "access-logs/"
    }
  }'
```

**AWS Config** (opcional):
Monitorear cambios en los recursos creados:

```bash
# Ejemplo: Regla para detectar cambios en el rol
aws configservice put-config-rule \
  --config-rule '{
    "ConfigRuleName": "cloudaltio-role-unchanged",
    "Description": "Detecta cambios en CloudAltioRole",
    "Source": {
      "Owner": "AWS",
      "SourceIdentifier": "IAM_POLICY_NO_STATEMENTS_WITH_ADMIN_ACCESS"
    }
  }'
```

### Seguridad del Bucket S3

El bucket implementa múltiples capas de seguridad:

1. **Bloqueo de acceso público**: Todas las opciones activadas
2. **Bucket policy explícita**: Solo permite acceso a servicios y roles específicos
3. **Encriptación**: AWS usa encriptación server-side por defecto (SSE-S3)
4. **Lifecycle policy**: Elimina datos antiguos automáticamente

Para agregar encriptación adicional con KMS (opcional):

```bash
# Crear KMS key para el bucket
aws kms create-key \
  --description "CloudAltio S3 Bucket Encryption" \
  --key-policy file://kms-policy.json

# Habilitar encriptación con KMS en el bucket
aws s3api put-bucket-encryption \
  --bucket cloudaltio-$(aws sts get-caller-identity --query Account --output text) \
  --server-side-encryption-configuration '{
    "Rules": [{
      "ApplyServerSideEncryptionByDefault": {
        "SSEAlgorithm": "aws:kms",
        "KMSMasterKeyID": "arn:aws:kms:us-east-1:ACCOUNT_ID:key/KEY_ID"
      }
    }]
  }'
```

## Costos Asociados

### Estimación de Costos Mensuales

| Servicio | Componente | Costo Estimado | Detalles |
|----------|-----------|----------------|----------|
| **IAM** | Role y Policies | $0.00 | Sin cargo |
| **S3** | Almacenamiento | ~$3-5/mes | Depende del volumen de datos |
| **S3** | Requests GET | $0.01 - $0.10/mes | CloudAltio lee datos diariamente |
| **S3** | Requests PUT | $0.00 | Mínimo, solo escrituras diarias |
| **Data Exports** | Servicio FOCUS | $0.00 | Sin cargo por el servicio |
| **CloudFormation** | Stack management | $0.00 | Sin cargo |
| **Data Transfer** | Egress desde S3 | $0 - $3/mes | Depende de la ubicación de CloudAltio (us-east-1) |

**Total estimado: $5 - $8/mes**

### Factores que Afectan el Costo

1. **Volumen de uso de AWS**: Más servicios y recursos = más datos en FOCUS = más almacenamiento S3
2. **Frecuencia de acceso**: CloudAltio accede diariamente, costo predecible
3. **Retención de datos**: Con lifecycle de 200 días, controlas el crecimiento del bucket
4. **Región**: us-east-1 tiene precios competitivos

## Solución de Problemas

### Error: "Bucket name already exists"

**Causa**: Los nombres de bucket S3 son globalmente únicos. Alguien más ya tiene un bucket con ese nombre.

**Solución 1** - Modificar el template para agregar un sufijo aleatorio:

```json
"BucketName": {
  "Fn::Sub": "cloudaltio-${AWS::AccountId}-${AWS::StackName}"
}
```

**Solución 2** - Usar un prefijo personalizado (requiere modificar el template):

```json
"BucketName": {
  "Fn::Sub": "mi-empresa-cloudaltio-${AWS::AccountId}"
}
```

### Error: "User is not authorized to perform: bcm-data-exports:CreateExport"

**Causa**: Tu usuario IAM no tiene permisos suficientes para crear Data Exports.

**Solución**: Solicita al administrador de AWS que agregue permisos. Política necesaria:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "bcm-data-exports:*",
        "cur:PutReportDefinition",
        "s3:CreateBucket",
        "s3:PutBucketPolicy",
        "iam:CreateRole",
        "iam:PutRolePolicy",
        "iam:AttachRolePolicy",
        "iam:CreatePolicy"
      ],
      "Resource": "*"
    }
  ]
}
```

### Error: Stack en estado ROLLBACK_COMPLETE

**Causa**: Uno o más recursos fallaron al crearse, CloudFormation revirtió todos los cambios.

**Diagnóstico**:
```bash
# Ver eventos del stack para identificar el error
aws cloudformation describe-stack-events \
  --stack-name cloudaltio-integration \
  --region us-east-1 \
  --query 'StackEvents[?ResourceStatus==`CREATE_FAILED`].[LogicalResourceId,ResourceStatusReason]' \
  --output table
```

**Pasos de recuperación**:

1. Identifica el recurso y la razón del fallo
2. Elimina el stack fallido:
```bash
aws cloudformation delete-stack \
  --stack-name cloudaltio-integration \
  --region us-east-1

# Esperar a que se elimine completamente
aws cloudformation wait stack-delete-complete \
  --stack-name cloudaltio-integration \
  --region us-east-1
```

3. Corrige el problema identificado
4. Reintenta la creación del stack

## Preguntas Frecuentes (FAQ)

### ¿Cuánto tiempo tarda el despliegue completo?

- **Creación del stack**: 3-5 minutos
- **Primera exportación FOCUS**: 24-48 horas
- **Tiempo total hasta datos disponibles**: ~48 horas

### ¿Puedo desplegar esto en múltiples cuentas?

Sí. Si tienes AWS Organizations:
- Despliega en la **cuenta de management** (payer account) para obtener costos consolidados
- O despliega en cuentas individuales para análisis específicos
- CloudAltio puede agregar múltiples cuentas en su plataforma

### ¿Qué datos puede ver CloudAltio exactamente?

CloudAltio puede ver:
- ✓ Metadatos de recursos (nombres, IDs, configuraciones)
- ✓ Datos de facturación y costos detallados
- ✓ Métricas de uso de recursos
- ✓ Información de tags y etiquetado
- ✗ **NO puede ver**: Contenido de bases de datos, objetos S3, secretos, código, logs

### ¿CloudAltio puede modificar mi infraestructura?

No. El rol solo tiene permisos de lectura. CloudAltio:
- ✗ No puede crear, modificar o eliminar recursos
- ✗ No puede cambiar configuraciones
- ✗ No puede ejecutar comandos
- ✓ Solo puede leer información y metadatos

### ¿Qué pasa si cambia el Account ID de CloudAltio?

Actualiza el stack con el nuevo Account ID siguiendo la sección "Actualización del Stack". El cambio tarda ~2 minutos.

### ¿Puedo cambiar el nombre del bucket después de crearlo?

No directamente. Los nombres de bucket S3 no se pueden cambiar. Deberías:
1. Crear un nuevo stack con un nombre diferente
2. Eliminar el stack antiguo
3. Actualizar la configuración en CloudAltio

### ¿El FOCUS export consume datos históricos?

FOCUS export genera datos desde el momento de su creación hacia adelante. Para datos históricos:
- Usa Cost and Usage Report (CUR) legacy con datos desde que se habilitó
- O consulta Cost Explorer API para historial
- Mediante un ticket a soporte AWS, solicita un fullfill de datos en tu FOCUS Export hacia la fecha deseada.

## Eliminación del Stack

### Antes de Eliminar

**ADVERTENCIA**: Esta acción es irreversible y eliminará:
- El rol IAM CloudAltioRole
- El bucket S3 y todos sus contenidos
- La exportación FOCUS configurada
- Todas las políticas asociadas

**Pasos previos**:

1. **Notificar a CloudAltio**: Informa que vas a eliminar la integración
2. **Respaldar datos** (opcional):
```bash
# Descargar todos los reportes FOCUS
aws s3 sync s3://cloudaltio-$(aws sts get-caller-identity --query Account --output text) ./backup-cloudaltio-reports/

# Comprimir el backup
tar -czf cloudaltio-backup-$(date +%Y%m%d).tar.gz ./backup-cloudaltio-reports/
```

### Proceso de Eliminación

#### Paso 1: Vaciar el Bucket S3

CloudFormation no puede eliminar buckets que contienen objetos.

```bash
# Obtener nombre del bucket
BUCKET_NAME="cloudaltio-$(aws sts get-caller-identity --query Account --output text)"

# Ver cuántos objetos hay
aws s3 ls s3://$BUCKET_NAME --recursive --summarize

# Eliminar todos los objetos
aws s3 rm s3://$BUCKET_NAME --recursive

# Verificar que está vacío
aws s3 ls s3://$BUCKET_NAME
```

#### Paso 2: Eliminar el Stack

**CLI**:
```bash
# Eliminar el stack
aws cloudformation delete-stack \
  --stack-name cloudaltio-integration \
  --region us-east-1

# Esperar a que se elimine completamente
aws cloudformation wait stack-delete-complete \
  --stack-name cloudaltio-integration \
  --region us-east-1
```

#### Paso 3: Verificar Limpieza Completa

```bash
# Verificar que el rol fue eliminado
aws iam get-role --role-name CloudAltioRole 2>&1 | grep NoSuchEntity

# Verificar que el bucket fue eliminado
aws s3 ls | grep cloudaltio
```

## Recursos Adicionales

### Documentación Oficial de AWS

- **CloudFormation User Guide**: https://docs.aws.amazon.com/cloudformation/
- **AWS Data Exports**: https://docs.aws.amazon.com/cur/latest/userguide/what-is-data-exports.html
- **FOCUS Specification**: https://docs.aws.amazon.com/cur/latest/userguide/table-dictionary-focus.html
- **IAM Roles**: https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles.html
- **S3 Security Best Practices**: https://docs.aws.amazon.com/AmazonS3/latest/userguide/security-best-practices.html

### FinOps Foundation

- **FOCUS Specification**: https://focus.finops.org/
- **FinOps Framework**: https://www.finops.org/framework/

---

**Documento creado por**: FinOps by Orion  
**Última actualización**: 13/01/2026  
**Versión del template**: 1.3  
**Región recomendada**: us-east-1
