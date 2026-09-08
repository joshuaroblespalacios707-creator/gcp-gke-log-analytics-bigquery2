**🇲🇽 Español** | [🇬🇧 English](#-monitoring-log-analytics--cybersecurity-on-google-cloud)

# 📊 Monitoreo, Log Analytics y Ciberseguridad en Google Cloud

Documentación técnica de dos laboratorios complementarios: observabilidad de microservicios en GKE mediante Log Analytics y BigQuery, y auditoría de seguridad con Cloud Audit Logs para trazabilidad forense.

---

## 📌 Módulo 1: Infraestructura GKE, Log Sinks y Análisis de Latencia con SQL

### 🎯 Objetivo General

Desplegar una arquitectura de microservicios en **Google Kubernetes Engine (GKE)**, estructurar un almacenamiento de logs avanzado mediante un **Log Bucket** con soporte de **Log Analytics**, y vincularlo a **BigQuery** para ejecutar consultas SQL de rendimiento y latencia.

---

### 🏗️ Componentes de Infraestructura

| Componente | Detalle |
|---|---|
| Clúster GKE | Entorno donde se ejecuta la aplicación de microservicios "Online Boutique" |
| Log Bucket personalizado | `day2ops-log`, configurado con Log Analytics activado |
| Log Sink | Tubería `day2ops-sink`, para filtrar y enrutar registros específicos |
| BigQuery Dataset | Conjunto de datos enlazado para persistencia y análisis multivariable a largo plazo |

---

### ⚙️ Pasos de Implementación y Código

**Paso 1 — Creación del Log Bucket Personalizado**

Se creó un bucket de logs con retención personalizada y soporte para consultas SQL directas:

```bash
gcloud logging buckets create day2ops-log \
    --location=global \
    --enable-analytics \
    --retention-days=30 \
    --description="Bucket personalizado para logs de GKE y analisis con Log Analytics"
```

**Paso 2 — Configuración del Log Sink**

Se configuró un enrutador de logs para capturar únicamente la telemetría proveniente de los contenedores de Kubernetes (`k8s_container`) y enviarla al bucket recién creado:

```bash
gcloud logging sinks create day2ops-sink \
    logging.googleapis.com/projects/$PROJECT_ID/locations/global/buckets/day2ops-log \
    --log-filter="resource.type=\"k8s_container\""
```

**Paso 3 — Vinculación con BigQuery**

Se creó un enlace para exponer los datos del Log Bucket hacia BigQuery, sin necesidad de duplicar el almacenamiento:

```bash
gcloud logging links create day2ops-link \
    --bucket=day2ops-log \
    --location=global \
    --dataset=day2ops_logs_dataset
```

**Paso 4 — Análisis de Latencia mediante SQL (Log Analytics)**

Con la integración lista, se ejecutó una consulta SQL para analizar los tiempos de respuesta y detectar cuellos de botella en los microservicios:

```sql
SELECT
  timestamp,
  resource.labels.pod_name AS pod_origen,
  httpRequest.requestUrl AS url_solicitada,
  httpRequest.status AS codigo_respuesta,
  CAST(JSON_VALUE(jsonPayload.duration) AS FLOAT64) AS latencia_segundos
FROM
  `day2ops_logs_dataset._AllLogs`
WHERE
  resource.type = "k8s_container"
  AND httpRequest.status IS NOT NULL
ORDER BY
  latencia_segundos DESC
LIMIT 20;
```

---

### 🔧 Errores Diagnosticados y Solución

- **Error:** al intentar consultar los datos en BigQuery o Log Analytics inmediatamente después de crear el sink, la consulta SQL devolvía 0 resultados.
- **Causa:** retraso (*propagation delay*) en la ingesta inicial de datos y en la creación del dataset enlazado.
- **Solución:** se generó tráfico sintético en la aplicación web para forzar la emisión de peticiones HTTP en los pods, y se esperó un lapso de 3 a 5 minutos hasta que los primeros bloques de registros fueron indexados en el bucket.

---

## 📌 Módulo 2: Ciberseguridad, Audit Logs y Acceso a Datos

### 🎯 Objetivo General

Habilitar y analizar la auditoría de seguridad en la infraestructura usando **Cloud Audit Logs** (*Admin Activity* y *Data Access*), rastreando las acciones ejecutadas por usuarios y cuentas de servicio para garantizar trazabilidad e investigación forense.

---

### 📋 Tipos de Logs Configurados

| Tipo | Descripción |
|---|---|
| **Admin Activity Logs** | Registran eventos administrativos de modificación o eliminación de recursos (activos por defecto) |
| **Data Access Logs** | Registran lecturas (`ADMIN_READ`, `DATA_READ`) y escrituras (`DATA_WRITE`) dentro de los servicios. Desactivados por defecto por volumen y costo |

---

### ⚙️ Pasos de Implementación y Código

**Paso 1 — Activación de Data Access Logs**

Se habilitaron los registros de acceso a datos para el servicio de Cloud Storage desde la consola, en **IAM y administración → Logs de auditoría**, seleccionando los tipos `Admin Read`, `Data Read` y `Data Write`.

**Paso 2 — Generación de Eventos de Auditoría desde Cloud Shell**

Se ejecutaron operaciones de infraestructura y manipulación de datos para generar entradas en la bitácora de auditoría:

```bash
# Definir ID de proyecto
export PROJECT_ID=$(gcloud config get-value project)

# 1. Crear bucket (Admin Activity)
gcloud storage buckets create gs://$PROJECT_ID --location=us-central1

# 2. Crear y subir archivo (Data Write)
echo "Hola mundo, auditando logs en GCP" > sample.txt
gcloud storage cp sample.txt gs://$PROJECT_ID

# 3. Leer archivo (Data Read)
gcloud storage ls gs://$PROJECT_ID

# 4. Crear red VPC y máquina virtual (Admin Activity)
gcloud compute networks create mynetwork --subnet-mode=auto
gcloud compute instances create default-us-vm \
    --zone=us-central1-a \
    --network=mynetwork \
    --machine-type=e2-medium

# 5. Crear tema Pub/Sub de prueba rápida (Admin Activity)
gcloud pubsub topics create audit-test-topic

# 6. Eliminar el bucket (Admin Activity - Destrucción)
gcloud storage rm -r gs://$PROJECT_ID
```

**Paso 3 — Extracción y Verificación Forense por CLI**

Se extrajeron los registros de auditoría mediante la herramienta de línea de comandos, para validar los campos críticos de ciberseguridad:

```bash
gcloud logging read "logName=~\"cloudaudit.googleapis.com\"" \
    --limit=5 \
    --format="yaml(timestamp, protoPayload.methodName, protoPayload.authenticationInfo.principalEmail)"
```

---

### 🔧 Errores Diagnosticados y Solución

**Error 1 — Cero resultados en la interfaz gráfica del Logs Explorer**
- **Causa:** la consola web tenía seleccionada una ventana de tiempo demasiado corta (últimos 15 segundos), o la búsqueda estaba acotada dentro de un Log View específico (`day2ops-log`) en lugar de apuntar a todo el proyecto.
- **Solución:** se cambió el ámbito del visor a **Registros del proyecto** y la ventana temporal a **Última 1 hora**.

**Error 2 — Descalce de contexto entre Cloud Shell y la Consola**
- **Causa:** la terminal ejecutaba comandos en un ID de proyecto diferente al seleccionado en la barra superior de la consola web.
- **Solución:** se forzó el contexto en la terminal ejecutando `gcloud config set project <PROJECT_ID>` antes de emitir las llamadas a la API.

---

### ✅ Resultados de la Verificación de Auditoría

La ejecución de extracción por CLI confirmó la captura exitosa del rastro de auditoría:

| Timestamp | Método ejecutado (`methodName`) | Usuario responsable (`principalEmail`) |
|---|---|---|
| 2026-09-08T21:34:19Z | `google.pubsub.v1.Publisher.CreateTopic` | `user@example.com` |
| 2026-09-08T21:28:51Z | `storage.buckets.delete` | `user@example.com` |
| 2026-09-08T21:28:48Z | `v1.compute.instances.insert` | `user@example.com` |
| 2026-09-08T21:28:38Z | `iam.serviceAccounts.actAs` | `user@example.com` |

---

### 🧹 Limpieza de Recursos y Control de Costos

Para evitar cobros recurrentes en la cuenta personal, se procedió con el desmantelamiento de los recursos creados:

```bash
# Eliminar instancia VM y red VPC
gcloud compute instances delete default-us-vm --zone=us-central1-a --quiet
gcloud compute networks delete mynetwork --quiet

# Eliminar tema de prueba Pub/Sub
gcloud pubsub topics delete audit-test-topic --quiet
```

---

## 🎓 Conceptos Aprendidos

- Configuración de Log Buckets personalizados con retención y Log Analytics habilitado
- Enrutamiento selectivo de logs mediante Log Sinks filtrados por tipo de recurso
- Vinculación de logs a BigQuery sin duplicar almacenamiento (Log Analytics linked datasets)
- Análisis de latencia y rendimiento de microservicios mediante consultas SQL sobre logs de Kubernetes
- Diferencia entre Admin Activity Logs y Data Access Logs, y cuándo habilitar cada uno
- Extracción forense de eventos de auditoría por CLI, validando trazabilidad de usuario y método ejecutado
- Diagnóstico de discrepancias de contexto de proyecto entre Cloud Shell y la consola web

---

## 📌 Notas

Este laboratorio forma parte de mi preparación continua hacia la certificación **Google Professional Cloud Security Engineer**, con enfoque práctico en observabilidad de microservicios, análisis de logs a escala y auditoría de seguridad forense en GCP.

---
---

[⬆ Español version above](#-monitoreo-log-analytics-y-ciberseguridad-en-google-cloud)

# 📊 Monitoring, Log Analytics & Cybersecurity on Google Cloud

Technical documentation of two complementary labs: GKE microservices observability via Log Analytics and BigQuery, and security auditing with Cloud Audit Logs for forensic traceability.

---

## 📌 Module 1: GKE Infrastructure, Log Sinks & SQL Latency Analysis

### 🎯 General Objective

Deploy a microservices architecture on **Google Kubernetes Engine (GKE)**, structure an advanced log storage layer via a **Log Bucket** with **Log Analytics** support, and link it to **BigQuery** to run SQL queries for performance and latency analysis.

---

### 🏗️ Infrastructure Components

| Component | Detail |
|---|---|
| GKE Cluster | Environment running the "Online Boutique" microservices application |
| Custom Log Bucket | `day2ops-log`, configured with Log Analytics enabled |
| Log Sink | `day2ops-sink` pipeline, filtering and routing specific log entries |
| BigQuery Dataset | Linked dataset for long-term persistence and multivariable analysis |

---

### ⚙️ Implementation Steps & Code

**Step 1 — Creating the Custom Log Bucket**

Created a log bucket with custom retention and support for direct SQL queries:

```bash
gcloud logging buckets create day2ops-log \
    --location=global \
    --enable-analytics \
    --retention-days=30 \
    --description="Bucket personalizado para logs de GKE y analisis con Log Analytics"
```

**Step 2 — Log Sink Configuration**

Configured a log router to capture only telemetry originating from Kubernetes containers (`k8s_container`) and forward it to the newly created bucket:

```bash
gcloud logging sinks create day2ops-sink \
    logging.googleapis.com/projects/$PROJECT_ID/locations/global/buckets/day2ops-log \
    --log-filter="resource.type=\"k8s_container\""
```

**Step 3 — BigQuery Linking**

Created a link to expose the Log Bucket's data to BigQuery, without duplicating storage:

```bash
gcloud logging links create day2ops-link \
    --bucket=day2ops-log \
    --location=global \
    --dataset=day2ops_logs_dataset
```

**Step 4 — Latency Analysis via SQL (Log Analytics)**

With the integration in place, executed a SQL query to analyze response times and detect bottlenecks across microservices:

```sql
SELECT
  timestamp,
  resource.labels.pod_name AS pod_origen,
  httpRequest.requestUrl AS url_solicitada,
  httpRequest.status AS codigo_respuesta,
  CAST(JSON_VALUE(jsonPayload.duration) AS FLOAT64) AS latencia_segundos
FROM
  `day2ops_logs_dataset._AllLogs`
WHERE
  resource.type = "k8s_container"
  AND httpRequest.status IS NOT NULL
ORDER BY
  latencia_segundos DESC
LIMIT 20;
```

---

### 🔧 Diagnosed Issues & Resolution

- **Issue:** querying the data in BigQuery or Log Analytics immediately after creating the sink returned 0 results.
- **Root cause:** propagation delay in the initial data ingestion and linked dataset creation.
- **Resolution:** generated synthetic traffic on the web application to force HTTP requests across the pods, and waited 3 to 5 minutes until the first batches of log records were indexed in the bucket.

---

## 📌 Module 2: Cybersecurity, Audit Logs & Data Access

### 🎯 General Objective

Enable and analyze security auditing across the infrastructure using **Cloud Audit Logs** (*Admin Activity* and *Data Access*), tracking actions performed by users and service accounts to ensure traceability and forensic investigation capability.

---

### 📋 Log Types Configured

| Type | Description |
|---|---|
| **Admin Activity Logs** | Record administrative events modifying or deleting resources (enabled by default) |
| **Data Access Logs** | Record reads (`ADMIN_READ`, `DATA_READ`) and writes (`DATA_WRITE`) within services. Disabled by default due to volume and cost |

---

### ⚙️ Implementation Steps & Code

**Step 1 — Enabling Data Access Logs**

Enabled data access logging for the Cloud Storage service via the console, under **IAM & Admin → Audit Logs**, selecting `Admin Read`, `Data Read`, and `Data Write` types.

**Step 2 — Generating Audit Events from Cloud Shell**

Executed infrastructure and data manipulation operations to generate entries in the audit log:

```bash
# Set project ID
export PROJECT_ID=$(gcloud config get-value project)

# 1. Create bucket (Admin Activity)
gcloud storage buckets create gs://$PROJECT_ID --location=us-central1

# 2. Create and upload a file (Data Write)
echo "Hola mundo, auditando logs en GCP" > sample.txt
gcloud storage cp sample.txt gs://$PROJECT_ID

# 3. Read file (Data Read)
gcloud storage ls gs://$PROJECT_ID

# 4. Create VPC network and virtual machine (Admin Activity)
gcloud compute networks create mynetwork --subnet-mode=auto
gcloud compute instances create default-us-vm \
    --zone=us-central1-a \
    --network=mynetwork \
    --machine-type=e2-medium

# 5. Create a quick test Pub/Sub topic (Admin Activity)
gcloud pubsub topics create audit-test-topic

# 6. Delete the bucket (Admin Activity - Destruction)
gcloud storage rm -r gs://$PROJECT_ID
```

**Step 3 — Forensic Extraction & Verification via CLI**

Extracted audit records using the command-line tool, to validate critical cybersecurity fields:

```bash
gcloud logging read "logName=~\"cloudaudit.googleapis.com\"" \
    --limit=5 \
    --format="yaml(timestamp, protoPayload.methodName, protoPayload.authenticationInfo.principalEmail)"
```

---

### 🔧 Diagnosed Issues & Resolution

**Issue 1 — Zero results in the Logs Explorer web UI**
- **Root cause:** the web console had an overly narrow time window selected (last 15 seconds), or the search was scoped to a specific Log View (`day2ops-log`) instead of the entire project.
- **Resolution:** changed the viewer's scope to **Project logs** and the time window to **Last 1 hour**.

**Issue 2 — Context mismatch between Cloud Shell and the Console**
- **Root cause:** the terminal was running commands against a different project ID than the one selected in the web console's top bar.
- **Resolution:** forced the correct context in the terminal by running `gcloud config set project <PROJECT_ID>` before issuing API calls.

---

### ✅ Audit Verification Results

The CLI extraction confirmed successful capture of the audit trail:

| Timestamp | Method executed (`methodName`) | Responsible user (`principalEmail`) |
|---|---|---|
| 2026-09-08T21:34:19Z | `google.pubsub.v1.Publisher.CreateTopic` | `user@example.com` |
| 2026-09-08T21:28:51Z | `storage.buckets.delete` | `user@example.com` |
| 2026-09-08T21:28:48Z | `v1.compute.instances.insert` | `user@example.com` |
| 2026-09-08T21:28:38Z | `iam.serviceAccounts.actAs` | `user@example.com` |

---

### 🧹 Resource Cleanup & Cost Control

To prevent recurring charges on the personal account, all created resources were decommissioned:

```bash
# Delete VM instance and VPC network
gcloud compute instances delete default-us-vm --zone=us-central1-a --quiet
gcloud compute networks delete mynetwork --quiet

# Delete the Pub/Sub test topic
gcloud pubsub topics delete audit-test-topic --quiet
```

---

## 🎓 Key Concepts Learned

- Configuring custom Log Buckets with retention and Log Analytics enabled
- Selective log routing via Log Sinks filtered by resource type
- Linking logs to BigQuery without duplicating storage (Log Analytics linked datasets)
- Analyzing microservice latency and performance via SQL queries over Kubernetes logs
- The difference between Admin Activity Logs and Data Access Logs, and when to enable each
- Forensic extraction of audit events via CLI, validating user traceability and executed method
- Diagnosing project context mismatches between Cloud Shell and the web console

---

## 📌 Notes

This lab is part of my ongoing preparation for the **Google Professional Cloud Security Engineer** certification, with a hands-on focus on microservices observability, large-scale log analysis, and forensic security auditing in GCP.
