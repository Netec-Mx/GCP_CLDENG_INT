---
layout: lab
title: "Práctica 3: Crear bucket en Cloud Storage con versionado y ciclo de vida"
permalink: /lab3/lab3/
images_base: /labs/lab3/img
duration: "30 minutos"
objective:
  - "Crear un bucket de **Cloud Storage** con **Object Versioning** habilitado y una política de **Object Lifecycle Management** para controlar el costo del versionado (retener solo versiones útiles)."
prerequisites:
  - "Proyecto de Google Cloud con permisos para Cloud Storage (crear buckets, actualizar configuración, listar/subir objetos)."
  - "Acceso a **Google Cloud Console** y **Cloud Shell**."
  - "Entender diferencia entre **región** y **multi-región**."
introduction: |
  Cloud Storage organiza datos en **buckets** (contenedores globalmente únicos) y **objetos**.  
  - **Object Versioning** conserva versiones “no actuales” (noncurrent) cuando sobrescribes o borras un objeto: útil para recuperación y auditoría.  
  - El **versionado incrementa almacenamiento** porque cada versión se cobra como un objeto completo; por eso se recomienda combinarlo con **Object Lifecycle Management**, que permite borrar versiones viejas o mover objetos a clases más frías automáticamente.
slug: lab3
lab_number: 3
final_result: >
  Al finalizar, tendrás un bucket con versionado activo, objetos con múltiples generaciones, y reglas de ciclo de vida aplicadas (por ejemplo: borrar versiones no actuales viejas y/o limitar versiones retenidas). Validarás configuración y versiones desde la consola y por CLI.
notes:
  - "Cloud Storage **tiene costo**: almacenamiento, operaciones (Class A/B), y salida de red. Object Versioning puede incrementar costo porque cada versión no actual se cobra como almacenamiento adicional."
  - "Lifecycle no ejecuta cambios inmediatamente al crear reglas; se aplica de forma automática cuando se cumplen condiciones."
  - "Recomendado: habilitar **Uniform bucket-level access** para una postura IAM moderna."
references:
  - text: "Crear un bucket (Console / gcloud)"
    url: https://docs.cloud.google.com/storage/docs/creating-buckets
  - text: "Usar Object Versioning (habilitar / validar / costos de versiones)"
    url: https://docs.cloud.google.com/storage/docs/using-object-versioning
  - text: "Usar objetos versionados (listar/restaurar/borrar versiones)"
    url: https://docs.cloud.google.com/storage/docs/using-versioned-objects
  - text: "Object Lifecycle Management (conceptos)"
    url: https://docs.cloud.google.com/storage/docs/lifecycle
  - text: "Configurar reglas de ciclo de vida (ejemplos y UI)"
    url: https://docs.cloud.google.com/storage/docs/lifecycle-configurations
  - text: "Ejemplo CLI: lifecycle-file con gcloud storage buckets update"
    url: https://docs.cloud.google.com/storage/docs/working-with-big-data
  - text: "Precios de Cloud Storage"
    url: https://cloud.google.com/storage/pricing
prev: /lab2/lab2/
next: /lab4/lab4/
---

---

## Instrucciones generales

- Esta práctica es **mayormente por interfaz gráfica (UI)** y se apoya en **Cloud Shell** para:
  - Generar un nombre único de bucket,
  - Validar configuración (versioning/lifecycle),
  - Listar versiones y evidencias.
- Mantendremos el estándar de carpetas del curso:
  - `~/labs-gcp-engineer/lab03/` con `scripts/`, `outputs/`, `data/`.
- Convención recomendada:
  - **Location:** `us-central1`.
  - **Storage class:** `Standard` (para el lab).

---

### Tarea 1. Preparar Cloud Shell + carpeta de práctica (Tiempo estimado: 5 min)

En esta tarea activarás Cloud Shell, confirmarás proyecto y crearás estructura de carpetas para scripts y evidencias del laboratorio.

> **IMPORTANTE:** Cloud Shell conserva tu `$HOME` (5 GB persistentes).
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

- {% include step_label.html %} Crea la carpeta del laboratorio 3.

  > **NOTA:** `scripts/` guarda configuración reproducible, `outputs/` guarda evidencias.
  {: .lab-note .info .compact}

  ```bash
  cd ~
  mkdir -p labs-gcp-engineer/lab03/{scripts,outputs,data}
  ```
  ```bash
  cd labs-gcp-engineer/lab03
  ```
  {% include step_image.html %}

- {% include step_label.html %} Crea archivo de variables del lab: `scripts/env.sh`.

  > **IMPORTANTE:** Edita el siguiente codigo para cambiar los valores de tu prefijo. Sustituye las **xxx** y los **###** por valores aleatorios.
  {: .lab-note .important .compact}

  ```bash
  cat > scripts/env.sh << 'EOF'
  # Ajusta la ubicación a tu laboratorio:
  # Ejemplos: US (multi-región), us-central1 (región), europe-west1 (región)
  export BUCKET_LOCATION="us-central1"

  # Clase por defecto (para el lab): STANDARD / NEARLINE / COLDLINE / ARCHIVE
  export DEFAULT_STORAGE_CLASS="STANDARD"

  # Prefijo del bucket (debe ser minúsculas, dígitos y guiones)
  export BUCKET_PREFIX="lab03-gcs-xxx-###"
  EOF
  ```
  ```bash
  source scripts/env.sh
  ```

- {% include step_label.html %} Valida las herramientas y guarda evidencia.

  > **NOTA:** `gcloud storage` es el grupo de comandos moderno para Storage.
  {: .lab-note .info .compact}

  ```bash
  gcloud version | head -n 1 | tee outputs/gcloud_version.txt
  ```
  ```bash
  gcloud storage --help >/dev/null 2>&1 && echo "OK: gcloud storage disponible" | tee outputs/gcloud_storage_ok.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Lista estructura de carpetas y guarda evidencia.

  ```bash
  ls -la | tee outputs/lab03_tree_root.txt
  ```
  ```bash
  ls -la scripts outputs data | tee outputs/lab03_tree_subdirs.txt
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[0] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 2. Crear bucket con configuración base (Tiempo estimado: 10 min)

Crearás un bucket desde la consola, configurando ubicación, storage class y uniform bucket-level access. Luego validarás desde Cloud Shell.

> **IMPORTANTE:** El nombre del bucket debe ser **globalmente único**.
{: .lab-note .important .compact}

#### Tarea 2.1 (Cloud Shell) — Generar nombre único y guardarlo

- {% include step_label.html %} Genera un nombre único (recomendado) y guárdalo en `scripts/env.sh` para reutilizarlo.

  > **NOTA:** El bucket name es global; sumar Project + timestamp minimiza colisiones.
  {: .lab-note .info .compact}

  ```bash
  source scripts/env.sh
  PROJECT_ID="$(gcloud config get-value project)"
  TS="$(date +%s)"
  BUCKET_NAME="${BUCKET_PREFIX}-${PROJECT_ID}-${TS}"
  BUCKET_NAME="$(echo "$BUCKET_NAME" | tr '[:upper:]' '[:lower:]' | tr '_' '-')"
  ```
  ```bash
  echo "export BUCKET_NAME="$BUCKET_NAME"" >> scripts/env.sh
  source scripts/env.sh
  ```
  ```bash
  echo "BUCKET_NAME=$BUCKET_NAME" | tee outputs/bucket_name.txt
  ```
  {% include step_image.html %}

#### Tarea 2.2 (UI) — Crear bucket

- {% include step_label.html %} Ve a **Cloud Storage** y luego a **Buckets**.

  {% include step_image.html %}

- {% include step_label.html %} Da clic en **Create** ó **Create bucket**.

  > **NOTA:** Un bucket es el contenedor “raíz” para objetos y políticas (versionado, lifecycle, IAM).
  {: .lab-note .info .compact}

  {% include step_image.html %}

- {% include step_label.html %} En **Getting started** configura lo siguiente:

  > **NOTA:** El nombre define el namespace global `gs://<bucket>`.
  {: .lab-note .info .compact}

  - **Bucket name:** pega el valor de `BUCKET_NAME` (desde `outputs/bucket_name.txt` o el que te dio en el **Cloud Shell**)

  {% include step_image.html %}

- {% include step_label.html %} Da clic en **Continue**

  {% include step_image.html %}

- {% include step_label.html %} En **Choose where to store your data**:

  > **NOTA:** Región es mejor para residencia/latencia.
  {: .lab-note .info .compact}

  - Location type: **Region**
  - Location: `us-central1`

  {% include step_image.html %}

- {% include step_label.html %} Da clic en **Continue**

- {% include step_label.html %} En **Choose how to store your data**:

  > **NOTA:** `Standard` es mejor para acceso frecuente; clases frías son para acceso raro.
  {: .lab-note .info .compact}

  - Default class: `Standard`

  {% include step_image.html %}

- {% include step_label.html %} Da clic en **Continue**

- {% include step_label.html %} En **Choose how to control access to objects**:

  > **NOTA:** `Uniform access` simplifica permisos usando IAM (en lugar de ACLs por objeto).
  {: .lab-note .info .compact}

  - Habilita **Uniform bucket-level access**

  {% include step_image.html %}

- {% include step_label.html %} Da clic en **Continue** y luego **Create**

  {% include step_image.html %}

#### Tarea 2.3 (Cloud Shell) — Confirmar bucket y metadatos

- {% include step_label.html %} **Desde Cloud Shell** Valida que el bucket existe y guarda evidencia.

  ```bash
  source scripts/env.sh
  gcloud storage buckets describe "gs://$BUCKET_NAME" --format="yaml(name,location,storageClass,iamConfiguration.uniformBucketLevelAccess.enabled)"     | tee outputs/bucket_describe_base.yaml
  ```
  {% include step_image.html %}

- {% include step_label.html %} Lista el bucket en tu proyecto (sanidad).

  > **NOTA:** Si aparece, estás apuntando al proyecto correcto y el bucket quedó creado.
  {: .lab-note .info .compact}

  ```bash
  gcloud storage buckets list \
    --filter="name:$BUCKET_NAME" \
    --format="value(name)" \
  | tee outputs/bucket_list.txt
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[1] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 3. Habilitar Object Versioning y comprobarlo (Tiempo estimado: 6 min)

Activarás el versionado para conservar generaciones anteriores al sobrescribir/borrar objetos.

> **IMPORTANTE:** No hay límite por defecto de versiones; cada versión no actual se cobra como almacenamiento adicional. Considera lifecycle para controlar costos.
{: .lab-note .important .compact}

#### Tarea 3.1 (UI) — Activar versioning

- {% include step_label.html %} Abre tu bucket en **Cloud Storage** / **Buckets** y entra a **Bucket details**.

  > **NOTA:** Las configuraciones de versionado/ciclo de vida se aplican a nivel bucket.
  {: .lab-note .info .compact}

  {% include step_image.html %}

- {% include step_label.html %} Ve a la pestaña **Protection** y habilita **Object versioning**.

  > **Nota:** En algunos flujos, la UI ofrece “Add recommended lifecycle rules to manage version costs”. Puedes dejarlo desmarcado por ahora; definiremos reglas de forma explícita más adelante. Versioning te permite “volver atrás” si alguien sobrescribe o borra un archivo.
  {: .lab-note .info .compact}

  {% include step_image.html %}

- {% include step_label.html %} Confirma la ventana emergente para la activacion del versionamiento. No actives el checkbox de recomendaciones.

  {% include step_image.html %}

#### Tarea 3.2 (Cloud Shell) — Confirmar versioning

- {% include step_label.html %} Valida por CLI que versioning quedó activo (busca `enabled: true`).

  ```bash
  source scripts/env.sh
  gcloud storage buckets describe "gs://$BUCKET_NAME" --format="yaml(name,versioning_enabled)" | tee outputs/bucket_versioning.yaml
  ```
  {% include step_image.html %}

- {% include step_label.html %} Guarda evidencia “amigable” del valor `enabled` (si aparece).

  ```bash
  grep -n "enabled" outputs/bucket_versioning.yaml || true
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[2] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 4. Subir objetos y crear múltiples versiones (Tiempo estimado: 6–7 min)

Subirás un archivo, lo sobrescribirás varias veces para generar versiones, listarás generaciones y restaurarás una versión previa.

#### Tarea 4.1 (Cloud Shell) — Crear archivos locales “v1/v2/v3”

- {% include step_label.html %} Crea un archivo y tres versiones locales.

  > **NOTA:** Sobrescribir el mismo objeto con contenido diferente crea “generaciones” cuando versioning está activo.
  {: .lab-note .info .compact}

  ```bash
  cat > data/report.txt <<'EOF'
  Reporte de prueba - version 1
  Timestamp: REPLACE_TS
  EOF
  sed -i "s/REPLACE_TS/$(date -u +%FT%TZ)/g" data/report.txt

  cp data/report.txt data/report_v1.txt
  echo "Reporte de prueba - version 2" > data/report_v2.txt
  echo "Reporte de prueba - version 3" > data/report_v3.txt

  ls -la data | tee outputs/data_files_list.txt
  ```
  {% include step_image.html %}

#### Tarea 4.2 (UI) — Subir el objeto (primera versión)

- {% include step_label.html %} En la UI del bucket (pestaña **Objects**), clic **Upload files** y sube `data/report_v1.txt`.

  > **Nota:** Renómbralo a `report.txt` desde el diálogo si la UI lo permite; si no, súbelo como `report_v1.txt` y en el siguiente paso usaremos CLI para el objeto “real” `report.txt`. UI es ideal para aprender el flujo manual del día a día.
  {: .lab-note .info .compact}

  > **IMPORTANTE:** Este es un paso demostrativo la carga la realizaras mediante el **Cloud Shell** ya que ahi se generaron los archivos.
  {: .lab-note .important .compact}

  {% include step_image.html %}

#### Tarea 4.3 (Cloud Shell) — Crear/actualizar el objeto `report.txt` tres veces

- {% include step_label.html %} Sube `report_v1.txt` como `report.txt`.

  > **NOTA:** Este será el “objeto principal” sobre el que generaremos versiones.
  {: .lab-note .info .compact}

  ```bash
  source scripts/env.sh
  gcloud storage cp data/report_v1.txt "gs://$BUCKET_NAME/report.txt"
  ```
  {% include step_image.html %}

- {% include step_label.html %} Sobrescribe el mismo objeto con v2 y v3 (crea versiones).

  > **NOTA:** Cada sobrescritura produce una nueva generación (la anterior pasa a noncurrent).
  {: .lab-note .info .compact}

  ```bash
  gcloud storage cp data/report_v2.txt "gs://$BUCKET_NAME/report.txt"
  ```
  ```bash
  gcloud storage cp data/report_v3.txt "gs://$BUCKET_NAME/report.txt"
  ```
  {% include step_image.html %}

- {% include step_label.html %} Lista versiones del objeto (muestra generaciones). Si `gsutil` está disponible, este comando es el más “visual” para ver generaciones:

  > **NOTA:** `-a` lista todas las versiones (incluye no actuales) usando sintaxis `#<generation>`.
  {: .lab-note .info .compact}

  ```bash
  gsutil ls -a "gs://$BUCKET_NAME/report.txt" | tee outputs/report_versions_ls_a.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Obtén metadatos (incluye `Generation` y `Metageneration`) de la versión “live”.

  > **NOTA:** **`Generation`** identifica una versión específica del objeto.
  {: .lab-note .info .compact}

  ```bash
  gsutil stat "gs://$BUCKET_NAME/report.txt" | tee outputs/report_stat_live.txt
  ```
  {% include step_image.html %}

#### Tarea 4.4 (UI) — Ver “Live and noncurrent objects” y restaurar una versión

- {% include step_label.html %} En la UI del bucket, usa el filtro **Show** y selecciona **Live and noncurrent objects** para ver versiones no actuales.

  > **NOTA:** En incidentes, esto sirve para recuperar “lo que había antes”.
  {: .lab-note .info .compact}

  {% include step_image.html %}

- {% include step_label.html %} Da clic en la opción **2 noncurrent versions** de la columna **Version history**

  {% include step_image.html %}

- {% include step_label.html %} Te mostrara todos los archivos cargados con las versiones.

  {% include step_image.html %}

- {% include step_label.html %} Restaura una versión previa por CLI (simulación práctica):

  > **NOTA:** Restaurar es literalmente “copiar” una generación vieja para que vuelva a ser la actual.
  {: .lab-note .info .compact}

  ```bash
  OBJ="report.txt"
  VERS_FILE="outputs/report_versions_ls_a.txt"
  ```
  ```bash
  grep -E "^gs://.*/${OBJ}#[0-9]+$" "$VERS_FILE" | head -n 20
  ```
  ```bash
  GEN_URL="$(grep -E "^gs://.*/${OBJ}#[0-9]+$" "$VERS_FILE" | sort -t'#' -k2,2n | head -n 1)"
  ```
  ```bash
  echo "GEN_URL=$GEN_URL"
  ```
  ```bash
  gsutil cp "$GEN_URL" "gs://$BUCKET_NAME/$OBJ"
  ```
  ```bash
  gsutil ls -a "gs://$BUCKET_NAME/$OBJ" | tee outputs/report_versions_after_restore.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica contenido actual tras restauración (debe cambiar).

  ```bash
  gcloud storage cat "gs://$BUCKET_NAME/report.txt" | head -n 5 | tee outputs/report_after_restore.txt
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[3] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 5. Configurar ciclo de vida (Lifecycle) para controlar versiones y costos (Tiempo estimado: 6–7 min)

Crearás reglas de ciclo de vida para:
- Borrar versiones no actuales cuando haya demasiadas (retener últimas N), opcionalmente mover objetos “live” a una clase más fría tras X días.

> **NOTA:** Lifecycle se configura en el bucket y aplica a objetos actuales y futuros.
{: .lab-note .info .compact}

#### Tarea 5.1 (UI) — Crear reglas (recomendado, visual)

- {% include step_label.html %} En el bucket, ve a la pestaña **Lifecycle** y clic **Add a rule**.

  {% include step_image.html %}

- {% include step_label.html %} Regla A (recomendada para versionado): **Delete noncurrent versions**.

  > **NOTA:** Esta regla evita que el bucket acumule demasiadas versiones (costo). 
  {: .lab-note .info .compact}

  - Select an action: **Delete object**
  - clic en **Continue**

  {% include step_image.html %}

  - En **Select object conditions**

  - Number of newer versions: **2**
  - Live state: **Noncurrent**
  - Days since custom time: **7**
  - Clic en **Continue**

  {% include step_image.html %}

- {% include step_label.html %} Clic en **Create**

  {% include step_image.html %}

- {% include step_label.html %} Regla B (opcional): Clic en **Add a rule**, **Set storage class** para objetos “live” antiguos.

  > **NOTA:** Mover a clases frías reduce costo de almacenamiento, pero puede aumentar costo/latencia de acceso.
  {: .lab-note .info .compact}

  - Select an action: **Set storage class to Nearline**
  - Clic en **Continue**

  {% include step_image.html %} 

  - En Select object conditions
  
  - Age: **30 days** 
  - Clic en **Continue**

  {% include step_image.html %}

- {% include step_label.html %} Clic en **Create**

  {% include step_image.html %}

#### Tarea 5.2 (Cloud Shell) — Aplicar lifecycle por archivo (reproducible) y verificar

- {% include step_label.html %} Crea un archivo JSON de lifecycle (reproducible) en `scripts/lifecycle_config.json`.

  > **Nota:** Este ejemplo:
  > - Borra versiones no actuales que tengan **2 versiones más nuevas** (retiene ~últimas 2).
  > - Borra versiones no actuales con **edad >= 7 días**.
  > - Mueve objetos live con edad >= 30 días a **NEARLINE**.
  {: .lab-note .info .compact}

  ```bash
  source scripts/env.sh
  ```
  ```bash
  cat > scripts/lifecycle_config.json <<'EOF'
  {
    "rule": [
      {
        "action": {"type": "Delete"},
        "condition": {"isLive": false, "numNewerVersions": 2}
      },
      {
        "action": {"type": "Delete"},
        "condition": {"isLive": false, "age": 7}
      },
      {
        "action": {"type": "SetStorageClass", "storageClass": "NEARLINE"},
        "condition": {"isLive": true, "age": 30}
      }
    ]
  }
  EOF
  ```

- {% include step_label.html %} Aplica el lifecycle por CLI al bucket (idempotente).

  ```bash
  gcloud storage buckets update "gs://$BUCKET_NAME" --lifecycle-file=scripts/lifecycle_config.json
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica que el bucket ya incluye reglas de lifecycle (y guarda evidencia).

  ```bash
  gcloud storage buckets describe "gs://$BUCKET_NAME" --format="yaml(name,lifecycle_config,versioning_enabled)" | tee outputs/bucket_lifecycle_and_versioning.yaml
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[4] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 6. Limpieza (Tiempo estimado: 2–4 min)

Eliminarás el bucket para evitar costos futuros (si no lo necesitas).

> **IMPORTANTE:** Para borrar el bucket, primero debes borrar los objetos (incluyendo versiones).
{: .lab-note .important .compact}

#### Tarea 6.1 (Cloud Shell) — Borrar objetos y bucket

- {% include step_label.html %} Borra **todas** las versiones de objetos dentro del bucket.

  > **NOTA:** `-a` elimina versiones (generaciones) además del objeto “live”.
  {: .lab-note .info .compact}

  ```bash
  source scripts/env.sh
  gsutil -m rm -a "gs://$BUCKET_NAME/**" || true
  ```
  {% include step_image.html %}

- {% include step_label.html %} Borra el bucket.

  ```bash
  gsutil rb "gs://$BUCKET_NAME" && echo "OK: bucket eliminado" | tee outputs/bucket_deleted.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica que ya no existe.

  ```bash
  gcloud storage buckets describe "gs://$BUCKET_NAME" >/dev/null 2>&1 || echo "OK: bucket no existe"
  ```
  {% include step_image.html %}


{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[5] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}