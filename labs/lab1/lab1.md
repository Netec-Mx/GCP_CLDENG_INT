---
layout: lab
title: "Práctica 1: Simular cuentas de facturación con datos ficticios"
permalink: /lab1/lab1/
images_base: /labs/lab1/img
duration: "30 minutos"
objective:
  - "Crear un **export simulado** de Cloud Billing en **BigQuery** usando **datos ficticios** (múltiples `billing_account_id`) para análisis de costos por **cuenta / servicio / proyecto**, combinando **Cloud Shell (CLI)** y **Google Cloud Console (UI)**."
prerequisites:
  - "Proyecto de Google Cloud con permisos para BigQuery."
  - "Acceso a **Google Cloud Console** y **Cloud Shell**."
  - "Permisos mínimos sugeridos: crear dataset/tablas y ejecutar consultas en BigQuery."
introduction: |
  En Google Cloud, el **Cloud Billing export a BigQuery** genera tablas con nombres tipo `gcp_billing_export_v1_<BILLING_ACCOUNT_ID>` dentro de un dataset que tú eliges. En esta práctica **no se crea una cuenta de facturación real**: simularemos el export creando una tabla con un **esquema reducido** y cargando un archivo **ndJSON** (un JSON por línea) con datos ficticios. Practicarás un flujo realista: preparar carpeta de laboratorio en Cloud Shell, crear dataset en BigQuery (UI), generar datos sintéticos, cargarlos a BigQuery (CLI), y validar con consultas (UI y CLI) más una **VIEW normalizada** para estabilizar análisis.
slug: lab1
lab_number: 1
final_result: >
  Al finalizar, tendrás un dataset `billing_sim` con una tabla `gcp_billing_export_v1_SIMULATED` (datos ficticios) y una vista `v_billing_sim_normalized` lista para análisis. Podrás demostrar costos y créditos simulados por cuenta/servicio/proyecto usando el editor de BigQuery y el comando `bq query`.
notes:
  - "**No existe** una “billing account ficticia” en GCP: una cuenta real se asocia a un perfil de pagos. Aquí solo simulamos el **export** en BigQuery."
  - "BigQuery **sí tiene costo** (almacenamiento y consultas). En on-demand, el **primer 1 TiB** de datos procesados por consultas al mes es gratis (consulta la página oficial de precios)."
  - "Cloud Shell incluye **5 GB** persistentes en `$HOME` (útil para tus labs)."
  - "En export real, el esquema puede evolucionar; una **VIEW** ayuda a evitar que tus dashboards/queries se rompan."
references:
  - text: "Cloud Billing: Tablas exportadas a BigQuery (nombres `gcp_billing_export_v1_<BILLING_ACCOUNT_ID>`)"
    url: https://docs.cloud.google.com/billing/docs/how-to/export-data-bigquery-tables
  - text: "Cloud Billing: Esquema del Standard usage cost export"
    url: https://docs.cloud.google.com/billing/docs/how-to/export-data-bigquery-tables/standard-usage
  - text: "BigQuery: Batch loading data (carga desde archivos locales / console / bq)"
    url: https://docs.cloud.google.com/bigquery/docs/batch-loading-data
  - text: "BigQuery: Cargar JSON (ndJSON) desde Cloud Storage"
    url: https://docs.cloud.google.com/bigquery/docs/loading-data-cloud-storage-json
  - text: "BigQuery: Usar archivo de esquema JSON (schema file) con `bq load`"
    url: https://docs.cloud.google.com/bigquery/docs/schemas
  - text: "Cloud Shell: cómo funciona (5 GB persistentes en $HOME)"
    url: https://docs.cloud.google.com/shell/docs/how-cloud-shell-works
  - text: "BigQuery pricing (on-demand: primer 1 TiB/mes gratis)"
    url: https://cloud.google.com/bigquery/pricing
prev: /
next: /lab2/lab2/
---

---

### Tarea 1. Preparar el entorno de trabajo 

En esta tarea dejarás listo el entorno en **Cloud Shell**, confirmarás el proyecto activo y crearás una **carpeta por práctica** para organizar schema, scripts, data, SQL y evidencias (outputs).

> **IMPORTANTE:** Cloud Shell usa un VM temporal, pero tu `$HOME` tiene almacenamiento persistente (ideal para laboratorios).
{: .lab-note .important .compact}

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

  > **Nota:** Si sale vacío o incorrecto, configúralo en el siguiente paso. El ID/NOMBRE del proyecto es diferente para cada usuario.
  {: .lab-note .info .compact}

  ```bash
  gcloud config get-value project
  ```
  {% include step_image.html %}

- {% include step_label.html %} Copia el nombre de tu proyecto del paso anterior y configuralo con el siguiente comando. **Sustituye la variable `TU_PROJECT_ID` por el nombre de tu proyecto**

  ```bash
  gcloud config set project TU_PROJECT_ID
  ```
  {% include step_image.html %}

- {% include step_label.html %} Valida que cuentas con herramientas base (`gcloud` y `bq`) operativas.

  ```bash
  gcloud version
  ```
  ```bash
  bq version
  ```
  ```bash
  python3 --version
  ```
  {% include step_image.html %}

- {% include step_label.html %} Crea la estructura base del directorio del curso en tu `$HOME` (persistente) y una carpeta por práctica.

  > **Nota:** Mantendremos una estructura similar en todas las prácticas. Separar carpetas simplifica versionado y auditoría: `schema/` (contrato), `data/` (fuente), `sql/` (análisis), `outputs/` (evidencia).
  {: .lab-note .info .compact}

  ```bash
  cd ~
  ```
  ```bash
  mkdir -p labs-gcp-engineer/lab01/{data,schema,scripts,sql,outputs}
  ```
  ```bash
  cd labs-gcp-engineer/lab01
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica la estructura con el siguiente comando.

  ```bash
  ls -la
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[0] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 2. Crear dataset `billing_sim` en BigQuery

Crearás un dataset en BigQuery usando la **interfaz gráfica**, fijando ubicación y nombre, y lo validarás desde Cloud Shell.

> **IMPORTANTE:** La ubicación del dataset se define al crearlo y no se cambia después.
{: .lab-note .important .compact}

#### Tarea 2.1

- {% include step_label.html %} En la consola, abre **BigQuery** (busca **BigQuery** en el menú lateral izquierdo en las 3 lineas).

  {% include step_image.html %}

- {% include step_label.html %} En el panel **Explorer**, selecciona tu proyecto, expandelo para crear un dataset:

  > **Nota:** Crear por UI acelera el flujo y reduce errores en flags.
  {: .lab-note .info .compact}

- {% include step_label.html %} Clic en simbolo **⋮** junto a **Datasets** selecciona **Create dataset** del menu desplegable.

  {% include step_image.html %}

- {% include step_label.html %} En el panel lateral derecho configura los siguientes datos

  - Dataset ID: `billing_sim`
  - Data location: `us-central1 (iowa)`

   {% include step_image.html %}

- {% include step_label.html %} Clic en el botón **Create dataset**

- {% include step_label.html %} Regresa a Cloud Shell y valida que el dataset existe.

   > **Nota:** Verificar por CLI te deja el proceso listo para automatización futura (IaC / scripts).
  {: .lab-note .info .compact}
   > **IMPORTANTE:** Si es necesario activa `cloudresourcemanager.googleapis.com` y `bigquery.googleapis.com` con el comando `gcloud services enable`
  {: .lab-note .important .compact}

  ```bash
  bq ls
  ```
  ```bash
  bq show --format=prettyjson billing_sim | head
  ```
  {% include step_image.html %}

- {% include step_label.html %} (Opcional) Guarda evidencia del dataset.

   > **Nota:** Si deseas ver el resultado consulta el directorio **outputs** es buena practica guardar evidencia.
  {: .lab-note .info .compact}  

  ```bash
  bq show --format=prettyjson billing_sim > outputs/dataset_billing_sim.json
  ```

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[1] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 3. Crear esquema + generar datos ficticios (ndJSON) en Cloud Shell

Definirás un esquema mínimo **tipo export de facturación** y generarás un archivo **ndJSON** con múltiples cuentas ficticias (`billing_account_id`) y campos clave para análisis.

> **NOTA:** BigQuery requiere que JSON sea **newline-delimited (ndJSON)**: un objeto JSON por línea.
{: .lab-note .info .compact}

#### Tarea 3.1

- {% include step_label.html %} En la terminal de **Google Cloud Shell** crea un archivo de **variables** para el laboratorio.

  ```bash
  cat > scripts/env.sh << 'EOF'
  export BQ_DATASET="billing_sim"
  export BQ_TABLE="gcp_billing_export_v1_SIMULATED"
  export ROWS="180"
  EOF
  ```
  ```bash
  source scripts/env.sh
  ```
  {% include step_image.html %}

- {% include step_label.html %} Crea el esquema `schema/billing_export_sim.schema.json`.

  > **NOTA:** El schema actúa como “contrato” de datos (como en un export real).
  {: .lab-note .info .compact}

  ```bash
  cat > schema/billing_export_sim.schema.json << 'EOF'
  [
    {"name":"billing_account_id","type":"STRING","mode":"REQUIRED"},
    {"name":"billing_account_name","type":"STRING","mode":"NULLABLE"},
    {"name":"invoice_month","type":"STRING","mode":"NULLABLE"},

    {"name":"usage_start_time","type":"TIMESTAMP","mode":"REQUIRED"},
    {"name":"usage_end_time","type":"TIMESTAMP","mode":"REQUIRED"},

    {"name":"project_id","type":"STRING","mode":"NULLABLE"},
    {"name":"project_name","type":"STRING","mode":"NULLABLE"},

    {"name":"service_description","type":"STRING","mode":"NULLABLE"},
    {"name":"sku_description","type":"STRING","mode":"NULLABLE"},

    {"name":"region","type":"STRING","mode":"NULLABLE"},
    {"name":"zone","type":"STRING","mode":"NULLABLE"},

    {"name":"usage_amount","type":"FLOAT","mode":"NULLABLE"},
    {"name":"usage_unit","type":"STRING","mode":"NULLABLE"},

    {"name":"cost","type":"NUMERIC","mode":"REQUIRED"},
    {"name":"currency","type":"STRING","mode":"REQUIRED"},

    {"name":"credits_total","type":"NUMERIC","mode":"NULLABLE"},
    {"name":"export_time","type":"TIMESTAMP","mode":"REQUIRED"}
  ]
  EOF
  ```

- {% include step_label.html %} Crea el script generador `scripts/generate_billing_data.py`.

  > **IMPORTANTE:** Los costos son ficticios (NO reflejan precios reales). El siguiente comando ya crea el script y carga el contenido.
  {: .lab-note .important .compact}

  > **NOTA:** El ndJSON es ideal para cargas: BigQuery asume un objeto por línea.
  {: .lab-note .info .compact}

  ```bash
  cat > scripts/generate_billing_data.py << 'EOF'
  import json, random
  from datetime import datetime, timedelta, timezone

  random.seed(42)

  BILLING_ACCOUNTS = [
      ("000AAA-111BBB-222CCC", "Billing-Sim-Dev"),
      ("333DDD-444EEE-555FFF", "Billing-Sim-Prod"),
      ("666GGG-777HHH-888III", "Billing-Sim-Shared"),
  ]

  PROJECTS = [
      ("finops-dev",  "FinOps Dev"),
      ("finops-prod", "FinOps Prod"),
      ("shared-sbx",  "Shared SBX"),
  ]

  SERVICES = [
      ("Compute Engine", [("N1 Core", "vCPU-hours"), ("N1 RAM", "GB-hours")]),
      ("Cloud Storage",  [("Standard Storage", "GB-month"), ("Egress", "GB")]),
      ("BigQuery",       [("Analysis", "TB"), ("Storage", "GB-month")]),
      ("GKE",            [("Cluster mgmt fee", "hours")]),
  ]

  REGIONS_ZONES = [
      ("us-central1","us-central1-a"),
      ("us-east1","us-east1-b"),
      ("europe-west1","europe-west1-b")
  ]

  def iso(ts): return ts.astimezone(timezone.utc).strftime("%Y-%m-%dT%H:%M:%SZ")

  def main(out_path="data/billing_export_sim.jsonl", rows=180):
      now = datetime.now(timezone.utc)
      start_base = now - timedelta(days=10)

      with open(out_path, "w", encoding="utf-8") as f:
          for _ in range(rows):
              ba_id, ba_name = random.choice(BILLING_ACCOUNTS)
              proj_id, proj_name = random.choice(PROJECTS)
              svc, sku_list = random.choice(SERVICES)
              sku, unit = random.choice(sku_list)
              region, zone = random.choice(REGIONS_ZONES)

              usage_start = start_base + timedelta(hours=random.randint(0, 10*24 - 2))
              usage_end   = usage_start + timedelta(hours=random.randint(1, 6))
              invoice_month = usage_start.strftime("%Y%m")

              usage_amount = round(random.uniform(0.2, 60.0), 3)

              # Tarifas ficticias (NO representan precios reales)
              base_rate = random.uniform(0.01, 2.25)
              cost = round(usage_amount * base_rate, 2)

              credits_total = 0.00
              if random.random() < 0.20:
                  credits_total = round(cost * random.uniform(0.05, 0.35), 2) * -1

              rec = {
                  "billing_account_id": ba_id,
                  "billing_account_name": ba_name,
                  "invoice_month": invoice_month,
                  "usage_start_time": iso(usage_start),
                  "usage_end_time": iso(usage_end),
                  "project_id": proj_id,
                  "project_name": proj_name,
                  "service_description": svc,
                  "sku_description": sku,
                  "region": region,
                  "zone": zone,
                  "usage_amount": usage_amount,
                  "usage_unit": unit,
                  "cost": str(cost),
                  "currency": "USD",
                  "credits_total": str(credits_total),
                  "export_time": iso(now),
              }
              f.write(json.dumps(rec, ensure_ascii=False) + "\n")

  if __name__ == "__main__":
      import argparse
      p = argparse.ArgumentParser()
      p.add_argument("--out", default="data/billing_export_sim.jsonl")
      p.add_argument("--rows", type=int, default=180)
      args = p.parse_args()
      main(args.out, args.rows)
  EOF
  ```

- {% include step_label.html %} Genera el archivo ndJSON con el número de filas configurado.

  > **IMPORTANTE:** Siempre que la terminal se cierra debes cargar las variables.
  {: .lab-note .important .compact}

  ```bash
  source scripts/env.sh
  ```
  ```bash
  python3 scripts/generate_billing_data.py --out data/billing_export_sim.jsonl --rows "$ROWS"
  ```
  {% include step_image.html %}

- {% include step_label.html %} Inspecciona el archivo creado en el paso anterior (sanidad y formato).

  > **IMPORTANTE:** `wc -l` coincide con `ROWS` (ej. 180) y `head -n 1` muestra un JSON completo por línea (sin saltos extra).
  {: .lab-note .important .compact}

  ```bash
  wc -l data/billing_export_sim.jsonl | tee outputs/rows_count.txt
  ```
  {% include step_image.html %}
  ```bash
  head -n 2 data/billing_export_sim.jsonl | tee outputs/sample_rows.txt
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[2] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 4. Cargar datos a BigQuery + VIEW normalizada

Cargarás el ndJSON con `bq load`, verificarás la tabla en UI y ejecutarás consultas para validar conteos, cuentas distintas y costos netos. Después crearás una VIEW para estabilizar el análisis.

#### Tarea 4.1

- {% include step_label.html %} Carga el archivo ndJSON a BigQuery usando `bq load`.

  > **Nota:** Este comando crea una **load job** automáticamente.
  {: .lab-note .info .compact}

  ```bash
  source scripts/env.sh
  ```
  ```bash
  bq load \
    --source_format=NEWLINE_DELIMITED_JSON \
    "${BQ_DATASET}.${BQ_TABLE}" \
    "data/billing_export_sim.jsonl" \
    "schema/billing_export_sim.schema.json"
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica en CLI que la tabla existe y revisa su esquema.

  ```bash
  bq show --schema --format=prettyjson "${BQ_DATASET}.${BQ_TABLE}" | head
  ```
  {% include step_image.html %}

- {% include step_label.html %} En la **UI** de **BigQuery**, da clic para abrir el dataset **billing_sim**.

  {% include step_image.html %}

- {% include step_label.html %} Ahora verifica que exista la tabla creada llamada **`gcp_billing_export_v1_SIMULATED`** y da clic en el nombre.

  {% include step_image.html %}

- {% include step_label.html %} Revisa la pestaña de **Preview**.

  > **Nota:** Toma unos minutos para revisar los datos de la tabla.
  {: .lab-note .info .compact}

  {% include step_image.html %}

- {% include step_label.html %} Revisa la pestaña de **Schema**.

  > **Nota:** Toma unos minutos para revisar el esquema de la tabla
  {: .lab-note .info .compact}

  {% include step_image.html %}

- {% include step_label.html %} Da clic en el botón llamado **Query**

  {% include step_image.html %}

- {% include step_label.html %} Prueba la siguiente consulta en la UI (Query editor) es una validación de conteos. Da clic en el botón **Run**

  ```sql
  SELECT
    COUNT(*) AS row_count,
    COUNT(DISTINCT billing_account_id) AS billing_accounts
  FROM `billing_sim.gcp_billing_export_v1_SIMULATED`;
  ```
  {% include step_image.html %}

- {% include step_label.html %} Ahora ejecuta la siguiente consulta para ver el costo neto por cuenta (cost + credits_total).

  ```sql
  SELECT
    billing_account_id,
    billing_account_name,
    ROUND(SUM(cost), 2) AS gross_cost_usd,
    ROUND(SUM(cost + IFNULL(credits_total, 0)), 2) AS net_cost_usd
  FROM `billing_sim.gcp_billing_export_v1_SIMULATED`
  GROUP BY 1,2
  ORDER BY net_cost_usd DESC;
  ```
  {% include step_image.html %}

- {% include step_label.html %} Abre tu terminal y prueba la siguiente consulta “top servicios” para automatización y guarda la evidencia desde (CLI).

  ```bash
  bq query --use_legacy_sql=false '
  SELECT
    service_description AS service,
    ROUND(SUM(cost + IFNULL(credits_total,0)),2) AS net_cost_usd
  FROM `billing_sim.gcp_billing_export_v1_SIMULATED`
  GROUP BY 1
  ORDER BY net_cost_usd DESC
  LIMIT 10;
  ' | tee outputs/top_services_cli.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Crea una VIEW normalizada desde la CLI.

  ```bash
  bq query --use_legacy_sql=false '
  CREATE OR REPLACE VIEW `billing_sim.v_billing_sim_normalized` AS
  SELECT
    billing_account_id,
    billing_account_name,
    invoice_month,
    project_id,
    project_name,
    service_description AS service,
    sku_description AS sku,
    region,
    zone,
    usage_start_time,
    usage_end_time,
    usage_amount,
    usage_unit,
    cost,
    IFNULL(credits_total, 0) AS credits_total,
    cost + IFNULL(credits_total, 0) AS net_cost,
    currency,
    export_time
  FROM `billing_sim.gcp_billing_export_v1_SIMULATED`;
  '
  ```
  {% include step_image.html %}

- {% include step_label.html %} Valida la VIEW (conteo igual al de la tabla).

  > **NOTA:** La VIEW es un “contrato estable” para BI; si el esquema real cambia, ajustas la VIEW y proteges dashboards.
  {: .lab-note .info .compact}

  ```bash
  bq query --use_legacy_sql=false '
  SELECT
    (SELECT COUNT(*) FROM `billing_sim.gcp_billing_export_v1_SIMULATED`) AS base_rows,
    (SELECT COUNT(*) FROM `billing_sim.v_billing_sim_normalized`) AS view_rows;
  ' | tee outputs/view_validation.txt
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[3] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 5. Limpieza de recursos

Eliminarás los recursos creados (tabla, view y/o dataset) para evitar consumo adicional.

> **NOTA:** BigQuery cobra por almacenamiento y consultas; limpia si este proyecto es compartido o si no lo necesitas.
{: .lab-note .info .compact}

#### Tarea 5.1

- {% include step_label.html %} En la terminal de GCP ejecuta el siguiente comando para eliminar la tabla.

  ```bash
  bq rm -f -t billing_sim.gcp_billing_export_v1_SIMULATED
  ```

- {% include step_label.html %} Ahora en la interfaz grafica de **BigQuery** da clic en los 3 puntos de la vista (lado derecho) y selecciona la opción **Delete**

  {% include step_image.html %}

- {% include step_label.html %} Confirma la ventana emergente escribe **`delete`** y da clic en el botón **Delete**

  {% include step_image.html %}

- {% include step_label.html %} En la interfaz grafica de **BigQuery** elimina también el dataset, clic en los 3 puntos de lado derecho y luego **Delete**

  {% include step_image.html %}

- {% include step_label.html %} Confirma la ventana emergente y da clic en **Delete**

  {% include step_image.html %}

- {% include step_label.html %} Ya que se elimino de la UI ahora reconfirma mediante la terminal que ya no exista.

  ```bash
  bq show billing_sim.gcp_billing_export_v1_SIMULATED >/dev/null 2>&1 || echo "Tabla eliminada"
  ```
  ```bash
  bq show billing_sim.v_billing_sim_normalized >/dev/null 2>&1 || echo "View eliminada"
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[4] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}
