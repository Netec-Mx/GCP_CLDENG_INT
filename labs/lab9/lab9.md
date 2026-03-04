---
layout: lab
title: "Práctica 9: Conectar Pub/Sub con Cloud Function para procesamiento de eventos"
permalink: /lab9/lab9/
images_base: /labs/lab9/img
duration: "40 minutos"
objective:
  - "Implementar un flujo **event-driven** donde un **mensaje de Pub/Sub** activa una **Cloud Function (Cloud Run functions)** para procesar el evento y registrarlo en **Cloud Logging**, validando extremo a extremo con publicación de mensajes (UI y CLI) y evidencias operativas."
prerequisites:
  - "Proyecto de Google Cloud con **facturación habilitada**."
  - "Permisos para: **Pub/Sub**, **Cloud Run functions/Cloud Functions**, **Eventarc**, **Cloud Build**, **Artifact Registry** y **Logging**."
  - "Acceso a **Google Cloud Console** y a **Cloud Shell**."
introduction: |
  Pub/Sub desacopla **productores** y **consumidores** mediante **topics**: el productor publica mensajes y los consumidores procesan de forma asíncrona. Con **Cloud Functions  (Cloud Run functions)** puedes reaccionar a eventos de Pub/Sub (vía **Eventarc**) para ejecutar lógica bajo demanda: validación, enriquecimiento, enrutamiento, notificación, o disparar pipelines.

  En esta práctica crearás un **topic**, publicarás eventos con **atributos**, desplegarás una **función** que decodifica el mensaje (base64), parsea JSON y escribe logs estructurados. Finalmente verificarás ejecuciones en UI (Logs) y por CLI (gcloud logging), y dejarás un flujo listo para evolucionar a producción.
slug: lab9
lab_number: 9
final_result: >
  Al finalizar, tendrás un topic de Pub/Sub y una Cloud Function (Cloud Run functions) conectada como consumidor de eventos, con pruebas end-to-end (mensajes publicados desde UI/CLI) y evidencias en Cloud Logging, además de comandos de verificación y limpieza opcional.
notes:
  - "Cloud Functions aparece como **Cloud Run functions** en algunos menús de la consola. Es la opción moderna recomendada."
  - "En Cloud Run Functions, el despliegue construye un contenedor (Cloud Build) y puede almacenar artefactos (Artifact Registry)."
  - "Los logs de ejecución se consultan desde Cloud Logging; es normal que tarden 1–2 minutos en aparecer tras la primera invocación."
  - "**Costos:** Pub/Sub y Cloud Run functions son pay-as-you-go. Cloud Build/Artifact Registry pueden generar consumo durante despliegue. Elimina recursos al terminar el laboratorio."
references:
  - text: "Pub/Sub: conceptos básicos"
    url: https://cloud.google.com/pubsub/docs/pubsub-basics
  - text: "Crear un topic de Pub/Sub"
    url: https://cloud.google.com/pubsub/docs/create-topic
  - text: "Publicar y recibir mensajes con gcloud"
    url: https://cloud.google.com/pubsub/docs/publish-receive-messages-gcloud
  - text: "Cloud Functions / Cloud Run functions - documentación"
    url: https://cloud.google.com/functions/docs
  - text: "Ejemplo Pub/Sub (CloudEvent) para Cloud Functions"
    url: https://cloud.google.com/functions/docs/samples/functions-cloudevent-pubsub
  - text: "Cloud Logging: ver y consultar logs"
    url: https://cloud.google.com/logging/docs/view/overview
  - text: "Eventarc: conceptos"
    url: https://cloud.google.com/eventarc/docs/overview
  - text: "Precios Pub/Sub"
    url: https://cloud.google.com/pubsub/pricing
  - text: "Precios Cloud Functions / Cloud Run functions"
    url: https://cloud.google.com/functions/pricing-overview
prev: /lab8/lab8/
next: /lab10/lab10/
---

---

### Tarea 1. Preparar el entorno de trabajo

> **Tiempo estimado:** 5 minutos
{: .lab-note .info .compact}

En esta tarea dejarás listo Cloud Shell, validarás el proyecto y crearás la carpeta del laboratorio (código + scripts + evidencias).

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

- {% include step_label.html %} Crea la carpeta del laboratorio 9 y subcarpetas estándar.

  > **NOTA:** `function/` = código desplegable; `scripts/` = utilidades; `outputs/` = evidencias para revisión.
  {: .lab-note .info .compact}

  ```bash
  cd ~
  mkdir -p labs-gcp-engineer/lab09/{function,scripts,outputs}
  ```
  ```bash
  cd labs-gcp-engineer/lab09
  ```

  {% include step_image.html %}

- {% include step_label.html %} Crea `scripts/env.sh` con variables del laboratorio (nombres alineados a lab09).

  > **NOTA:** Variables consistentes = menos errores al copiar/pegar y más rapidez en validaciones y limpieza.
  {: .lab-note .important .compact}

  ```bash
  cat > scripts/env.sh <<'EOF'
  export REGION="us-central1"
  export LAB_ID="lab09"

  # Pub/Sub
  export TOPIC_ID="lab09-events"
  export SUB_ID="lab09-events-sub"

  # Cloud Run functions / Cloud Functions ()
  export FUNCTION_NAME="lab09-ps-handler"
  export ENTRY_POINT="process_pubsub_event"
  export RUNTIME="python312"
  EOF
  ```
  ```bash
  source scripts/env.sh
  ```

- {% include step_label.html %} Valida estructura del directorio y guarda evidencia.

  > **NOTA:** Evidencia básica de que tu laboratorio quedó en la ruta esperada.
  {: .lab-note .warning .compact}

  ```bash
  pwd | tee outputs/pwd.txt
  ```
  ```bash
  ls -la | tee outputs/ls_root.txt
  ```
  ```bash
  ls -la function scripts outputs | tee outputs/ls_subfolders.txt
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[0] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 2. Habilitar APIs necesarias

> **Tiempo estimado:** 6 minutos
{: .lab-note .info .compact}

En esta tarea habilitarás las APIs requeridas para Pub/Sub y Cloud Run functions (), incluyendo Eventarc, Cloud Build, Artifact Registry y Logging.

#### Tarea 2.1

- {% include step_label.html %} **En Cloud Shell**, ejecuta el siguiente comando para instalar las APIs faltantes.

  > **NOTA:** En Google Cloud, habilitar APIs es equivalente a “activar capacidades” del proyecto.
  {: .lab-note .info .compact}

  > **NOTA:** El despliegue usa Cloud Build y puede generar artefactos; Eventarc conecta eventos; Logging registra ejecución.
  {: .lab-note .warning .compact}

  ```bash
  gcloud services enable \
    pubsub.googleapis.com \
    cloudfunctions.googleapis.com \
    run.googleapis.com \
    eventarc.googleapis.com \
    cloudbuild.googleapis.com \
    artifactregistry.googleapis.com \
    logging.googleapis.com
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica por CLI que las APIs están habilitadas y guarda evidencia.

  > **NOTA:** Esta salida es evidencia objetiva y acelera troubleshooting.
  {: .lab-note .info .compact}

  ```bash
  gcloud services list --enabled \
    --format='value(config.name)' \
  | grep -E 'pubsub|cloudfunctions|run|eventarc|cloudbuild|artifactregistry|logging\.googleapis\.com' \
  | sort \
  | tee outputs/enabled_apis.txt
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[1] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 3. Crear Pub/Sub topic y suscripción, y probar publicación/consumo

> **Tiempo estimado:** 8 minutos
{: .lab-note .info .compact}

En esta tarea crearás un topic y una suscripción de prueba para validar Pub/Sub antes de conectar la función.

#### Tarea 3.1

- {% include step_label.html %} En el buscador de la consola, escribe **Pub/Sub** luego  **Topics**.

  {% include step_image.html %}

- {% include step_label.html %} Haz clic en **Create topic**.

  {% include step_image.html %}

- {% include step_label.html %} Crea el **topic** con los siguientes datos:

  > **NOTA:** El **topic** es el “canal” donde se publican eventos (los consumidores se conectan con suscripciones). La suscripción por defecto acelera la verificación manual en laboratorios.
  {: .lab-note .info .compact}

  - Topic ID: `lab09-events`
  - Marca **Add a default subscription** para prueba rápida

  {% include step_image.html %}

- {% include step_label.html %} Click en **Create**

- {% include step_label.html %} Crea también una suscripción dedicada, clic en **Subscriptions**:

  > **NOTA:** Tener una suscripción propia te permite probar pull sin depender del nombre autogenerado de la “default”.
  {: .lab-note .info .compact}

  {% include step_image.html %}

- {% include step_label.html %} Ahora clic en **Create subscription**

  {% include step_image.html %}

- {% include step_label.html %} Configura los siguientes datos:

  > **NOTA:** Este paso es un ejemplo de la creación de la siscripción. En la practica se usara la que se creo con el topico.
  {: .lab-note .info .compact}

  - Subscription ID: `lab09-events-sub-test`
  - Select a Cloud Pub/Sub topic: **lab09-events**
  - Delivery type: Pull

  {% include step_image.html %}

- {% include step_label.html %} Click en **Create**

- {% include step_label.html %} Publica un mensaje de ejemplo, da clic en **Topics** → `lab09-events` → **Messages → Publish message**:

  {% include step_image.html %}

- {% include step_label.html %} Configura los siguientes datos:

  > **NOTA:** Atributos son metadatos útiles para ruteo y observabilidad (por ejemplo: `env`, `source`, `type`).
  {: .lab-note .info .compact}

  - Message body: `hola-lab09`
  - Messages attributes:
    - Key 1: `source`
    - Value 1: `ui`

  {% include step_image.html %}

- {% include step_label.html %} Click en **Publish**

- {% include step_label.html %} Valida por CLI que el **topic** existe y guarda evidencia.

  > **NOTA:** Confirma que el recurso existe y en el proyecto correcto.
  {: .lab-note .info .compact}

  ```bash
  source scripts/env.sh
  gcloud pubsub topics describe "$TOPIC_ID" --format="yaml(name,labels,messageStoragePolicy)" | tee outputs/topic_describe.yaml
  ```
  {% include step_image.html %}

- {% include step_label.html %} Valida por CLI que la **suscripción** existe y guarda evidencia.

  > **NOTA:** Verifica que la suscripción apunta al topic correcto.
  {: .lab-note .info .compact}

  ```bash
  source scripts/env.sh
  gcloud pubsub subscriptions describe "$SUB_ID" --format="yaml(name,topic,ackDeadlineSeconds)" | tee outputs/sub_describe.yaml
  ```
  {% include step_image.html %}

- {% include step_label.html %} Consume (pull) 1 mensaje desde la suscripción y confirma que llega.

  > **NOTA:** Si aquí no puedes consumir, aún NO es problema de la función; es Pub/Sub/permisos/configuración.
  {: .lab-note .important .compact}

  ```bash
  source scripts/env.sh
  gcloud pubsub subscriptions pull "$SUB_ID" --auto-ack --limit=1 | tee outputs/pull_1.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Publica 2 mensajes más por CLI (para siguientes tareas) y guarda evidencia.

  > **NOTA:** CLI es ideal para generar mensajes repetibles y automatizables.
  {: .lab-note .info .compact}

  ```bash
  source scripts/env.sh
  gcloud pubsub topics publish "$TOPIC_ID" --message "ping-1" --attribute=source=cli,env=lab | tee -a outputs/publish_cli.txt
  ```
  ```bash
  gcloud pubsub topics publish "$TOPIC_ID" --message "ping-2" --attribute=source=cli,env=lab | tee -a outputs/publish_cli.txt
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[2] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---
### Tarea 4. Preparar el código de la Cloud Run Function

> **Tiempo estimado:** 7 minutos
{: .lab-note .info .compact}

En esta tarea crearás el código (Python) que recibe eventos Pub/Sub como CloudEvent, decodifica base64, parsea JSON si aplica y escribe un log estructurado.

#### Tarea 4.1

- {% include step_label.html %} **En Cloud Shell**, crea `main.py` y `requirements.txt` en `function/`.

  > **NOTA:** En , el runtime instala dependencias en build. `functions-framework` habilita el entrypoint estándar.
  {: .lab-note .info .compact}

  ```bash
  cd ~/labs-gcp-engineer/lab09/function
  ```
  {% include step_image.html %}
  ```bash
  cat > main.py <<'PY'
  import base64
  import json
  from datetime import datetime, timezone

  from cloudevents.http import CloudEvent
  import functions_framework


  @functions_framework.cloud_event
  def hello_pubsub(cloud_event: CloudEvent) -> None:
      """Procesa un evento Pub/Sub (CloudEvent) y escribe un log estructurado."""
      data = cloud_event.data or {}
      message = data.get("message", {}) or {}

      data_b64 = message.get("data", "") or ""
      attributes = message.get("attributes", {}) or {}
      message_id = message.get("messageId") or message.get("message_id")

      raw = ""
      if data_b64:
          try:
              raw = base64.b64decode(data_b64).decode("utf-8")
          except Exception:
              raw = ""

      event = {"raw": raw}
      if raw:
          try:
              event = json.loads(raw)
          except json.JSONDecodeError:
              event = {"raw": raw}

      event_type = "text"
      if isinstance(event, dict) and "event" in event:
          event_type = str(event.get("event"))

      out = {
          "received_at": datetime.now(timezone.utc).isoformat(),
          "pubsub_message_id": message_id,
          "attributes": attributes,
          "event_type": event_type,
          "event": event,
      }

      print(json.dumps(out, ensure_ascii=False))
  PY
  ```
  ```bash
  cat > requirements.txt <<'REQ'
  functions-framework>=3.0.0
  cloudevents>=1.0.0
  REQ
  ```

- {% include step_label.html %} Verifica que los archivos existen y que el contenido está correcto.

  > **NOTA:** Evita fallas por entrypoint inexistente o archivo incompleto.
  {: .lab-note .important .compact}

  ```bash
  ls -la | tee ../outputs/function_ls.txt
  sed -n '1,220p' main.py | tee ../outputs/main_py_preview.txt
  cat requirements.txt | tee ../outputs/requirements_preview.txt
  ```

- {% include step_label.html %} Validación rápida de sintaxis (antes de desplegar).

  > **NOTA:** Un error de sintaxis te haría fallar en Cloud Build. Detectarlo aquí ahorra tiempo.
  {: .lab-note .important .compact}

  ```bash
  python3 -m py_compile main.py && echo \"OK: sintaxis Python\" | tee ../outputs/python_compile.txt
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[3] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 5. Desplegar Cloud Run functions (Cloud Functions) con trigger Pub/Sub

> **Tiempo estimado:** 10 minutos
{: .lab-note .info .compact}

En esta tarea desplegarás la función desde la interfaz, conectándola al topic de Pub/Sub. Luego verificarás su configuración con gcloud.

#### Tarea 5.1

- {% include step_label.html %} En consola, en el buscador escribe **Cloud Run** y entra al servicio.

  {% include step_image.html %}

- {% include step_label.html %} Ve a **Services (menú izquierdo)** y en la parte superior da clic en **Write a function.**

  > **Alternativa:** Si no ves Write a function, usa Create service y en la pantalla siguiente selecciona la tarjeta Function.
  {: .lab-note .info .compact}

  {% include step_image.html %}

- {% include step_label.html %} En **Create service**, selecciona la tarjeta **Function** **Use an inline editor to create a function.**

{% include step_image.html %}

- {% include step_label.html %} En la sección **Configure**, captura los siguientes datos:

  > **NOTA:** El trigger hace que cada publicación en el topic invoque la función.
  {: .lab-note .info .compact}

  - Name: `lab09-ps-handler`
  - Region: `us-central1`
  - Endpoint URL: **Python 3.14**

  {% include step_image.html %}

- {% include step_label.html %} Continúa a la configuración de la función y en **Trigger (optional)** configura los siguientes datos:

  - Add trigger: **Pub/Sub trigger**

  {% include step_image.html %}

- {% include step_label.html %} En la ventana emergente lateral derecha, configura los siguientes datos:

  - Select a Cloud Pub/Sub topic: `lab09-events`
  - Cloud Pub/Sub needs the role roles/: Clic **Grant**

  {% include step_image.html %}

- {% include step_label.html %} Clic en **Save trigger**

  {% include step_image.html %}

- {% include step_label.html %} En **Authentication**, selecciona **Allow public access** 

  {% include step_image.html %}

- {% include step_label.html %} Luego clic en **Create**.

- {% include step_label.html %} En la pestaña de **Source**, copia y pega el contenido de python previamente validado:

  > **NOTA:** En laboratorio, inline editor acelera. En producción, preferirías repositorio/CI/CD.
  {: .lab-note .info .compact}

  - `main.py` (desde `~/labs-gcp-engineer/lab09/function/main.py`)

  ```bash
  import base64
  import json
  from datetime import datetime, timezone

  from cloudevents.http import CloudEvent
  import functions_framework


  @functions_framework.cloud_event
  def hello_pubsub(cloud_event: CloudEvent) -> None:
      """Procesa un evento Pub/Sub (CloudEvent) y escribe un log estructurado."""
      data = cloud_event.data or {}
      message = data.get("message", {}) or {}

      data_b64 = message.get("data", "") or ""
      attributes = message.get("attributes", {}) or {}
      message_id = message.get("messageId") or message.get("message_id")

      raw = ""
      if data_b64:
          try:
              raw = base64.b64decode(data_b64).decode("utf-8")
          except Exception:
              raw = ""

      event = {"raw": raw}
      if raw:
          try:
              event = json.loads(raw)
          except json.JSONDecodeError:
              event = {"raw": raw}

      event_type = "text"
      if isinstance(event, dict) and "event" in event:
          event_type = str(event.get("event"))

      out = {
          "received_at": datetime.now(timezone.utc).isoformat(),
          "pubsub_message_id": message_id,
          "attributes": attributes,
          "event_type": event_type,
          "event": event,
      }

      print(json.dumps(out, ensure_ascii=False))
  ``` 
  {% include step_image.html %} 

- {% include step_label.html %} Ahora en la misma pestaña **Source** selecciona **requirements.txt** y agrega el siguiente contenido.

  - `requirements.txt` (desde `~/labs-gcp-engineer/lab09/function/requirements.txt`)

  ```bash
  functions-framework>=3.0.0
  cloudevents>=1.0.0
  ```
  {% include step_image.html %}

- {% include step_label.html %} Ahora si clic en **Save and redeploy**.

  {% include step_image.html %}

- {% include step_label.html %} En la ventana emergente da clic en **Grant all**.

  {% include step_image.html %}

- {% include step_label.html %} Espera a que el estado sea **Active/Ready**. Toma evidencia en UI (estado verde).

  > **NOTA:** El primer despliegue puede tardar (Cloud Build + aprovisionamiento del servicio).
  {: .lab-note .warning .compact}

  {% include step_image.html %}

- {% include step_label.html %} Verifica por CLI que la función existe y guarda evidencia.

  ```bash
  gcloud run services describe "$FUNCTION_NAME" --region "$REGION" \
    --format="yaml(metadata.name,status.url,metadata.labels,metadata.annotations)" \
  | tee ../outputs/run_service_describe.yaml
  ```
  {% include step_image.html %}

- {% include step_label.html %} Lista triggers de Eventarc en la región (evidencia).

  > **NOTA:** Eventarc es el “pegamento” que enruta eventos hacia la función .
  {: .lab-note .info .compact}

  ```bash
  source scripts/env.sh
  gcloud eventarc triggers list --location "$REGION" \
    --format="table(name,eventTypes,transport.pubsub.topic,destination.cloudRun.service)" \
  | tee ../outputs/eventarc_triggers.txt
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[4] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 6. Probar el flujo end-to-end y revisar logs

> **Tiempo estimado:** 6 minutos
{: .lab-note .info .compact}

En esta tarea publicarás eventos JSON y verificarás ejecución consultando logs en UI y por CLI.

#### Tarea 6.1

- {% include step_label.html %} Publica un evento JSON desde **Cloud Shell** (con atributos).

  > **NOTA:** Este mensaje simula un evento real (“pedido creado”) y además agrega atributos para observabilidad.
  {: .lab-note .info .compact}

  ```bash
  cd ..
  source scripts/env.sh
  gcloud pubsub topics publish "$TOPIC_ID" \
    --message '{"event":"order.created","orderId":"A-1001","amount":1299.90,"currency":"MXN"}' \
    --attribute=source=cli,env=lab,type=order
  ```
  {% include step_image.html %}

- {% include step_label.html %} Publica otro mensaje desde la UI para comparar, ve a **Pub/Sub** → **Topics** → `lab09-events` → **Messages** → **Publish message**:

  > **NOTA:** En demostraciones con equipos no técnicos, UI facilita visualizar el evento.
  {: .lab-note .info .compact}

  - Message Body: `{"event":"order.created","orderId":"A-1002","amount":250,"currency":"MXN"}`
  - Messages attributes:
    - Key 1: `source`
    - Value 1: `ui`
    - Key 2: `env`
    - Value 2: `lab`
    - Key 3: `type`
    - Value 3: `order`

  {% include step_image.html %}

- {% include step_label.html %} Clic en **Publish**

- {% include step_label.html %} Abre logs por UI (ruta recomendada) ve a Cloud Run, abre `lab09-ps-handler` y clic en la pestaña **Logs**:

  > **NOTA:** En **Cloud Run Function**, la ejecución corre sobre Cloud Run; por eso los logs se ven como logs de servicio/revisión.
  {: .lab-note .info .compact}

  {% include step_image.html %}

- {% include step_label.html %} Ahora filtra por `event_type` y veras los mensajes publicados

  > **NOTA:** Puedes revisar un mensaje e ir expandiendo las opciones para detectar el body y keys previamente publicados en los mensajes.
  {: .lab-note .info .compact}

  {% include step_image.html %}

- {% include step_label.html %} **Verificación por CLI:** consulta logs recientes del servicio y guarda evidencia.

  > **NOTA:** Si no aparece salida, espera 60–120 segundos y repite. La primera invocación puede tardar en reflejar logs.
  {: .lab-note .warning .compact}

  ```bash
  source scripts/env.sh
  EVENT_TYPE="order.created"
  gcloud logging read \
    'resource.type="cloud_run_revision"
    AND resource.labels.service_name="'"$FUNCTION_NAME"'"
    AND jsonPayload.event_type="'"$EVENT_TYPE"'"' \
    --limit 25 \
    --format='table(timestamp,severity,jsonPayload.event_type,jsonPayload.pubsub_message_id)' \
  | tee outputs/logging_read.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Describe la función otra vez y confirma que sigue en estado activo.

  > **NOTA:** Si el estado no es ACTIVE/READY, revisa logs de build o errores de entrypoint.
  {: .lab-note .important .compact}

  ```bash
  source scripts/env.sh
  gcloud run services describe "$FUNCTION_NAME" --region "$REGION" \
  --format="value(status.conditions[0].status)"
  ```

  {% include step_image.html %}

- {% include step_label.html %} Verifica que el topic tiene suscripciones (incluidas las internas usadas por el trigger).

  > **NOTA:** Te ayuda a entender “quién consume” el topic (tu sub de prueba + recursos del trigger).
  {: .lab-note .info .compact}

  ```bash
  source scripts/env.sh
  gcloud pubsub topics list-subscriptions "$TOPIC_ID" \
  | tee outputs/topic_subscriptions.txt
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[5] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 7. Limpieza

> **Tiempo estimado:** 3 minutos
{: .lab-note .info .compact}

Elimina recursos para evitar costos y dejar el proyecto limpio.

#### Tarea 7.1

- {% include step_label.html %} Elimina la función desde UI (recomendado). Clic en **Cloud Run** luego en **Services** marca la casilla `lab09-ps-handler` y da clic en **Delete**.

  > **NOTA:** UI te confirma el recurso exacto a eliminar y evita borrar la función equivocada.
  {: .lab-note .important .compact}

  {% include step_image.html %}

- {% include step_label.html %} Verifica por CLI que la función ya no existe.

  > **NOTA:** Patrón de “describe + fallback” útil para scripts de limpieza.
  {: .lab-note .important .compact}

  ```bash
  source scripts/env.sh
  if gcloud run services describe "$FUNCTION_NAME" --region "$REGION" >/dev/null 2>&1; then
    echo "WARN: servicio todavía existe"
  else
    echo "OK: función eliminada"
  fi | tee outputs/cleanup_function.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Elimina la suscripción de prueba. Ve a **Pub/Sub** clic en **Subscriptions** selecciona **todas las suscripciones** y clic en **Delete**

  > **NOTA:** Evita “basura” de subs huérfanas en proyectos de laboratorio.
  {: .lab-note .important .compact}

  {% include step_image.html %}

- {% include step_label.html %} Elimina el topic. Da clic en **Topics** y luego selecciona **lab09-events** da clic en **Delete** escribe `delete` y clic nuevamente en **Delete**

  {% include step_image.html %}

- {% include step_label.html %} Verificación final: confirma que ya no existe el topic.

  > **NOTA:** Confirma limpieza completa y evita costos residuales por recursos olvidados.
  {: .lab-note .important .compact}

  ```bash
  source scripts/env.sh
  if gcloud pubsub topics describe "$TOPIC_ID" >/dev/null 2>&1; then
    echo "WARN: topic todavía existe"
  else
    echo "OK: topic eliminado"
  fi | tee outputs/cleanup_topic.txt
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[6] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}