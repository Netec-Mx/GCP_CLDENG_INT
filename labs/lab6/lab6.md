---
layout: lab
title: "Práctica 6: Crear grupo administrado con autoescalado y health checks"
permalink: /lab6/lab6/
images_base: /labs/lab6/img
duration: "45 minutos"
objective:
  - "Crear un **Managed Instance Group (MIG)** en **Compute Engine** a partir de un **Instance Template**, habilitando **autoscaling** por CPU y **autohealing** basado en **HTTP Health Checks**, accediendo de forma segura a instancias **sin IP pública** mediante **IAP TCP forwarding/SSH**, y validando comportamiento de escalado y recuperación."
prerequisites:
  - "Proyecto de Google Cloud con permisos para: **VPC**, **Firewall**, **Compute Engine**, **Health checks** e **IAM/IAP**."
  - "Acceso a **Google Cloud Console** y **Cloud Shell**."
  - "Navegador con sesión activa en Google."
introduction: |
  Un **Managed Instance Group (MIG)** administra un conjunto de VMs homogéneas creadas desde un **Instance Template**.
  - **Autoscaling** ajusta el número de instancias según una métrica (por ejemplo, CPU objetivo).
  - **Autohealing** recrea automáticamente instancias que fallen un **Health Check** (por ejemplo, HTTP `/healthz`).
  En esta práctica crearás una VPC dedicada, un template que instala **Nginx** y publica una página con estilo + endpoint `/healthz`, un Health Check HTTP, y un MIG con autoscaling y autohealing. Probarás:
  - 1) acceso seguro por **IAP** a VMs sin IP pública,  
  - 2) escalado al inducir CPU alta,  
  - 3) recreación al forzar fallo de health check.
slug: lab6
lab_number: 6
final_result: >
  Al finalizar, tendrás una VPC del laboratorio, un Instance Template reproducible, un Health Check HTTP, un Managed Instance Group con autoscaling (min/max) y autohealing, y evidencia de que el grupo escala y reemplaza instancias no saludables. Accederás a la app por IAP sin exponer IP pública.
notes:
  - "**Costos:** Compute Engine cobra por VM (vCPU/RAM) y disco. El autoscaling puede crear más VMs (más costo). Health checks no suelen tener costo directo, pero el tráfico/egress puede aplicar. Elimina recursos al final."
  - "Este laboratorio usa VMs **sin IP pública** y acceso por **IAP**, por lo que debes permitir el rango de IAP en firewall."
  - "Autohealing y autoscaling no son instantáneos: se basan en ventanas/métricas y pueden tardar minutos en reflejarse."
references:
  - text: "Managed instance groups (MIG): conceptos y administración"
    url: https://cloud.google.com/compute/docs/instance-groups
  - text: "Autoscaling para MIG"
    url: https://cloud.google.com/compute/docs/autoscaler
  - text: "Autohealing en MIG usando health checks"
    url: https://cloud.google.com/compute/docs/instance-groups/autohealing-instances-in-migs
  - text: "Health checks HTTP en Compute Engine"
    url: https://cloud.google.com/load-balancing/docs/health-checks
  - text: "IAP TCP forwarding / acceso a VMs sin IP pública"
    url: https://cloud.google.com/iap/docs/using-tcp-forwarding
  - text: "Precios de Compute Engine"
    url: https://cloud.google.com/compute/all-pricing
prev: /lab5/lab5/
next: /lab7/lab7/
---

---

## Instrucciones generales

- Esta práctica es **mayormente por interfaz gráfica (UI)**:
  - Crear VPC/subnet/firewall, instance template, health check y MIG (autoscaling/autohealing).
- Usaremos **Cloud Shell** para:
  - crear estructura de carpetas, definir variables,
  - validar recursos con `gcloud`,
  - conectarnos por **IAP** a instancias del MIG para pruebas (sin IP pública).
- Carpeta por práctica en Cloud Shell:
  - `~/labs-gcp-engineer/lab06/` con `scripts/` y `outputs/`.

---

### Tarea 1. Preparar Cloud Shell y variables del laboratorio (Tiempo estimado: 5 min)

En esta tarea configurarás el contexto (proyecto/región/zona) y crearás la estructura de carpetas y variables.

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

- {% include step_label.html %} Crea la carpeta del laboratorio 6.

  ```bash
  cd ~
  mkdir -p labs-gcp-engineer/lab06/{scripts,outputs}
  ```
  ```bash
  cd labs-gcp-engineer/lab06
  ```
  {% include step_image.html %}

- {% include step_label.html %} Crea `scripts/env.sh` con nombres y variables alineadas a **Práctica 6**.

  ```bash
  cat > scripts/env.sh <<'EOF'
  # Región / zona
  export REGION="us-central1"
  export ZONE="us-central1-a"

  # Red del lab
  export VPC_NAME="lab06-vpc"
  export SUBNET_NAME="lab06-subnet-app"
  export SUBNET_CIDR="10.60.10.0/24"

  # Firewall tags
  export TAG_IAP="lab06-iap"
  export TAG_WEB="lab06-web"

  # Recursos Compute
  export TEMPLATE_NAME="lab06-tpl-web"
  export MIG_NAME="lab06-mig-web"

  # Health check
  export HC_NAME="lab06-hc-http"
  export HC_PORT="80"
  export HC_PATH="/healthz"
  EOF
  ```
  ```bash
  source scripts/env.sh
  ```

- {% include step_label.html %} Configura región/zona por defecto y guarda evidencia.

  ```bash
  gcloud config set compute/region "$REGION"
  ```
  ```bash
  gcloud config set compute/zone "$ZONE"
  ```
  ```bash
  gcloud config list --format="yaml(core.project,compute.region,compute.zone)" | tee outputs/gcloud_config.yaml
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[0] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 2. Crear VPC + subnet + firewall para IAP y Health Checks (Tiempo estimado: 6 min)

En esta tarea crearás una VPC dedicada con una subnet. Luego, reglas de firewall para:
- SSH por IAP (tcp:22) hacia instancias sin IP pública,
- HTTP por IAP (tcp:80) para ver la página web con túnel,
- HTTP desde rangos de **Health Checks** para autohealing.

> **Nota:** En un entorno real, segmentarías reglas y fuentes con mayor precisión; aquí lo mantenemos didáctico.
{: .lab-note .info .compact}

#### Tarea 2.1 (UI) — Crear VPC y subnet

- {% include step_label.html %} En la consola: **VPC network** y luego **VPC networks**

  {% include step_image.html %}

- {% include step_label.html %} Da clic en la opción **Create VPC network**.

  > **NOTA:** La VPC es el contenedor de subredes, firewall rules y rutas.
  {: .lab-note .info .compact}

  {% include step_image.html %}

- {% include step_label.html %} Configura los siguientes datos:

  > **NOTA:** Este rango será el “espacio” de IPs privadas de las VMs del MIG.
  {: .lab-note .info .compact}

  - Name: `lab06-vpc`
  - Subnet creation mode: **Custom**

  {% include step_image.html %}

- {% include step_label.html %} Agrega una subnet:

  - Subnet name: `lab06-subnet-app`
  - Region: `us-central1 (Iowa)`
  - IPv4 range: `10.60.10.0/24`
  - Private Google Access: **ON**
  - Clic en **Done**

  {% include step_image.html %}

- {% include step_label.html %} Clic en **Create**.

- {% include step_label.html %} Valida por CLI la VPC creada.

  ```bash
  source scripts/env.sh
  gcloud compute networks describe "$VPC_NAME" --format="yaml(name,autoCreateSubnetworks)" | tee outputs/vpc.yaml
  ```
  ```bash
  gcloud compute networks subnets describe "$SUBNET_NAME" --region "$REGION" --format="yaml(name,ipCidrRange,privateIpGoogleAccess,network)" | tee outputs/subnet.yaml
  ```
  {% include step_image.html %}

#### Tarea 2.2 (UI) — Firewall para IAP (SSH y HTTP)

- {% include step_label.html %} En consola da clic en: **VPC network** y luego **Firewall**. Si ya estas en el menú busca solo **Firewall**

  {% include step_image.html %}

- {% include step_label.html %} Ahora da clic en **Create firewall rule**.

  {% include step_image.html %}

- {% include step_label.html %} Configura los siguientes datos:

  > **NOTA:** En vez de exponer SSH públicamente, lo consumes por IAP.
  {: .lab-note .info .compact}

  - Name: `lab06-fw-allow-iap-ssh`
  - Network: `lab06-vpc`
  - Direction of traffic: **Ingress**

  {% include step_image.html %}

  - Targets: **Specified target tags**
  - Target tags: `lab06-iap`
  - Source IPv4 ranges: `35.235.240.0/20`
  - Protocols/ports: `tcp:22`

  {% include step_image.html %}

- {% include step_label.html %} Clic en el botón **Create** al final de la pagina.

- {% include step_label.html %} Crea una segunda regla (IAP HTTP) para acceder al puerto 80 vía túnel.

  - Name: `lab06-fw-allow-iap-http`
  - Network: `lab06-vpc`
  - Direction of traffic: **Ingress**

  {% include step_image.html %}

  - Targets: **Specified target tags**
  - Target tags: `lab06-web`
  - Source IPv4 ranges: `35.235.240.0/20`
  - Protocols/ports: `tcp:80`

  {% include step_image.html %}

- {% include step_label.html %} Clic en el botón **Create** al final de la pagina.

- {% include step_label.html %} Valida las reglas por CLI.

  > **NOTA:** Aseguras que las reglas apuntan a tags y no a “todas las instancias”.
  {: .lab-note .info .compact}

  ```bash
  gcloud compute firewall-rules describe lab06-fw-allow-iap-ssh \
    --format="yaml(name,network,sourceRanges,allowed,targetTags)" | tee outputs/fw_iap_ssh.yaml
  ```
  ```bash
  gcloud compute firewall-rules describe lab06-fw-allow-iap-http \
    --format="yaml(name,network,sourceRanges,allowed,targetTags)" | tee outputs/fw_iap_http.yaml
  ```
  {% include step_image.html %}

#### Tarea 2.3 (UI) — Firewall para Health Checks (autohealing)

- {% include step_label.html %} Crea **una regla mas** para permitir probes del health check hacia el puerto 80.

  > **NOTA:** Los health checkers de Google necesitan llegar a tu endpoint `/healthz` para evaluar salud.
  {: .lab-note .info .compact}

  - Name: `lab06-fw-allow-hc-http`
  - Network: `lab06-vpc`
  - Direction of traffic: **Ingress**

  {% include step_image.html %}

  - Targets: **Specified target tags**
  - Target tags: `lab06-web`
  - Source IPv4 ranges: `130.211.0.0/22,35.191.0.0/16`
  - Protocols/ports: `tcp:80`
  
  {% include step_image.html %}

- {% include step_label.html %} Clic en el botón **Create** al final de la pagina.

- {% include step_label.html %} Valida la regla creada por CLI.

  ```bash
  gcloud compute firewall-rules describe lab06-fw-allow-hc-http \
    --format="yaml(name,network,sourceRanges,allowed,targetTags)" | tee outputs/fw_hc_http.yaml
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[1] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 3. Crear Instance Template con startup script (Tiempo estimado: 10 min)

En esta tarea crearás un **Instance Template** que instala Nginx, publica una página con estilo y expone `/healthz`.

#### Tarea 3.1 (Cloud Shell) — Crear startup script como archivo (reproducible)

- {% include step_label.html %} Crea `scripts/startup.sh` para instalar Nginx y publicar `index.html` + `healthz`.

  > **NOTA:** Guardar el startup script en archivo facilita versionarlo y reutilizarlo.
  {: .lab-note .info .compact}

  ```bash
  cd ~/labs-gcp-engineer/lab06
  cat > scripts/startup.sh <<'EOF'
  #!/bin/bash
  set -euo pipefail

  apt-get update -y
  apt-get install -y nginx curl

  INSTANCE_NAME="$(curl -fsH 'Metadata-Flavor: Google' http://metadata.google.internal/computeMetadata/v1/instance/name || hostname)"
  ZONE_FULL="$(curl -fsH 'Metadata-Flavor: Google' http://metadata.google.internal/computeMetadata/v1/instance/zone || true)"
  ZONE="${ZONE_FULL##*/}"

  cat > /var/www/html/index.html <<HTML
  <!doctype html>
  <html>
    <head>
      <meta charset="utf-8"/>
      <title>Lab06 - MIG Web</title>
      <style>
        body { font-family: system-ui, -apple-system, Segoe UI, Roboto, Arial; background: #0b1220; color: #e6eefc; padding: 28px; }
        .wrap { max-width: 980px; margin: 0 auto; }
        .card { background: rgba(255,255,255,0.06); border: 1px solid rgba(255,255,255,0.12); border-radius: 16px; padding: 18px; box-shadow: 0 10px 28px rgba(0,0,0,0.35); }
        h1 { margin: 0 0 10px; font-size: 26px; }
        .pill { display:inline-block; padding: 6px 10px; border-radius: 999px; background: rgba(59,130,246,0.20); border: 1px solid rgba(59,130,246,0.40); }
        table { width: 100%; border-collapse: collapse; margin-top: 12px; overflow: hidden; border-radius: 12px; }
        th, td { padding: 10px 12px; border-bottom: 1px solid rgba(255,255,255,0.10); font-size: 14px; }
        th { text-align: left; background: rgba(255,255,255,0.05); }
        code { background: rgba(255,255,255,0.08); padding: 2px 6px; border-radius: 6px; }
      </style>
    </head>
    <body>
      <div class="wrap">
        <div class="card">
          <h1>Lab06: Managed Instance Group</h1>
          <p class="pill">Nginx OK</p>
          <table>
            <tr><th>Instance</th><td><code>${INSTANCE_NAME}</code></td></tr>
            <tr><th>Zone</th><td><code>${ZONE}</code></td></tr>
            <tr><th>Health</th><td><code>/healthz</code> returns 200</td></tr>
            <tr><th>Timestamp (UTC)</th><td><code>$(date -u +%FT%TZ)</code></td></tr>
          </table>
          <p style="opacity:.85;margin-top:10px;">Este sitio es generado por startup script del Instance Template.</p>
        </div>
      </div>
    </body>
  </html>
  HTML

  echo "ok" > /var/www/html/healthz

  nginx -t
  systemctl enable --now nginx
  systemctl restart nginx
  EOF

  chmod +x scripts/startup.sh
  sed -n '1,120p' scripts/startup.sh | tee outputs/startup_preview.txt
  ```

#### Tarea 3.2 (UI) — Crear el Instance Template

- {% include step_label.html %} En consola: da clic en **Compute Engine** luego **Instance templates**.

  {% include step_image.html %}

- {% include step_label.html %} Da clic en **Create instance template**.

  > **NOTA:** El template define “cómo se ven” todas las instancias del MIG.
  {: .lab-note .info .compact}

  {% include step_image.html %}

- {% include step_label.html %} Configura los siguientes datos:

  - Name: `lab06-tpl-web`
  - Location: **Regional**
  - Region: **us-central1 (iowa)**

  {% include step_image.html %}

  - Machine type: **e2-medium**

  {% include step_image.html %}

  - Boot disk: **Debian 12**

  {% include step_image.html %}

  - Advanced options/Networking/Network tags
    - `lab06-iap`
    - `lab06-web`

    {% include step_image.html %}

  - Advanced options/Networking/Network interfaces
    - Network: **lab06-vpc**
    - Subnetwork: **lab06-subnet-app**
    - External IPv4: **None**
    - Clic en **Done**

  {% include step_image.html %}

- {% include step_label.html %} En la sección **Advanced options** luego **Management** da clic en **Startup script**, pega el siguiente script:

  > **NOTA:** Es el mismo que guardaste en los pasos anteriores.
  {: .lab-note .info .compact}

  ```bash
  #!/bin/bash
  set -euo pipefail
  export DEBIAN_FRONTEND=noninteractive

  # (Opcional pero recomendado) log para depurar
  exec > >(tee -a /var/log/startup-script.log) 2>&1

  apt-get update -y
  apt-get install -y nginx curl

  INSTANCE_NAME="$(curl -fsS -H 'Metadata-Flavor: Google' \
    http://metadata.google.internal/computeMetadata/v1/instance/name || hostname)"

  ZONE_FULL="$(curl -fsS -H 'Metadata-Flavor: Google' \
    http://metadata.google.internal/computeMetadata/v1/instance/zone || true)"
  ZONE="${ZONE_FULL##*/}"

  cat > /var/www/html/index.html <<HTML
  <!doctype html>
  <html>
    <head>
      <meta charset="utf-8"/>
      <title>Lab06 - MIG Web</title>
      <style>
        body { font-family: system-ui, -apple-system, Segoe UI, Roboto, Arial; background: #0b1220; color: #e6eefc; padding: 28px; }
        .wrap { max-width: 980px; margin: 0 auto; }
        .card { background: rgba(255,255,255,0.06); border: 1px solid rgba(255,255,255,0.12); border-radius: 16px; padding: 18px; box-shadow: 0 10px 28px rgba(0,0,0,0.35); }
        h1 { margin: 0 0 10px; font-size: 26px; }
        .pill { display:inline-block; padding: 6px 10px; border-radius: 999px; background: rgba(59,130,246,0.20); border: 1px solid rgba(59,130,246,0.40); }
        table { width: 100%; border-collapse: collapse; margin-top: 12px; overflow: hidden; border-radius: 12px; }
        th, td { padding: 10px 12px; border-bottom: 1px solid rgba(255,255,255,0.10); font-size: 14px; }
        th { text-align: left; background: rgba(255,255,255,0.05); }
        code { background: rgba(255,255,255,0.08); padding: 2px 6px; border-radius: 6px; }
      </style>
    </head>
    <body>
      <div class="wrap">
        <div class="card">
          <h1>Lab06: Managed Instance Group</h1>
          <p class="pill">Nginx OK</p>
          <table>
            <tr><th>Instance</th><td><code>${INSTANCE_NAME}</code></td></tr>
            <tr><th>Zone</th><td><code>${ZONE}</code></td></tr>
            <tr><th>Health</th><td><code>/healthz</code> returns 200</td></tr>
            <tr><th>Timestamp (UTC)</th><td><code>$(date -u +%FT%TZ)</code></td></tr>
          </table>
          <p style="opacity:.85;margin-top:10px;">Este sitio es generado por startup script del Instance Template.</p>
        </div>
      </div>
    </body>
  </html>
  HTML

  echo "ok" > /var/www/html/healthz

  nginx -t
  systemctl enable --now nginx
  systemctl restart nginx
  ```
  {% include step_image.html %}

- {% include step_label.html %} Clic en **Create**.

- {% include step_label.html %} Valida el template por CLI.

  > **NOTA:** Confirmas tags, red/subred y que el startup script se guardó como metadata.
  {: .lab-note .info .compact}

  ```bash
  source scripts/env.sh
  gcloud compute instance-templates list \
    --filter="name=($TEMPLATE_NAME)" \
    --format="table(name,region,creationTimestamp)" \
    | tee outputs/template.yaml
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[2] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 4. Crear Health Check HTTP (Tiempo estimado: 4 min)

En esta tarea crearás un Health Check HTTP que consulta `/healthz` en el puerto 80.

#### Tarea 4.1 (UI) — Crear health check

- {% include step_label.html %} Primero **Dentro de Cloud Shell** crea un Cloud NAT temporal para la descarga de dependencias de la practica, ejecuta todo el codigo sisguiente

  ```bash
  REGION="us-central1"
  NETWORK="lab06-vpc"
  ROUTER_NAME="nat-router"
  NAT_NAME="nat-config"

  gcloud compute routers create "$ROUTER_NAME" \
    --region="$REGION" \
    --network="$NETWORK" || true

  gcloud compute routers nats create "$NAT_NAME" \
    --router="$ROUTER_NAME" \
    --region="$REGION" \
    --nat-all-subnet-ip-ranges \
    --auto-allocate-nat-external-ips
  ```

- {% include step_label.html %} En consola: **Compute Engine** luego **Health checks**.

  {% include step_image.html %}

- {% include step_label.html %} Clic en **Create health check**.

  > **NOTA:** Autohealing usa este health check para decidir cuándo reemplazar una instancia.
  {: .lab-note .info .compact}

  {% include step_image.html %}

- {% include step_label.html %} Configura los siguientes datos:

  > **NOTA:** `/healthz` es un patrón estándar para pruebas de salud.
  {: .lab-note .info .compact}

  - Name: `lab06-hc-http`
  - Scope: **Global**
  - Protocol: **HTTP**
  - Port: `80`
  - Request path: `/healthz`

  {% include step_image.html %}

- {% include step_label.html %} Clic en **Create**.

- {% include step_label.html %} Valida por CLI el health check creado.

  > **NOTA:** Confirmas puerto y path exactos que usará autohealing.
  {: .lab-note .info .compact}

  ```bash
  source scripts/env.sh
  gcloud compute health-checks describe "$HC_NAME" \
    --format="yaml(name,type,httpHealthCheck.port,httpHealthCheck.requestPath,checkIntervalSec,timeoutSec)" \
    | tee outputs/health_check.yaml
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[3] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 5. Crear MIG con autoscaling + autohealing (Tiempo estimado: 12 min)

En esta tarea crearás el Managed Instance Group, habilitarás autoscaling por CPU y autohealing con el health check.

#### Tarea 5.1 (UI) — Crear Managed Instance Group

- {% include step_label.html %} Ve a **Compute Engine** luego en  **Instance groups** 

  {% include step_image.html %}

- {% include step_label.html %} Clic en **Create instance group**.

  > **NOTA:** Un MIG mantiene el “número deseado” de instancias y puede recrearlas automáticamente.
  {: .lab-note .info .compact}

  {% include step_image.html %}

- {% include step_label.html %} Selecciona **New managed instance group (stateless)** y configura los siguientes datos:

  > **NOTA:** Single zone simplifica el lab; en producción suelen usarse MIG regionales.
  {: .lab-note .info .compact}

  - Name: `lab06-mig-web`
  - Instance template: `lab06-tpl-web`

  {% include step_image.html %}

  - Location: **Multiple zones**
    - Zones: `us-central1-a`
    - Zones: `us-central1-b`
    - Zones: `us-central1-c`

  {% include step_image.html %}

- {% include step_label.html %} En **Autoscaling**:

  > **NOTA:** Autoscaling por CPU es una forma simple de demostrar escalado sin load balancer.
  {: .lab-note .info .compact}

  - Enable autoscaling: **ON: add and remove instances to the group**
  - Minimum number of instances: `1`
  - Maximum number of instances: `3`

  {% include step_image.html %}

  - Autoscaling signals/Edit signal
    - Signal type: **CPU utilization**
    - Target CPU utilization: `60` (60%)
    - Clic en **Done**

  {% include step_image.html %}

- {% include step_label.html %} En **Autohealing**:

  > **NOTA:** “Initial delay” evita que se marque como unhealthy mientras termina el startup script.
  {: .lab-note .info .compact}

  - Health check: **lab06-hc-http (HTTP)**
  - Initial delay: `60` seconds

  {% include step_image.html %}

- {% include step_label.html %} Clic en **Create**.

#### Tarea 5.2 (Cloud Shell) — Validar MIG e instancias

- {% include step_label.html %} Verifica configuración del MIG.

  > **NOTA:** Confirma que autoscaling y autohealing están activos y apuntan a tu health check.
  {: .lab-note .info .compact}

  ```bash
  source scripts/env.sh
  gcloud compute instance-groups managed list  \
    --format="yaml(name,instanceTemplate,targetSize,autoscaler,autoHealingPolicies,namedPorts)" \
    | tee outputs/mig.yaml
  ```
  {% include step_image.html %} 

- {% include step_label.html %} Lista instancias del MIG y guarda evidencia.

  > **NOTA:** `healthState` te permite ver si el health check ya reporta HEALTHY.
  {: .lab-note .info .compact}

  ```bash
  source scripts/env.sh
  REGION="${REGION:-${ZONE%-*}}"

  gcloud compute instance-groups managed list-instances "$MIG_NAME" \
    --region "$REGION" \
    --format="table(instance.basename(),instanceStatus,zone.basename())" \
    | tee outputs/mig_instances.txt
  ```
  {% include step_image.html %} 

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[4] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 6. Probar health checks, autohealing y autoscaling (Tiempo estimado: 6 min)

En esta tarea accederás a una VM del MIG por IAP, validarás la página web, forzarás un fallo de salud para observar reemplazo y generarás CPU alta para observar escalado.

#### Tarea 6.1 (Cloud Shell) — Seleccionar una instancia del MIG

- {% include step_label.html %} Captura el nombre de la primera instancia del MIG.

  > **NOTA:** Las instancias de un MIG tienen nombres derivados; por eso las listamos.
  {: .lab-note .info .compact}

  ```bash
  source scripts/env.sh
  REGION="${REGION:-${ZONE%-*}}"

  INSTANCE_1="$(gcloud compute instance-groups managed list-instances "$MIG_NAME" \
    --region "$REGION" \
    --format="value(instance.basename())" \
    | head -n 1)"

  echo "INSTANCE_1=$INSTANCE_1" | tee outputs/instance_1.txt
  ```
  {% include step_image.html %} 

#### Tarea 6.2 (Cloud Shell) — Validar la página web por IAP TCP tunnel

- {% include step_label.html %} Abre un túnel IAP al puerto 80 (mantén esta sesión abierta).

  > **NOTA:** Accedes al puerto 80 sin IP pública ni reglas abiertas a internet.
  {: .lab-note .info .compact}

  ```bash
  source scripts/env.sh
  gcloud compute start-iap-tunnel "$INSTANCE_1" 80 --local-host-port=localhost:8080 --zone "$ZONE"
  ```
  {% include step_image.html %} 

- {% include step_label.html %} Abre el **Web preview** y **Preview on port 8080**

  {% include step_image.html %} 

- {% include step_label.html %} Observa el resultado exitoso del sitio web.

  {% include step_image.html %}

- {% include step_label.html %} En otra terminal de Cloud Shell, valida la página.

  > **NOTA:** Si `/healthz` devuelve 200, el health check debería marcar HEALTHY.
  {: .lab-note .info .compact}

  ```bash
  curl -sS http://localhost:8080/ | grep -E "Lab06|Managed Instance Group|Nginx OK" | head
  ```
  ```bash
  curl -sS -o /dev/null -w "HTTP %{http_code}\n" http://localhost:8080/healthz | tee outputs/healthz_http_code.txt
  ```
  {% include step_image.html %} 

#### Tarea 6.3 (Cloud Shell) — Forzar fallo de health check y observar autohealing

- {% include step_label.html %} **En la primera terminal** ejecuta `CTRL + C` y detén Nginx en la instancia (provoca que `/healthz` falle).

  > **NOTA:** Un health check HTTP fallará si el servidor no responde o devuelve error.
  {: .lab-note .info .compact}

  ```bash
  source scripts/env.sh
  gcloud compute ssh "$INSTANCE_1" --zone "$ZONE" --tunnel-through-iap --command "sudo systemctl stop nginx && systemctl status nginx --no-pager -l | sed -n '1,25p'"
  ```
  {% include step_image.html %} 

- {% include step_label.html %} Observa el estado del grupo (ejecuta varias veces o usa `watch`).

  > **NOTA:** Cuando una instancia se marca UNHEALTHY, autohealing normalmente la recrea (verás cambiar el nombre/estado). **Es normal que tarde un par de minutos**
  {: .lab-note .info .compact}

  > **IMPORTANTE:** Sino se muestra el cambio la autosanación volvio a levantar solo el servicio de nginx.
  {: .lab-note .important .compact}

  ```bash
  source scripts/env.sh
  watch -n 10 "gcloud compute instance-groups managed list-instances $MIG_NAME --zone $ZONE --format='table(instance,status,healthState)'"
  ```
  {% include step_image.html %} 

- {% include step_label.html %} Guarda evidencia final del listado (cuando veas reemplazo o cambio de estado).

  > **NOTA:** Esta evidencia muestra el comportamiento de autohealing.
  {: .lab-note .info .compact}

  ```bash
  source scripts/env.sh
  REGION="${REGION:-${ZONE%-*}}"

  gcloud compute instance-groups managed list-instances "$MIG_NAME" \
    --region "$REGION" \
    --format="table(instance.basename(),instanceStatus,zone.basename())" \
    | tee outputs/mig_instances_after_autohealing.txt
  ```
  {% include step_image.html %} 

#### Tarea 6.4 (Cloud Shell) — Inducir CPU alta para disparar autoscaling

- {% include step_label.html %} Selecciona una instancia HEALTHY (si se reemplazó, vuelve a tomar la primera).

  > **NOTA:** Después de autohealing, puede cambiar el nombre de la instancia.
  {: .lab-note .info .compact}

  ```bash
  source scripts/env.sh
  REGION="${REGION:-${ZONE%-*}}"

  INSTANCE_CPU="$(gcloud compute instance-groups managed list-instances "$MIG_NAME" \
    --region "$REGION" \
    --format="value(instance.basename())" \
    | head -n 1)"

  echo "INSTANCE_CPU=$INSTANCE_CPU" | tee outputs/instance_cpu.txt
  ```
  {% include step_image.html %} 

- {% include step_label.html %} Instala `stress-ng` y ejecuta carga por 180s para elevar CPU.

  > **NOTA:** Con 1 instancia, un CPU al 100% suele superar el objetivo (60%) y el autoscaler puede incrementar `targetSize`.
  {: .lab-note .info .compact}

  ```bash
  source scripts/env.sh
  REGION="${REGION:-${ZONE%-*}}"

  # 1) Nombre de la 1ª VM del MIG (esto sí te está funcionando)
  INSTANCE_CPU="$(gcloud compute instance-groups managed list-instances "$MIG_NAME" \
    --region "$REGION" \
    --format="value(name)" \
    | head -n 1)"

  # 2) Zona REAL de esa VM (directo desde Compute Instances)
  ZONE_CPU="$(gcloud compute instances list \
    --filter="name=($INSTANCE_CPU)" \
    --format="value(zone.basename())" \
    | head -n 1)"

  echo "INSTANCE_CPU=$INSTANCE_CPU" | tee outputs/instance_cpu.txt
  echo "ZONE_CPU=$ZONE_CPU" | tee outputs/zone_cpu.txt

  # 3) SSH por IAP y carga CPU
  gcloud compute ssh "$INSTANCE_CPU" \
    --zone "$ZONE_CPU" \
    --tunnel-through-iap \
    --command "sudo apt-get update -y && sudo apt-get install -y stress-ng && stress-ng --cpu 1 --timeout 180s"
  ```
  {% include step_image.html %} 

- {% include step_label.html %} **En la segunda pestaña** ejecuta el siguiente comando y monitorea el tamaño objetivo del MIG y guarda evidencia.

  > **NOTA:** Si `targetSize` aumenta (por ejemplo de 1 a 2), confirmas autoscaling.
  {: .lab-note .info .compact}

  ```bash
  source scripts/env.sh
  REGION="${REGION:-${ZONE%-*}}"

  watch -n 10 "gcloud compute instance-groups managed describe '$MIG_NAME' --region '$REGION' --format='value(targetSize)'"
  ```
  {% include step_image.html %} 

- {% include step_label.html %} Regres a la **Primera terminal** y captura evidencia del tamaño y lista instancias finales.

  > **NOTA:** Evidencia final de escalado (si el grupo creció).
  {: .lab-note .info .compact}

  ```bash
  source scripts/env.sh
  REGION="${REGION:-${ZONE%-*}}"

  gcloud compute instance-groups managed describe "$MIG_NAME" \
    --region "$REGION" \
    --format="yaml(name,targetSize)" \
    | tee outputs/mig_targetsize_final.yaml
  ```
  {% include step_image.html %} 
  ```bash
  source scripts/env.sh
  REGION="${REGION:-${ZONE%-*}}"

  gcloud compute instance-groups managed list-instances "$MIG_NAME" \
    --region "$REGION" \
    --format="table(name,zone,status,healthState)" \
    | tee outputs/mig_instances_final.txt
  ```
  {% include step_image.html %} 

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[5] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 7. Limpieza (Tiempo estimado: 2 min)

Elimina recursos para evitar costos.

#### Tarea 7.1

- {% include step_label.html %} Elimina el MIG (UI o CLI). CLI recomendado para rapidez:

  > **NOTA:** El MIG elimina instancias administradas; así evitas VMs “olvidadas”.
  {: .lab-note .info .compact}

  ```bash
  source scripts/env.sh
  REGION="${REGION:-${ZONE%-*}}"

  gcloud compute instance-groups managed delete "$MIG_NAME" \
    --region "$REGION" \
    --quiet
  ```

- {% include step_label.html %} Elimina el Instance Template.

  > **NOTA:** Plantillas también deben limpiarse para dejar el proyecto ordenado.
  {: .lab-note .info .compact}

  ```bash
  source scripts/env.sh
  REGION="${REGION:-${ZONE%-*}}"

  gcloud compute instance-templates delete "$TEMPLATE_NAME" \
    --region "$REGION" \
    --quiet
  ```

- {% include step_label.html %} Elimina el Health Check.

  ```bash
  source scripts/env.sh
  gcloud compute health-checks delete "$HC_NAME" --quiet
  ```

- {% include step_label.html %} Elimina reglas de firewall del lab.

  ```bash
  gcloud compute firewall-rules delete \
    lab06-fw-allow-iap-ssh \
    lab06-fw-allow-iap-http \
    lab06-fw-allow-hc-http \
    --quiet
  ```

- {% include step_label.html %} **Cloud Shell** Elimina Cloud NAT:

  ```bash
  source scripts/env.sh
  REGION="${REGION:-us-central1}"
  NETWORK="${NETWORK:-lab06-vpc}"
  ROUTER_NAME="${ROUTER_NAME:-lab06-router}"
  NAT_NAME="${NAT_NAME:-lab06-nat}"

  gcloud compute routers nats delete "$NAT_NAME" \
    --router "$ROUTER_NAME" \
    --region "$REGION" \
    --quiet
  ```
  ```bash
  gcloud compute routers delete "$ROUTER_NAME" \
  --region "$REGION" \
  --quiet
  ```

- {% include step_label.html %} Elimina subnet y VPC.

  ```bash
  source scripts/env.sh
  gcloud compute networks subnets delete "$SUBNET_NAME" --region "$REGION" --quiet
  gcloud compute networks delete "$VPC_NAME" --quiet
  ```

- {% include step_label.html %} Verifica que ya no existan .

  ```bash
  source scripts/env.sh; REGION="${REGION:-${ZONE%-*}}"; \
  echo "=== PROJECT ==="; gcloud config get-value project; \
  echo; echo "=== MIG (regional) ==="; \
  gcloud compute instance-groups managed describe "$MIG_NAME" --region "$REGION" >/dev/null 2>&1 && echo "MIG EXISTS: $MIG_NAME (region $REGION)" || echo "OK: MIG eliminado"; \
  echo; echo "=== Instance Template (global/regional) ==="; \
  gcloud compute instance-templates describe "$TEMPLATE_NAME" >/dev/null 2>&1 && echo "TEMPLATE EXISTS (global): $TEMPLATE_NAME" || echo "OK: template global no existe"; \
  gcloud compute instance-templates describe "$TEMPLATE_NAME" --region "$REGION" >/dev/null 2>&1 && echo "TEMPLATE EXISTS (regional): $TEMPLATE_NAME (region $REGION)" || echo "OK: template regional no existe"; \
  echo; echo "=== Health Check (global/regional) ==="; \
  gcloud compute health-checks describe "$HC_NAME" --global >/dev/null 2>&1 && echo "HC EXISTS (global): $HC_NAME" || echo "OK: HC global no existe"; \
  gcloud compute health-checks describe "$HC_NAME" --region "$REGION" >/dev/null 2>&1 && echo "HC EXISTS (regional): $HC_NAME (region $REGION)" || echo "OK: HC regional no existe"; \
  echo; echo "=== Firewall rules (lab06-*) ==="; \
  for r in lab06-fw-allow-iap-ssh lab06-fw-allow-iap-http lab06-fw-allow-hc-http; do \
    gcloud compute firewall-rules describe "$r" >/dev/null 2>&1 && echo "FW EXISTS: $r" || echo "OK: FW no existe: $r"; \
  done; \
  echo; echo "=== Cloud NAT / Router ==="; \
  gcloud compute routers describe "${ROUTER_NAME:-lab06-router}" --region "$REGION" >/dev/null 2>&1 && echo "ROUTER EXISTS: ${ROUTER_NAME:-lab06-router} (region $REGION)" || echo "OK: router no existe"; \
  gcloud compute routers nats describe "${NAT_NAME:-lab06-nat}" --router "${ROUTER_NAME:-lab06-router}" --region "$REGION" >/dev/null 2>&1 && echo "NAT EXISTS: ${NAT_NAME:-lab06-nat} (router ${ROUTER_NAME:-lab06-router})" || echo "OK: NAT no existe"; \
  echo; echo "=== Subnet / VPC ==="; \
  gcloud compute networks subnets describe "$SUBNET_NAME" --region "$REGION" >/dev/null 2>&1 && echo "SUBNET EXISTS: $SUBNET_NAME (region $REGION)" || echo "OK: subnet no existe"; \
  gcloud compute networks describe "$VPC_NAME" >/dev/null 2>&1 && echo "VPC EXISTS: $VPC_NAME" || echo "OK: VPC no existe"
  ```
  {% include step_image.html %} 

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[6] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}