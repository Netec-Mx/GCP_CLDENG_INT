---
layout: lab
title: "Práctica 12: Analizar logs de error de app en GKE y exportarlos a BigQuery"
permalink: /lab12/lab12/
images_base: /labs/lab12/img
duration: "30 minutos"
objective:
  - "Crear un clúster **GKE Autopilot**, desplegar una app que genera errores, **analizar logs** en **Cloud Logging (Logs Explorer)** y **exportar** esos logs a **BigQuery** mediante **Log Router (sink)**, verificando la llegada de eventos con consultas en BigQuery."
prerequisites:
  - "Proyecto de Google Cloud con **facturación habilitada**."
  - "Permisos para: **Kubernetes Engine Admin**, **Logging Admin** (o Logs Configuration Writer), **BigQuery Admin** (o Data Editor para datasets) y acceso a **Cloud Shell**."
  - "Acceso a **Google Cloud Console** y a **Cloud Shell (gcloud/bq)**."
introduction: |
  En operación real, los errores de una aplicación en Kubernetes se investigan en **Cloud Logging** usando filtros por clúster, namespace, pod y texto del mensaje. Cuando necesitas analítica avanzada, correlación, retención extendida o integraciones con BI, puedes **exportar logs** a destinos como **BigQuery** con **Log Router** (sinks).  
  
  En esta práctica desplegarás un “generador de errores” en GKE, aprenderás a filtrar y entender los logs en Logs Explorer, y luego exportarás esos logs a BigQuery para consultarlos con SQL.
slug: lab12
lab_number: 12
final_result: >
  Al finalizar, tendrás un clúster GKE Autopilot con un namespace `lab12` y una app que genera logs de error, una consulta guardada en Logs Explorer, un sink de Log Router exportando a BigQuery, y evidencias de consulta en BigQuery mostrando los eventos exportados.
notes:
  - "**Costos:** GKE Autopilot cobra por recursos de los Pods (CPU/Mem/Storage), BigQuery cobra por consultas (por bytes procesados) y almacenamiento. Logging/export puede generar costos según volumen/retención. Elimina recursos al finalizar."
  - "Para laboratorio, usa filtros por `textPayload` (ej. contiene `ERROR lab12`) para asegurar coincidencias sin depender de mapeo de severidad."
  - "Log Router requiere permisos en el dataset (BigQuery) para su **service account**. Si no lo concedes, el export fallará."
references:
  - text: "GKE Autopilot: Overview"
    url: https://cloud.google.com/kubernetes-engine/docs/concepts/autopilot-overview
  - text: "Cloud Logging: Logs Explorer"
    url: https://cloud.google.com/logging/docs/view/logs-explorer-interface
  - text: "Log Router: Exportar logs (sinks)"
    url: https://cloud.google.com/logging/docs/export
  - text: "Exportar logs a BigQuery"
    url: https://cloud.google.com/logging/docs/export/bigquery
  - text: "BigQuery: Datasets"
    url: https://cloud.google.com/bigquery/docs/datasets-intro
  - text: "GKE Pricing"
    url: https://cloud.google.com/kubernetes-engine/pricing
  - text: "BigQuery Pricing"
    url: https://cloud.google.com/bigquery/pricing
prev: /lab11/lab11/
next: /lab13/lab13/
---

---

### Tarea 1. Preparar el entorno y variables del laboratorio

> **Tiempo estimado:** 4 minutos
{: .lab-note .info .compact}

En esta tarea abrirás Cloud Shell, validarás el proyecto activo y crearás una carpeta independiente para el laboratorio con variables estándar (región, clúster, dataset y sink).

#### Tarea 1.1

- {% include step_label.html %} Abre el navegador **Google Chrome** para autenticarte a **Google Cloud Console** --> [**AQUÍ**](https://cloud.google.com/cloud-console).

  > **Nota:** Utiliza el **Usuario** y **Contraseña** asignados al curso para iniciar sesion en la cuenta de **GCP Console**
  {: .lab-note .info .compact}

  {% include step_image.html %}

- {% include step_label.html %} Una vez autenticado activa **Cloud Shell** (ícono `>_` en la esquina superior derecha).

  {% include step_image.html %}

- {% include step_label.html %} Si aparece la ventana emergente **Autoriza Cloud Shell** da clic en el botón **Autorizar**

  {% include step_image.html %}

- {% include step_label.html %} Verifica el **proyecto activo** que se te asigno en el curso dentro de Cloud Shell.

  > **Nota:** Si sale vacío o incorrecto, configúralo en el siguiente paso. El **ID/NOMBRE del proyecto** es diferente para cada usuario.
  {: .lab-note .info .compact}

  ```bash
  gcloud projects list
  ```
  {% include step_image.html %}

- {% include step_label.html %} Configura el proyecto sustituye la variable **TU_PROJECT_ID** con la de ru proyecto.

  ```bash
  gcloud config set project TU_PROJECT_ID
  ```
  {% include step_image.html %}

- {% include step_label.html %} Crea la carpeta del laboratorio 12 y subcarpetas estándar.

  > **NOTA:** Cada práctica en su carpeta (manifests/scripts/outputs) mantiene el trabajo ordenado y reproducible.
  {: .lab-note .info .compact}

  ```bash
  cd ~
  mkdir -p labs-gcp-engineer/lab12/{manifests,scripts,outputs}
  ```
  ```bash
  cd labs-gcp-engineer/lab12
  ```
  {% include step_image.html %}

- {% include step_label.html %} Crea el archivo `scripts/env.sh` con variables del laboratorio y cárgalas.

  > **NOTA:** Ajusta región/zona si tu organización lo requiere.
  {: .lab-note .info .compact}

  > **NOTA:** Variables consistentes hacen que los pasos sean repetibles y evitan errores de nombres.
  {: .lab-note .info .compact}

  ```bash
  cat > scripts/env.sh <<'EOF'
  export REGION="us-central1"
  export ZONE="us-central1-a"

  export CLUSTER_NAME="lab12-gke"
  export NAMESPACE="lab12"

  export DATASET="lab12_logs"
  export SINK_NAME="lab12-sink-gke-errors"

  # Texto “ancla” para filtrar/exportar sin depender de severidad
  export ERROR_ANCHOR="ERROR lab12"
  EOF
  ```
  ```bash
  source scripts/env.sh
  echo "REGION=$REGION ZONE=$ZONE CLUSTER_NAME=$CLUSTER_NAME NAMESPACE=$NAMESPACE"
  echo "DATASET=$DATASET SINK_NAME=$SINK_NAME ERROR_ANCHOR=$ERROR_ANCHOR"
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica estructura y guarda evidencia.

  > **NOTA:** Evidencia básica de preparación y organización del laboratorio.
  {: .lab-note .info .compact}

  ```bash
  pwd | tee outputs/pwd.txt
  ```
  ```bash
  ls -la | tee outputs/ls_root.txt
  ```
  ```bash
  ls -la manifests scripts outputs | tee outputs/ls_subfolders.txt
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[0] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 2. Crear clúster GKE Autopilot y desplegar app que genera errores

> **Tiempo estimado:** 12 minutos
{: .lab-note .info .compact}

En esta tarea crearás un clúster GKE Autopilot (rápido para laboratorio), conectarás `kubectl` y desplegarás una app que emite mensajes de error periódicos. Después validarás que los pods están corriendo.

#### Tarea 2.1

- {% include step_label.html %} En la consola, abre **Kubernetes Engine** luego **Clusters**.

  {% include step_image.html %}

- {% include step_label.html %} Haz clic en **Create**.

  {% include step_image.html %}

- {% include step_label.html %} Da clic en **Configure** de la sección **Autopilot: Google manages your cluster (Recommended)**.

  > **NOTA:** Autopilot simplifica el clúster y acelera el despliegue para prácticas.
  {: .lab-note .info .compact}

  {% include step_image.html %}

- {% include step_label.html %} Configura los siguientes datos.

  > **NOTA:** Necesitas Logging habilitado para ver logs de contenedores en Cloud Logging.
  {: .lab-note .important .compact}

  - Name: `lab12-gke`
  - Region: **us-central1**

  {% include step_image.html %}

  - Networking: marca **Enable Observability (Dataplane V2)**

  {% include step_image.html %}

- {% include step_label.html %} Luego clic en **Create**. 

- {% include step_label.html %} Mientras se crea el clúster, en Cloud Shell verifica/habilita APIs necesarias (si tu consola no las habilita automáticamente).

  > **NOTA:** Sin estas APIs podrías fallar al crear clúster o configurar export a BigQuery.
  {: .lab-note .important .compact}

  ```bash
  source scripts/env.sh
  gcloud services enable container.googleapis.com logging.googleapis.com bigquery.googleapis.com
  ```
  {% include step_image.html %}

- {% include step_label.html %} Cuando el clúster esté listo (STATUS: RUNNING), conecta `kubectl` desde la UI con **Connect**.

  {% include step_image.html %}

- {% include step_label.html %} Ahora da clic **Run in Cloud Shell**.

  > **NOTA:** La UI genera el comando `gcloud container clusters get-credentials` exacto para tu clúster.
  {: .lab-note .info .compact}

  {% include step_image.html %}
  {% include step_image.html %}

- {% include step_label.html %} Verifica el contexto de `kubectl` y lista nodos.

  > **NOTA:** Confirmas conectividad y salud básica del clúster antes de desplegar workloads.
  {: .lab-note .info .compact}

  ```bash
  kubectl config current-context | tee outputs/kubectl_context.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl get nodes -o wide | tee outputs/nodes.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Crea el namespace `lab12`.

  > **NOTA:** Separar por namespace facilita filtrar logs y limpiar recursos después.
  {: .lab-note .info .compact}

  ```bash
  source scripts/env.sh
  kubectl create ns "$NAMESPACE" 2>/dev/null || true
  ```
  {% include step_image.html %}
  ```bash
  kubectl get ns "$NAMESPACE" -o wide | tee outputs/namespace.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Crea el manifiesto del “generador de errores” (logs a stderr) en `manifests/errgen.yaml`.

  > **NOTA:** Es una forma simple y controlada de producir logs repetibles para análisis y exportación.
  {: .lab-note .info .compact}

  ```bash
  source scripts/env.sh
  cat > manifests/errgen.yaml <<EOF
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: errgen
    namespace: ${NAMESPACE}
    labels:
      app: errgen
  spec:
    replicas: 1
    selector:
      matchLabels:
        app: errgen
    template:
      metadata:
        labels:
          app: errgen
      spec:
        containers:
        - name: errgen
          image: bash:5.2
          command: ["/usr/local/bin/bash","-lc"]
          args:
            - |
              i=0
              while true; do
                i=$((i+1))
                echo "${ERROR_ANCHOR} - simulated error #$i - ts=$(date -u +%Y-%m-%dT%H:%M:%SZ)" 1>&2
                sleep 5
              done
          resources:
            requests:
              cpu: "50m"
              memory: "64Mi"
            limits:
              cpu: "200m"
              memory: "128Mi"
  EOF
  ```

- {% include step_label.html %} Aplica el manifiesto y verifica rollout/pod.

  > **NOTA:** Antes de analizar logs, asegúrate de que el contenedor esté corriendo sin crash loops.
  {: .lab-note .important .compact}

  ```bash
  source scripts/env.sh
  kubectl apply -f manifests/errgen.yaml
  ```
  {% include step_image.html %}
  ```bash
  kubectl -n "$NAMESPACE" rollout status deploy/errgen | tee outputs/rollout_errgen.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl -n "$NAMESPACE" get pods -l app=errgen -o wide | tee outputs/pods_errgen.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica que el pod realmente está generando logs (preview rápido con `kubectl logs`).

  > **NOTA:** Si ves líneas con `ERROR lab12`, entonces Cloud Logging también debería empezar a recibirlas.
  {: .lab-note .info .compact}

  ```bash
  source scripts/env.sh
  POD="$(kubectl -n "$NAMESPACE" get pods -l app=errgen -o jsonpath='{.items[0].metadata.name}')"
  echo "POD=$POD" | tee outputs/pod_name.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl -n "$NAMESPACE" logs "$POD" --tail=20 | tee outputs/kubectl_logs_tail.txt
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[1] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 3. Analizar logs de error en Cloud Logging (Logs Explorer) y guardar consulta

> **Tiempo estimado:** 7 minutos
{: .lab-note .info .compact}

En esta tarea usarás Logs Explorer para filtrar logs por clúster y namespace, buscar el texto “ancla” del error, inspeccionar campos útiles y guardar una consulta para uso operativo.

#### Tarea 3.1

- {% include step_label.html %} En la consola, abre **Monitoring** luego **Logs Explorer**.

  > **NOTA:** Logs Explorer es el lugar principal para investigar errores y correlacionar eventos.
  {: .lab-note .info .compact}

  {% include step_image.html %}

- {% include step_label.html %} Construye el filtro base (copia y pega) y ejecútalo.

  > **NOTA:** Filtrar por clúster + namespace reduce ruido y te deja solo lo relevante del laboratorio.
  {: .lab-note .info .compact}

  ```bash
  resource.type="k8s_container"
  resource.labels.cluster_name="lab12-gke"
  resource.labels.namespace_name="lab12"
  textPayload:"ERROR lab12"
  ```

  {% include step_image.html %}

- {% include step_label.html %} En un log entry, expande y revisa campos clave:

  > **NOTA:** En escenarios reales, estos campos te permiten identificar qué pod/versión está fallando.
  {: .lab-note .info .compact}

  - `timestamp`
  - `resource.labels.pod_name`
  - `resource.labels.container_name`
  - `textPayload`
  - `logName`

  {% include step_image.html %}

- {% include step_label.html %} Cambia el rango de tiempo a **Last 30 minutes** y confirma que hay eventos continuos (cada ~5s).

  > **NOTA:** El patrón temporal ayuda a confirmar si el error es constante o intermitente.
  {: .lab-note .info .compact}

  {% include step_image.html %}

- {% include step_label.html %} Clic en el icono de **Save**:

  {% include step_image.html %}

- {% include step_label.html %} Define los siguientes datos para el guardado.

  > **NOTA:** Guardar consultas reduce tiempo de diagnóstico en incidentes reales.
  {: .lab-note .info .compact}

  - Name: `LAB12 - GKE Errors (errgen)`
  - Descripción: `Errores simulados para export a BigQuery`

  {% include step_image.html %}

- {% include step_label.html %} Clic en el botón **Save Query**:

- {% include step_label.html %} Evidencia por Cloud Shell: consulta los últimos logs con `gcloud logging read` usando un filtro equivalente.

  > **NOTA:** Validación rápida por CLI (útil cuando no tienes UI, o para automatizar diagnósticos).
  {: .lab-note .info .compact}

  ```bash
  source scripts/env.sh

  FILTER='resource.type="k8s_container" AND resource.labels.cluster_name="'"$CLUSTER_NAME"'" AND resource.labels.namespace_name="'"$NAMESPACE"'" AND textPayload:"'"$ERROR_ANCHOR"'"'

  gcloud logging read "$FILTER" \
    --limit=10 \
    --format='table(timestamp,resource.labels.pod_name,textPayload)'
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[2] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 4. Exportar logs de error a BigQuery con Log Router

> **Tiempo estimado:** 6 minutos
{: .lab-note .info .compact}

En esta tarea crearás un dataset en BigQuery, crearás un **sink** en Log Router con un filtro para los errores del laboratorio y darás permisos al service account del sink para escribir en BigQuery. Luego validarás que se crean tablas y que llegan eventos.

#### Tarea 4.1

- {% include step_label.html %} En la consola, abre **BigQuery** y crea un dataset:

  {% include step_image.html %}

- {% include step_label.html %} Da clic en el siguiente botón para crear el **dataset**.

  {% include step_image.html %}

- {% include step_label.html %} Ahora crea un **dataset** con los siguientes datos

  > **NOTA:** Log Router necesita un dataset destino existente para exportar.
  {: .lab-note .important .compact}

  - Dataset ID: `lab12_logs`
  - Location type: **us-central1 (Iowa)**

  {% include step_image.html %}

- {% include step_label.html %} Clic en **Create dataset**

- {% include step_label.html %} Verifica por Cloud Shell que el dataset existe.

  > **NOTA:** Evidencia objetiva para confirmar el dataset antes de crear el sink.
  {: .lab-note .info .compact}

  ```bash
  source scripts/env.sh
  bq show --format=prettyjson "${PROJECT_ID}:${DATASET}" | head -n 40 | tee outputs/bq_dataset_show.json
  ```
  {% include step_image.html %}

- {% include step_label.html %} Da clic en **Monitoring** y luego en **Log Router**.

  {% include step_image.html %}

- {% include step_label.html %} Ahora clic en **Create sink**.

  {% include step_image.html %}

- {% include step_label.html %} Configura los siguientes datos:

  > **NOTA:** El filtro reduce el volumen exportado (buena práctica para costos y relevancia).
  {: .lab-note .warning .compact}

  - Sink name: `lab12-sink-gke-errors`
  - Clic **Next**

  {% include step_image.html %}

  - Sink destination: **BigQuery dataset**
  - Destination: **lab12_logs**

  {% include step_image.html %}

- {% include step_label.html %} Ahora en la opcion **Build inclusion filter** de la sección **Choose logs to include in sink** agrega el siguiente codigo del filtro.

  ```bash
  resource.type="k8s_container"
  resource.labels.cluster_name="lab12-gke"
  resource.labels.namespace_name="lab12"
  textPayload:"ERROR lab12"
  ```
  {% include step_image.html %}

- {% include step_label.html %} Clic en **Create sink**

- {% include step_label.html %} Da clic nuevamente en la opción **Log Router**.

- {% include step_label.html %} Ahora en la tabla inferior da clic en los 3 puntos de tu **BigQuery dataset** y en el menu deplegable clic en **View sink details**.

  {% include step_image.html %}

- {% include step_label.html %} Identifica el **Writer identity** (service account) mostrado por la UI y copia su valor.

  > **NOTA:** Ese service account es el que escribirá en BigQuery; sin permisos, no habrá export.
  {: .lab-note .important .compact}

  {% include step_image.html %}

- {% include step_label.html %} Regresa al dataset en **BigQuery** y da clic en **Share** y luego en **Manage Permissions**

  {% include step_image.html %}

- {% include step_label.html %} Ahora clic en **Add principal**

  {% include step_image.html %}

- {% include step_label.html %} Otorga permisos al **writer identity** en el dataset:

  > **NOTA:** Permiso mínimo práctico para escribir en el dataset.
  {: .lab-note .info .compact}

  - New principals: **Pega el valor de tu Writer identity**
  - Select Role: **BigQuery Data Editor**

  {% include step_image.html %}

- {% include step_label.html %} Clic en el botón **Save**

- {% include step_label.html %} Verifica en la consola de **BigQuery** que se hayan creado la tabla.

  {% include step_image.html %}

- {% include step_label.html %} Espera **1–3 minutos** y valida que se crean tablas en el dataset (Cloud Shell).

  > **NOTA:** Si no hay tablas, revisa permisos/filtro y espera un poco más.
  {: .lab-note .important .compact}

  ```bash
  source scripts/env.sh
  bq ls --max_results=50 "${PROJECT_ID}:${DATASET}" | tee outputs/bq_tables_list.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Identifica el nombre de tabla real y guárdalo.

  > **NOTA:** BigQuery crea tablas según `logName`. Listar tablas es la forma más efectiva de descubrir el nombre real.
  {: .lab-note .info .compact}

  ```bash
  source scripts/env.sh
  TABLE_ID="$(bq ls "${PROJECT_ID}:${DATASET}" | awk 'NR>2{print $1}' | grep '^stderr_' | sort | tail -n1)"
  ```
  ```bash
  echo "TABLE_ID=$TABLE_ID" | tee outputs/bq_table_id.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Ejecuta una consulta simple para ver eventos exportados (últimos 50).

  > **NOTA:** Cierre del ciclo: el mismo error de Logs Explorer ahora es consultable en BigQuery.
  {: .lab-note .info .compact}

  ```bash
  source scripts/env.sh
  bq query --use_legacy_sql=false --format=prettyjson "
  SELECT
    timestamp,
    textPayload,
    resource.labels.namespace_name AS namespace,
    resource.labels.pod_name AS pod,
    resource.labels.container_name AS container
  FROM \`${PROJECT_ID}.${DATASET}.${TABLE_ID}\`
  WHERE textPayload LIKE '%${ERROR_ANCHOR}%'
  ORDER BY timestamp DESC
  LIMIT 50;
  " | head -n 60 | tee outputs/bq_query_sample.json
  ```
  {% include step_image.html %}

- {% include step_label.html %} Si la consulta no devuelve filas, confirma que el pod sigue generando logs y reintenta tras 1–2 minutos.

  > **NOTA:** A veces el primer lote tarda en exportarse; confirmar el origen evita perder tiempo.
  {: .lab-note .info .compact}

  ```bash
  source scripts/env.sh
  kubectl -n "$NAMESPACE" get pods -l app=errgen -o wide
  ```
  ```bash
  kubectl -n "$NAMESPACE" logs "$(kubectl -n "$NAMESPACE" get pods -l app=errgen -o jsonpath='{.items[0].metadata.name}')" --tail=10
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[3] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 5. Limpieza

> **Tiempo estimado:** 1 minuto
{: .lab-note .info .compact}

En esta tarea eliminarás recursos para evitar costos: deployment/namespace, sink, dataset y clúster. Harás verificaciones rápidas por CLI.

#### Tarea 5.1

- {% include step_label.html %} Elimina el deployment y el namespace.

  > **NOTA:** Limpieza por namespace es la forma más rápida y segura de borrar recursos del laboratorio.
  {: .lab-note .info .compact}

  ```bash
  source scripts/env.sh
  kubectl -n "$NAMESPACE" delete deploy/errgen --ignore-not-found
  kubectl delete ns "$NAMESPACE" --ignore-not-found
  ```
  ```bash
  kubectl get ns | grep -n "$NAMESPACE" || echo "OK: namespace eliminado" | tee outputs/cleanup_ns.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Elimina el sink.

  > **NOTA:** Si dejas el sink, seguirá exportando logs y podría generar costos a futuro.
  {: .lab-note .warning .compact}

  ```bash
  source scripts/env.sh
  gcloud logging sinks delete "$SINK_NAME" --quiet || true
  ```
  {% include step_image.html %}

- {% include step_label.html %} Elimina el dataset.

  > **NOTA:** BigQuery cobra por almacenamiento y consultas; un dataset pequeño aún es mejor limpiarlo en laboratorio.
  {: .lab-note .warning .compact}

  ```bash
  source scripts/env.sh
  bq rm -r -f -d "${PROJECT_ID}:${DATASET}" || true
  ```
  ```bash
  bq show "${PROJECT_ID}:${DATASET}" >/dev/null 2>&1 || echo "OK: dataset eliminado" | tee outputs/cleanup_dataset.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Elimina el clúster de GKE Autopiloto.

  > **NOTA:** GKE (Autopilot) tiene costos por recursos; eliminar el clúster evita cargos residuales.
  {: .lab-note .warning .compact}

  ```bash
  source scripts/env.sh
  gcloud container clusters delete "$CLUSTER_NAME" --region "$REGION" --quiet || true
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica que ya no existe el cluster.

  ```bash
  source scripts/env.sh
  if gcloud container clusters describe "$CLUSTER_NAME" --region "$REGION" >/dev/null 2>&1; then
    echo "WARN: clúster todavía existe"
  else
    echo "OK: clúster eliminado"
  fi | tee outputs/cleanup_gke_cluster.txt
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[4] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}