---
layout: lab
title: "Práctica 11: Crear tablero en Monitoring con métricas de CPU y alertas"
permalink: /lab11/lab11/
images_base: /labs/lab11/img
duration: "40 minutos"
objective:
  - "Crear un recurso monitoreable (VM en Compute Engine), construir un **Dashboard** en **Cloud Monitoring** con métricas de **CPU**, y configurar una **alerta** (Alerting Policy) con canal de notificación, validando la alerta al generar carga de CPU y revisando incidentes."
prerequisites:
  - "Proyecto de Google Cloud con **facturación habilitada**."
  - "Permisos para: **Compute Admin** (crear VM), **Monitoring Admin** (dashboards/alertas) y acceso a **Cloud Shell**."
  - "Acceso a **Google Cloud Console** y a **Cloud Shell (gcloud)**."
introduction: |
  **Cloud Monitoring** (Google Cloud Observability) permite visualizar métricas, crear tableros (dashboards) y generar alertas cuando una condición se cumple (por ejemplo, CPU alta). 

  En esta práctica crearás una VM sencilla, instalarás una herramienta para generar carga, construirás un dashboard con la métrica **CPU utilization**, y configurarás una alerta basada en CPU con un canal de notificación (email). Finalmente provocarás la condición para verificar que la alerta dispara y queda registrada como incidente.
slug: lab11
lab_number: 11
final_result: >
  Al finalizar, tendrás una VM monitoreada, un dashboard con gráficos de CPU y una política de alertamiento que se activa cuando la CPU supera el umbral definido durante un periodo, con evidencia de la alerta (incidente) en Cloud Monitoring.
notes:
  - "Para que una alerta sea útil, define: **umbral**, **ventana de evaluación** y **duración** (evita falsos positivos)."
  - "Para laboratorio, puedes usar un umbral bajo y una ventana corta para disparar rápido; en producción se recomienda una evaluación más conservadora."
  - "**Costos:** VM (Compute Engine), discos y potencialmente costos asociados a observabilidad según el uso/retención. Elimina recursos al terminar."
references:
  - text: "Cloud Monitoring: Overview"
    url: https://cloud.google.com/monitoring/docs
  - text: "Dashboards en Cloud Monitoring"
    url: https://cloud.google.com/monitoring/dashboards
  - text: "Alerting: crear y administrar políticas"
    url: https://cloud.google.com/monitoring/alerts
  - text: "Métricas de Compute Engine (CPU utilization)"
    url: https://cloud.google.com/monitoring/api/metrics_gcp#gcp-compute
  - text: "Compute Engine pricing"
    url: https://cloud.google.com/compute/all-pricing
prev: /lab10/lab10/
next: /lab12/lab12/
---

---

### Tarea 1. Preparar el entorno y variables del laboratorio

> **Tiempo estimado:** 4 minutos
{: .lab-note .info .compact}

En esta tarea abrirás Cloud Shell, validarás el proyecto y crearás una carpeta independiente para el laboratorio con variables estándar (nombre de VM, zona y nombres de dashboard/alerta).

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

- {% include step_label.html %} Crea la carpeta del laboratorio 11 y subcarpetas estándar.

  > **NOTA:** Cada práctica en su carpeta = evidencias y comandos reproducibles.
  {: .lab-note .info .compact}

  ```bash
  cd ~
  mkdir -p labs-gcp-engineer/lab11/{scripts,outputs}
  ```
  ```bash
  cd labs-gcp-engineer/lab11
  ```
  {% include step_image.html %}

- {% include step_label.html %} Crea `scripts/env.sh` con variables del laboratorio y cárgalas.

  > **Nota:** Ajusta región/zona si tu organización lo requiere.
  {: .lab-note .info .compact}

  > **NOTA:** Estandariza nombres y simplifica comandos.
  {: .lab-note .info .compact}

  ```bash
  cat > scripts/env.sh <<'EOF'
  export REGION="us-central1"
  export ZONE="us-central1-a"

  export VM_NAME="lab11-mon-vm"
  export VM_TYPE="e2-micro"

  export DASHBOARD_NAME="LAB11 - CPU Dashboard"
  export ALERT_POLICY_NAME="LAB11 - High CPU Alert"
  EOF
  ```
  ```bash
  source scripts/env.sh
  echo "REGION=$REGION ZONE=$ZONE VM_NAME=$VM_NAME"
  ```

- {% include step_label.html %} Valida estructura y guarda evidencia.

  > **NOTA:** Evidencia básica para auditoría y soporte.
  {: .lab-note .info .compact}

  ```bash
  pwd | tee outputs/pwd.txt
  ```
  ```bash
  ls -la | tee outputs/ls_root.txt
  ```
  ```bash
  ls -la scripts outputs | tee outputs/ls_subfolders.txt
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[0] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 2. Crear VM desde cero y generar carga de CPU

> **Tiempo estimado:** 12 minutos
{: .lab-note .info .compact}

En esta tarea crearás una VM pequeña, habilitarás acceso por SSH, instalarás una utilidad para generar carga (CPU stress) y validarás que la métrica de CPU cambia (lo cual facilita comprobar dashboard y alertas).

#### Tarea 2.1

- {% include step_label.html %} En la consola, ve a **Compute Engine** luego **VM instances**.

  > **NOTA:** Monitoring de CPU se apoya en métricas del recurso. Una VM es el ejemplo más directo.
  {: .lab-note .info .compact}

  {% include step_image.html %}

- {% include step_label.html %} Clic en **Create instance**

  {% include step_image.html %}

- {% include step_label.html %} Configura los siguientes datos:

  > **NOTA:** Para laboratorio, una VM pequeña reduce costos y acelera el despliegue.
  {: .lab-note .info .compact}

  - Name: `lab11-mon-vm`
  - Region: **us-central1**
  - Zone: **us-central1-a**

  {% include step_image.html %}

  - Machine type: `e2-medium`
  
  {% include step_image.html %}
  
  - Boot disk: **Debian GNU/Linux 12 (bookworm)**

  {% include step_image.html %}

  - Networking/Firewall: marca **Allow HTTP traffic**
  - Network tags: `ssh-server`

  {% include step_image.html %}

- {% include step_label.html %} Clic en **Create**

- {% include step_label.html %} Verifica por CLI que la VM está `RUNNING` y guarda evidencia.

  > **NOTA:** Confirma que se creó en la zona correcta y que está activa.
  {: .lab-note .info .compact}

  ```bash
  source scripts/env.sh
  gcloud compute instances describe "$VM_NAME" --zone "$ZONE" --format="yaml(name,zone,status,machineType.basename(),disks[0].source.basename())" | tee outputs/vm_describe.yaml
  ```
  {% include step_image.html %}

- {% include step_label.html %} Conéctate por **SSH** a la VM **lab11-mon-vm** desde la UI.

  {% include step_image.html %}

- {% include step_label.html %} Da clic en el botón **Authorize**.

  {% include step_image.html %}

- {% include step_label.html %} Dentro de la VM, instala `stress-ng` y valida instalación.

  > **NOTA:** `stress-ng` permite elevar CPU de forma controlada para probar dashboards/alertas.
  {: .lab-note .info .compact}

  ```bash
  sudo apt-get update
  ```
  ```bash
  sudo apt-get install -y stress-ng
  ```
  ```bash
  stress-ng --version
  ```
  {% include step_image.html %}

- {% include step_label.html %} En la VM, inicia una carga de CPU por ~8 minutos (no cierres la sesión).

  > **NOTA:** Una carga sostenida facilita que el umbral se cumpla durante la ventana de evaluación.
  {: .lab-note .info .compact}

  ```bash
  stress-ng --cpu 1 --cpu-method matrixprod --timeout 8m --metrics-brief
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[1] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 3. Crear dashboard en Cloud Monitoring con métricas de CPU

> **Tiempo estimado:** 10 minutos
{: .lab-note .info .compact}

En esta tarea crearás un Dashboard (tablero) con gráficos de CPU para la VM. Usarás la UI para construir el gráfico y confirmarás que se visualiza la instancia correcta.

#### Tarea 3.1

- {% include step_label.html %} En la consola, abre **Monitoring** luego **Dashboards**.

  {% include step_image.html %}

- {% include step_label.html %} Clic en **Create Custom Dashboard**

  > **NOTA:** Un dashboard es una vista consolidada de métricas clave para operación.
  {: .lab-note .info .compact}

  {% include step_image.html %}

- {% include step_label.html %} Primero da clic en el nombre predeterminado para el dashboard.

  {% include step_image.html %}

- {% include step_label.html %} Asigna el nombre del dashboard: **`LAB11 - CPU Dashboard`** y da clic en cualquier parte afuera del panel de edición del nombre.

  > **NOTA:** Un nombre claro ayuda a operaciones y a auditorías.
  {: .lab-note .info .compact}

  {% include step_image.html %}

- {% include step_label.html %} Da clie en el botón **Add widget**

  {% include step_image.html %}

- {% include step_label.html %} Agrega un widget **Line chart**.

  {% include step_image.html %}

- {% include step_label.html %} Selecciona la métrica de CPU en el filtro.

  > **NOTA:** CPU utilization es un ratio (0.0 a 1.0). En UI suele mostrarse como porcentaje.
  {: .lab-note .info .compact}

  - Resource type: **VM Instance**
  - Metric: **CPU utilization** (`compute.googleapis.com/instance/cpu/utilization`)
  - Clic en **Apply**

  {% include step_image.html %}

  - Filter: `instance_name = lab11-mon-vm`
  - Aggregation: **Mean** by **None**
  - Clic en el simbolor **+**, selecciona **Min interval** = **1m**

  {% include step_image.html %}

- {% include step_label.html %} Ahora clic en el botón **Apply**

  {% include step_image.html %}

- {% include step_label.html %} Verifica que el dashboard muestra variación/pico de CPU durante la prueba. Puede llegar hasta el **100%**

  > **Nota:** Ajusta el rango a “Last 30 minutes” y espera 1–2 minutos si no ves datos.
  {: .lab-note .info .compact}

  > **NOTA:** Puede existir una latencia breve entre la ejecución y la visualización de métricas.
  {: .lab-note .info .compact}

  {% include step_image.html %}

- {% include step_label.html %} **Desde Cloud Shell**: registra el nombre del dashboard.

  > **NOTA:** Evidencia textual complementa capturas de UI.
  {: .lab-note .info .compact}

  ```bash
  source scripts/env.sh
  echo "Dashboard creado: ${DASHBOARD_NAME} (Monitoring > Dashboards)" | tee outputs/dashboard_evidence.txt
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[2] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 4. Crear alerta de CPU alta con canal de notificación y validar incidente

> **Tiempo estimado:** 11 minutos
{: .lab-note .info .compact}

En esta tarea crearás un canal de notificación (email) y una política de alerta que dispare cuando la CPU supere un umbral durante un periodo. Luego validarás que se genera un incidente.

#### Tarea 4.1

- {% include step_label.html %} En la consola, abre **Monitoring** luego en **Alerting**.

  > **NOTA:** Alerting genera incidentes cuando una condición de métrica se cumple.
  {: .lab-note .info .compact}

  {% include step_image.html %}

- {% include step_label.html %} Da clic en el botón **Edit notification channels**

  {% include step_image.html %}

- {% include step_label.html %} En la sección de **Email** da clic en el botón **Add new**

  {% include step_image.html %}

- {% include step_label.html %} En la ventana emergente coloca tu correo (personal donde puedas recibier mensajes)

  > **NOTA:** Email es simple en laboratorio y no requiere herramientas de terceros.
  {: .lab-note .info .compact}

  - Email Address: **Tu correo**
  - Display Name: `CPU Monitoring Alert`

  {% include step_image.html %}

- {% include step_label.html %} Clic en el botón **Save**

- {% include step_label.html %} De vuelta en **Alerting** crea una **Alert policy**, da clic en **+ Create policy**

  {% include step_image.html %}

- {% include step_label.html %} Primero selecciona el modo **Builder** y luego la metrica:

  - Metric: **`CPU utilization`**
  - Clic en **Apply**

  {% include step_image.html %}

- {% include step_label.html %} Ahora agrega el filtro de la maquina virtual. Clic en **Add filter**

  > **NOTA:** Filtrar por instancia evita alertas disparadas por otras VMs del proyecto.
  {: .lab-note .info .compact}

  - Filter: `instance_name`**=**`lab11-mon-vm`
  - Clic en **Done**

  {% include step_image.html %}

- {% include step_label.html %} Clic en **Next**

- {% include step_label.html %} Configura los siguientes datos en la sección **Configure alert trigger**

  > **NOTA:** Umbral bajo + ventana corta = disparo rápido. En producción, usa umbrales mayores y ventanas más largas.
  {: .lab-note .info .compact}

  - Condition type: **Threshold**
  - Alert trigger: **Any time series violates**
  - Threshold position: **Above threshold**
  - Threshold value: `50`

  {% include step_image.html %}

- {% include step_label.html %} Clic en **Next**

- {% include step_label.html %} En la sección **Configure notifications and finalize alert**, define:

  > **NOTA:** Documentación acelera respuesta operativa.
  {: .lab-note .info .compact}

  - Notification channels: selecciona el email configurado
  - Notification subject line: `Monitoring`

  {% include step_image.html %}

  - Name the alert policy: `LAB11 - High CPU Alert`

  {% include step_image.html %}

- {% include step_label.html %} Clic en **Next**

- {% include step_label.html %} Clic en **Create policy**

  {% include step_image.html %}

- {% include step_label.html %} Valida incidente del incremento del CPU, puede aparecer vacio.

- {% include step_label.html %} Clic en **Alerting** luego en **Incidents**

  {% include step_image.html %}

- {% include step_label.html %} Si necesitas re-disparar: en la VM ejecuta otra carga por 6 minutos.

  > **NOTA:** Si no dispara, vuelve a ejecutar carga CPU en la VM y/o baja el umbral.
  {: .lab-note .info .compact}

  - Espera a que aparezca un incidente activo (1–3 min)

  ```bash
  stress-ng --cpu 1 --cpu-method matrixprod --timeout 6m --metrics-brief
  ```
  {% include step_image.html %}

- {% include step_label.html %} Vuelve a dar clic en **Alerting** luego en **Incidents**

  > **NOTA:** Es normal que tarde varios minutos en aparecer la alerta.
  {: .lab-note .info .compact}

  {% include step_image.html %}

- {% include step_label.html %} Puedes observar tu correo y detectar el mensaje de la alerta

  {% include step_image.html %}

- {% include step_label.html %} **Desde Cloud Shell:** registra que la alerta fue creada y dónde validarla.

  ```bash
  source scripts/env.sh
  echo "Alerta creada: ${ALERT_POLICY_NAME}. Validar en Monitoring > Alerting > Incidents." | tee outputs/alert_evidence.txt
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[3] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 5. Limpieza

> **Tiempo estimado:** 3 minutos
{: .lab-note .info .compact}

En esta tarea eliminarás la VM y la política de alertas y el dashboard, evitando costos residuales. Validarás que la VM ya no existe por CLI.

#### Tarea 5.1

- {% include step_label.html %} En UI: elimina la VM:

  > **NOTA:** La VM es el principal costo en esta práctica.
  {: .lab-note .info .compact}

  - **Compute Engine** luego **VM instances** selecciona `lab11-mon-vm` clic en **Delete**

  {% include step_image.html %}

- {% include step_label.html %} En UI: elimina/inhabilita la **política de alerta.**

  > **NOTA:** En laboratorio normalmente se limpia; en producción podrías conservar como plantilla.
  {: .lab-note .info .compact}

  - **Monitoring** luego **Alerting**
  - Pestaña **Policies**
  - Selecciona LAB11 - High CPU Alert
  - Clic en **Delete**

  {% include step_image.html %}

- {% include step_label.html %} En UI: **elimina el dashboard.**

  - **Monitoring** luego **Dashboards**
  - Selecciona **LAB11 - CPU Dashboard**
  - Menú (⋮) clic en **Delete**

  {% include step_image.html %}

- {% include step_label.html %} Verifica por CLI que la **política de alerta** ya no existe

  ```bash
  source scripts/env.sh
  if gcloud alpha monitoring policies list --filter='displayName="LAB11 - High CPU Alert"' \
    --format="value(name)" | grep -q .; then
    echo "WARN: alerta todavía existe"
  else
    echo "OK: alerta eliminada"
  fi | tee outputs/cleanup_alert_policy.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica por CLI que el **dashboard** ya no existe

  ```bash
  source scripts/env.sh
  if gcloud monitoring dashboards list --format="value(displayName)" | grep -qx "LAB11 - CPU Dashboard"; then
    echo "WARN: dashboard todavía existe"
  else
    echo "OK: dashboard eliminado"
  fi | tee outputs/cleanup_dashboard.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica por CLI que la **VM ya no existe.**

  ```bash
  source scripts/env.sh
  if gcloud compute instances describe "$VM_NAME" --zone "$ZONE" >/dev/null 2>&1; then
    echo "WARN: VM todavía existe"
  else
    echo "OK: VM eliminada"
  fi | tee outputs/cleanup_vm.txt
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[4] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}