---
layout: lab
title: "Práctica 7: Configurar balanceador global con certificado SSL administrado"
permalink: /lab7/lab7/
images_base: /labs/lab7/img
duration: "45 minutos"
objective:
  - "Configurar un **HTTP(S) Load Balancer global externo** en Google Cloud con un **certificado SSL administrado por Google (Google-managed)**, sin pagar herramientas de terceros, publicando una aplicación web detrás de un **Managed Instance Group (MIG)** y validando salud, HTTPS y el estado del certificado con evidencia por UI y comandos."
prerequisites:
  - "Proyecto de Google Cloud con permisos para **Compute Engine**, **VPC**, **Firewall**, **Instance groups**, **Health checks**, **Load Balancing**, **Certificate Manager** e **IAM**."
  - "Acceso a **Google Cloud Console** y **Cloud Shell**."
  - "Navegador con sesión activa en Google."
introduction: |
  Un **balanceador HTTP(S) global externo** (Global external HTTPS) publica tu aplicación en internet usando la infraestructura global de Google (GFE), y enruta tráfico a tus backends (por ejemplo, un MIG) basado en **Backend Service + Health Checks**.

  Para HTTPS, la opción más efectiva y práctica es un **certificado SSL administrado por Google** (Google-managed):
  - No compras certificados ni pagas herramientas externas.
  - Google gestiona emisión, instalación y renovación automática.
  - Requiere un **nombre de dominio** que resuelva a la IP del balanceador para validar control del dominio.

  En este laboratorio usarás **dos opciones de dominio**, sin “herramientas pagadas”:
  - **Opción A (Recomendada para entornos reales):** un dominio propio (puede tener costo del dominio, pero sin herramientas externas).
  - **Opción B (Para laboratorio sin comprar dominio):** usar un hostname dinámico tipo `sslip.io` basado en la IP del LB (no requiere comprar dominio).  
    Ejemplo: si tu IP es `34.120.10.20`, el dominio `34-120-10-20.sslip.io` resuelve a esa IP.
slug: lab7
lab_number: 7
final_result: >
  Al finalizar, tendrás un Load Balancer global externo HTTPS con un certificado administrado (Google-managed) en estado ACTIVE, un backend en un MIG con health checks saludables, y acceso funcional por HTTPS con verificación de certificado (curl/openssl) y evidencias en `outputs/`.
notes:
  - "**Costos:** El Global external HTTP(S) Load Balancer tiene costo (reglas de reenvío, procesamiento, etc.). Compute Engine cobra por VMs. El certificado Google-managed **no tiene costo adicional**. Elimina recursos al final."
  - "La emisión del certificado puede tardar algunos minutos; valida el estado en UI/CLI hasta que esté **ACTIVE**."
  - "Para que el certificado se emita, el dominio debe resolver a la IP del LB y el frontend HTTPS debe estar creado."
references:
  - text: "HTTP(S) Load Balancing: visión general"
    url: https://cloud.google.com/load-balancing/docs/https
  - text: "Certificados SSL administrados (Google-managed) con Certificate Manager"
    url: https://docs.cloud.google.com/load-balancing/docs/ssl-certificates/google-managed-certs
  - text: "Configurar un balanceador HTTP(S) con backends en instance group"
    url: https://cloud.google.com/load-balancing/docs/https/setting-up-https
  - text: "Health checks y rangos de IP para LB/health check"
    url: https://cloud.google.com/load-balancing/docs/health-checks
  - text: "Managed Instance Groups"
    url: https://cloud.google.com/compute/docs/instance-groups
  - text: "Compute Engine pricing"
    url: https://cloud.google.com/compute/all-pricing
prev: /lab6/lab6/
next: /lab8/lab8/
---

---

## Instrucciones generales

- Esta práctica es **mayormente por interfaz gráfica (UI)**:
  - VPC/subnet/firewall, instance template, MIG, health check, load balancer y certificado.
- Usaremos **Cloud Shell** para:
  - crear estructura de carpetas y variables,
  - generar un startup script reproducible,
  - validar recursos con `gcloud`,
  - verificar HTTPS y certificado con `curl`/`openssl`.

Carpeta de trabajo (Cloud Shell):
- `~/labs-gcp-engineer/lab07/` con `scripts/` y `outputs/`.

---

## Opciones de certificado (sin pagar herramientas de terceros)

- **Opción 1 (Recomendada): Google-managed certificate**
  - Se crea en **Certificate Manager**.
  - Google emite y renueva automáticamente.
  - Requiere dominio resolviendo a la IP del LB (DNS A/AAAA).
- **Opción 2 (Alternativa): Self-managed certificate (OpenSSL)**
  - Gratis si lo generas tú mismo, pero **tú** administras renovación/rotación.
  - Útil solo para demos/labs. No es ideal para producción.
- **Opción 3 (Evitar): Certbot/Let’s Encrypt en VMs**
  - No se integra “limpio” con un Global HTTPS LB (el cert vive en el LB, no en el backend).
  - Aumenta complejidad operativa.

En este laboratorio implementaremos **Opción 1**.

---

### Tarea 1. Preparar Cloud Shell y variables del laboratorio (Tiempo estimado: 5 min)

En esta tarea crearás la estructura del lab y definirás variables alineadas a **Práctica 7**.

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

- {% include step_label.html %} Crea la carpeta del laboratorio 7.

  ```bash
  cd ~
  mkdir -p labs-gcp-engineer/lab07/{scripts,outputs}
  ```
  ```bash
  cd labs-gcp-engineer/lab07
  ```
  {% include step_image.html %}

- {% include step_label.html %} Crea `scripts/env.sh` (nombres alineados a lab07).

  ```bash
  cat > scripts/env.sh <<'EOF'
  # Región / zona (backends)
  export REGION="us-central1"
  export ZONE="us-central1-a"

  # Red del lab
  export VPC_NAME="lab07-vpc"
  export SUBNET_NAME="lab07-subnet-app"
  export SUBNET_CIDR="10.70.10.0/24"

  # Tags firewall
  export TAG_WEB="lab07-web"

  # Compute backend
  export TEMPLATE_NAME="lab07-tpl-web"
  export MIG_NAME="lab07-mig-web"

  # Health check HTTP (backend)
  export HC_NAME="lab07-hc-http"
  export HC_PORT="80"
  export HC_PATH="/healthz"

  # Load balancer / certificado
  export LB_NAME="lab07-lb-https"
  export LB_IP_NAME="lab07-lb-ip"
  export CERT_NAME="lab07-cert-managed"
  export URLMAP_NAME="lab07-urlmap"
  export PROXY_NAME="lab07-https-proxy"
  export FWD_RULE_NAME="lab07-fwdrule-https"
  export BACKEND_SVC_NAME="lab07-backend-svc"
  EOF
  ```
  ```bash
  source scripts/env.sh
  ```

- {% include step_label.html %} Configura región y zona por defecto y guarda evidencia.

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

### Tarea 2. Crear backend (VPC + MIG + health check) (Tiempo estimado: 15 min)

En esta tarea crearás una VPC del lab, una regla firewall para permitir tráfico del Load Balancer/health checks, un Instance Template con Nginx + endpoint `/healthz`, un Health Check HTTP y un MIG.

> **Nota:** El LB global usará tu MIG como backend. Las instancias no requieren IP pública.
{: .lab-note .info .compact}

#### Tarea 2.1 (UI) — Crear VPC y subnet

- {% include step_label.html %} En la consola: **VPC network** y luego **VPC networks**

  {% include step_image.html %}

- {% include step_label.html %} Da clic en la opción **Create VPC network**.

  > **NOTA:** Mantener recursos del lab aislados facilita limpieza y evita colisiones.
  {: .lab-note .info .compact}

  {% include step_image.html %}

- {% include step_label.html %} Configura los siguientes datos:

  - Name: `lab07-vpc`
  - Subnet creation mode: **Custom**

  {% include step_image.html %}

- {% include step_label.html %} Agrega una subnet:

  - Subnet name: `lab07-subnet-app`
  - Region: **us-central1 (Iowa)**
  - IPv4 range: `10.70.10.0/24`
  - Private Google Access: **ON**
  - Clic en **Done**

  {% include step_image.html %}

- {% include step_label.html %} Clic en **Create**.

- {% include step_label.html %} Verifica mediante CLI la creacion de la Virtual Network.

  > **NOTA:** Confirma CIDR y red correcta.
  {: .lab-note .info .compact}

  ```bash
  source scripts/env.sh
  gcloud compute networks describe "$VPC_NAME" --format="yaml(name,autoCreateSubnetworks)" | tee outputs/vpc.yaml
  ```
  ```bash
  gcloud compute networks subnets describe "$SUBNET_NAME" --region "$REGION" --format="yaml(name,ipCidrRange,network)" | tee outputs/subnet.yaml
  ```
  {% include step_image.html %}

#### Tarea 2.2 (UI) — Firewall para LB/Health Checks hacia backends

- {% include step_label.html %} En consola da clic en: **VPC network** y luego **Firewall**. Si ya estas en el menú busca solo **Firewall**

  {% include step_image.html %}

- {% include step_label.html %} Ahora da clic en **Create firewall rule**.

  {% include step_image.html %}

- {% include step_label.html %} Configura los siguientes datos:

  > **NOTA:** En vez de exponer SSH públicamente, lo consumes por IAP.
  {: .lab-note .info .compact}

  - Name: `lab07-fw-allow-iap-ssh`
  - Network: `lab07-vpc`
  - Direction of traffic: **Ingress**
  - Targets: **Specified target tags**
  - Target tags: `lab07-iap`
  - Source IPv4 ranges: `35.235.240.0/20`
  - Protocols/ports: `tcp:22`

- {% include step_label.html %} Clic en el botón **Create** al final de la pagina.

- {% include step_label.html %} Ahora crea otra regla y configura los siguientes datos:

  > **NOTA:** Estos rangos incluyen health checkers y proxies del HTTP(S) LB; sin esto, tu backend queda UNHEALTHY.
  {: .lab-note .important .compact}

  **Configura:**
  - Name: `lab07-fw-allow-lb-http`
  - Network: **lab07-vpc**
  - Direction of traffic: **Ingress**

  {% include step_image.html %}

  - Targets: **Specified target tags**
  - Target tags: `lab07-web`
  - Source IPv4 ranges: `130.211.0.0/22,35.191.0.0/16`
  - Protocols/ports: `tcp:80`

  {% include step_image.html %}

- {% include step_label.html %} Clic en el botón **Create** al final de la pagina.

- {% include step_label.html %} Verifica la configuración mediante CLI.

  ```bash
  gcloud compute firewall-rules describe lab07-fw-allow-lb-http \
    --format="yaml(name,network,sourceRanges,allowed,targetTags)" | tee outputs/fw_allow_lb_http.yaml
  ```
  {% include step_image.html %}

#### Tarea 2.3 (Cloud Shell) — Crear startup script reproducible

- {% include step_label.html %} Crea el siguiente archivo `scripts/startup.sh`.

  > **NOTA:** El endpoint `/healthz` es el objetivo del health check.
  {: .lab-note .info .compact}

  ```bash
  cd ~/labs-gcp-engineer/lab07
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
      <title>Lab07 - Global HTTPS LB</title>
      <style>
        body { font-family: system-ui, -apple-system, Segoe UI, Roboto, Arial; background: #0b1220; color: #e6eefc; padding: 28px; }
        .wrap { max-width: 980px; margin: 0 auto; }
        .card { background: rgba(255,255,255,0.06); border: 1px solid rgba(255,255,255,0.12); border-radius: 16px; padding: 18px; box-shadow: 0 10px 28px rgba(0,0,0,0.35); }
        h1 { margin: 0 0 10px; font-size: 26px; }
        .pill { display:inline-block; padding: 6px 10px; border-radius: 999px; background: rgba(34,197,94,0.18); border: 1px solid rgba(34,197,94,0.35); }
        table { width: 100%; border-collapse: collapse; margin-top: 12px; overflow: hidden; border-radius: 12px; }
        th, td { padding: 10px 12px; border-bottom: 1px solid rgba(255,255,255,0.10); font-size: 14px; }
        th { text-align: left; background: rgba(255,255,255,0.05); }
        code { background: rgba(255,255,255,0.08); padding: 2px 6px; border-radius: 6px; }
      </style>
    </head>
    <body>
      <div class="wrap">
        <div class="card">
          <h1>Lab07: Global HTTPS Load Balancer</h1>
          <p class="pill">Backend Nginx OK</p>
          <table>
            <tr><th>Instance</th><td><code>${INSTANCE_NAME}</code></td></tr>
            <tr><th>Zone</th><td><code>${ZONE}</code></td></tr>
            <tr><th>Health</th><td><code>/healthz</code> returns 200</td></tr>
            <tr><th>Timestamp (UTC)</th><td><code>$(date -u +%FT%TZ)</code></td></tr>
          </table>
          <p style="opacity:.85;margin-top:10px;">Este backend será servido a través de un Global HTTPS LB con certificado administrado.</p>
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
  sed -n '1,90p' scripts/startup.sh | tee outputs/startup_preview.txt
  ```

#### Tarea 2.4 (UI) — Crear Instance Template y MIG

- {% include step_label.html %} En consola: da clic en **Compute Engine** luego **Instance templates**.

  {% include step_image.html %}

- {% include step_label.html %} Da clic en **Create instance template**.

  > **NOTA:** Todas las VMs del MIG serán idénticas por este template.
  {: .lab-note .info .compact}

  {% include step_image.html %}

- {% include step_label.html %} Configura los siguientes datos:

  - Name: `lab07-tpl-web`
  - Location: **Regional**
  - Region: **us-central1 (Iowa)**

  {% include step_image.html %}

  - Machine type: **e2-medium**

  {% include step_image.html %}

  - Boot disk: **Debian 12**
  
  {% include step_image.html %}

  - Advanced options/Networking/Network tags
    - `lab07-web`
    - `lab07-iap` 

    {% include step_image.html %}

  - Advanced options/Networking/Network interfaces
    - Network: **lab07-vpc**
    - Subnetwork: **lab07-subnet-app**
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

  # Log útil para depurar desde serial console / /var/log
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
      <title>Lab07 - Global HTTPS LB</title>
      <style>
        body { font-family: system-ui, -apple-system, Segoe UI, Roboto, Arial; background: #0b1220; color: #e6eefc; padding: 28px; }
        .wrap { max-width: 980px; margin: 0 auto; }
        .card { background: rgba(255,255,255,0.06); border: 1px solid rgba(255,255,255,0.12); border-radius: 16px; padding: 18px; box-shadow: 0 10px 28px rgba(0,0,0,0.35); }
        h1 { margin: 0 0 10px; font-size: 26px; }
        .pill { display:inline-block; padding: 6px 10px; border-radius: 999px; background: rgba(34,197,94,0.18); border: 1px solid rgba(34,197,94,0.35); }
        table { width: 100%; border-collapse: collapse; margin-top: 12px; overflow: hidden; border-radius: 12px; }
        th, td { padding: 10px 12px; border-bottom: 1px solid rgba(255,255,255,0.10); font-size: 14px; }
        th { text-align: left; background: rgba(255,255,255,0.05); }
        code { background: rgba(255,255,255,0.08); padding: 2px 6px; border-radius: 6px; }
      </style>
    </head>
    <body>
      <div class="wrap">
        <div class="card">
          <h1>Lab07: Global HTTPS Load Balancer</h1>
          <p class="pill">Backend Nginx OK</p>
          <table>
            <tr><th>Instance</th><td><code>${INSTANCE_NAME}</code></td></tr>
            <tr><th>Zone</th><td><code>${ZONE}</code></td></tr>
            <tr><th>Health</th><td><code>/healthz</code> returns 200</td></tr>
            <tr><th>Timestamp (UTC)</th><td><code>$(date -u +%FT%TZ)</code></td></tr>
          </table>
          <p style="opacity:.85;margin-top:10px;">Este backend será servido a través de un Global HTTPS LB con certificado administrado.</p>
        </div>
      </div>
    </body>
  </html>
  HTML

  echo "ok" > /var/www/html/healthz

  nginx -t
  systemctl enable --now nginx
  ```
  {% include step_image.html %}

- {% include step_label.html %} Clic en **Create**.

- {% include step_label.html %} En consola: **Compute Engine** luego **Health checks**.

  {% include step_image.html %}

- {% include step_label.html %} Clic en **Create health check**.

  {% include step_image.html %}

- {% include step_label.html %} Configura los siguientes datos:

  > **NOTA:** Este health check será usado por el backend service.
  {: .lab-note .info .compact}

  - Name: `lab07-hc-http`
  - Scope: **Global**
  - Protocol: **HTTP**
  - Port: `80`
  - Request path: `/healthz`

  {% include step_image.html %}

- {% include step_label.html %} Clic en **Create**.

- {% include step_label.html %} Ve a **Compute Engine** luego en  **Instance groups** 

  {% include step_image.html %}

- {% include step_label.html %} Clic en **Create instance group**.

  > **NOTA:** Con 1 instancia basta para validar el LB; puedes escalar después.
  {: .lab-note .info .compact}

  {% include step_image.html %}

- {% include step_label.html %} Primero **Dentro de Cloud Shell** crea un Cloud NAT temporal para la descarga de dependencias de la practica, ejecuta todo el codigo sisguiente

  ```bash
  REGION="us-central1"
  NETWORK="lab07-vpc"
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

- {% include step_label.html %} Selecciona **New managed instance group (stateless)** y configura los siguientes datos:

  - Name: `lab07-mig-web`
  - Instance template: **lab07-tpl-web**

  {% include step_image.html %}

  - Location: Single zone
  - Region: **us-central1 (Iowa)
  - Zone: **us-central1-a**

  {% include step_image.html %}

  - Auto scaling mode: **On: add and remove instances to the group**
  - Minimum number of instances: `1`
  - Maximum number of instances: `2`

  {% include step_image.html %}

- {% include step_label.html %} Clic en **Create**.

- {% include step_label.html %} Ejecuta los siguientes coamdnso para la videncia por CLI (template, health check y MIG).

  > **NOTA:** Confirmas recursos base antes de armar el LB.
  {: .lab-note .info .compact}

  ```bash
  source scripts/env.sh
  REGION="${REGION:-${ZONE%-*}}"

  gcloud compute instance-templates describe "$TEMPLATE_NAME" \
    --region "$REGION" \
    --format="yaml(name,properties.tags,properties.networkInterfaces,properties.metadata.items)" \
    | tee outputs/template.yaml
  ```
  ```bash
  gcloud compute health-checks describe "$HC_NAME" \
    --format="yaml(name,type,httpHealthCheck.requestPath,httpHealthCheck.port)" \
    | tee outputs/health_check.yaml
  ```
  {% include step_image.html %}
  ```bash
  gcloud compute instance-groups managed describe "$MIG_NAME" --zone "$ZONE" \
    --format="yaml(name,instanceTemplate,targetSize,namedPorts)" \
    | tee outputs/mig.yaml
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[1] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 3. Reservar IP global y definir dominio (Tiempo estimado: 6 min)

En esta tarea crearás una **IP global estática** para el balanceador y definirás el dominio que usarás para el certificado administrado.

#### Tarea 3.1 (UI) — Reservar IP global estática

- {% include step_label.html %} En consola: **VPC network** luego **IP addresses**.

  {% include step_image.html %}

- {% include step_label.html %} Ahora da clic en **Reserve external**.

  > **NOTA:** El certificado y el DNS deben apuntar a una IP estable.
  {: .lab-note .info .compact}

  {% include step_image.html %}

- {% include step_label.html %} Configura los siguientes datos:

  - Name: `lab07-lb-ip`
  - Network Service Tier: **Premium**
  - IP version: **IPv4**
  - Type: **Global**

  {% include step_image.html %}

- {% include step_label.html %} clic en el botón **Reserve**

- {% include step_label.html %} Obtén la IP por CLI y guárdala en un bloc de notas.

  > **NOTA:** Esta IP se usará para el frontend HTTPS y para el dominio.
  {: .lab-note .info .compact}

  ```bash
  source scripts/env.sh
  export LB_IP="$(gcloud compute addresses describe "$LB_IP_NAME" --global --format='value(address)')"
  echo "LB_IP=$LB_IP" | tee outputs/lb_ip.txt
  ```
  {% include step_image.html %}

#### Tarea 3.2 (Cloud Shell) — Definir el dominio

- {% include step_label.html %} **Opción (laboratorio sin comprar dominio):** usa `sslip.io` con la IP del LB.

  > **NOTA:** `sslip.io` resuelve automáticamente el hostname a la IP embebida; es ideal para laboratorios sin comprar dominio.
  {: .lab-note .info .compact}

  ```bash
  # Convierte "34.120.10.20" -> "34-120-10-20"
  DASHED_IP="${LB_IP//./-}"
  export DOMAIN="${DASHED_IP}.sslip.io"
  echo "DOMAIN=$DOMAIN" | tee outputs/domain.txt

  # Verifica resolución DNS (debe regresar la IP del LB)
  dig +short "$DOMAIN" | tee outputs/dns_lookup.txt
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[2] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 4. Crear certificado SSL administrado (Google-managed) (Tiempo estimado: 5 min)

En esta tarea crearás un certificado administrado por Google con el dominio definido.

#### Tarea 4.1 (UI) — Crear certificado (Certificate Manager)

- {% include step_label.html %} En consola: **Security** luego **Certificate Manager**.

  {% include step_image.html %}

- {% include step_label.html %} Da clic en **Enable** para habilitar la API

  {% include step_image.html %}

- {% include step_label.html %} Ahora da clic en **Create SSL certificate** de la sección **Classic Certificates**

  > **NOTA:** Certificate Manager administra certificados y renovaciones automáticamente.
  {: .lab-note .info .compact}

  {% include step_image.html %}

- {% include step_label.html %} Configura los siguientes datos:

  > **NOTA:** El certificado se emitirá cuando el dominio apunte al frontend HTTPS del LB.
  {: .lab-note .info .compact}

  - Certificate Name: `lab07-cert-managed`
  - Location: **Create Google-managed certificate**
  - Domains: pega el valor de tu variable DOMAIN **Ej: 34-120-10-20.sslip.io**

  {% include step_image.html %}

- {% include step_label.html %} Da clic en el botón **Create**

- {% include step_label.html %} Evidencia por CLI puede tardar un par de minutos el estado.

  > **NOTA:** Al inicio puede estar en PROVISIONING / PENDING. Recurso regional/global depende del tipo y consola; listamos para evidencia.
  {: .lab-note .info .compact}

  ```bash
  source scripts/env.sh
  CERT_CLASSIC_NAME="${CERT_CLASSIC_NAME:-lab07-cert-managed}"

  # 1) Listar certificados clásicos (globales)
  gcloud compute ssl-certificates list \
    --format="table(name,type,managed.status,managed.domains,creationTimestamp)" \
    | tee outputs/classic_cert_list.txt
  ```
  {% include step_image.html %}
  ```bash
  # 2) Describir el certificado clásico (estado/DOMINIOS)
  gcloud compute ssl-certificates describe "$CERT_CLASSIC_NAME" \
    --global \
    --format="yaml(name,type,managed.status,managed.domains,creationTimestamp,expireTime)" \
    | tee outputs/classic_cert.yaml
  ```
  {% include step_image.html %}

- {% include step_label.html %} Aun con el estatus **PROVISIONING** avanza a la siguiente **Tarea 5**.

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[3] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 5. Crear Global HTTPS Load Balancer y asociar el certificado (Tiempo estimado: 10 min)

En esta tarea crearás el balanceador global HTTPS, conectarás el backend (MIG) y usarás el certificado administrado en el frontend 443.

#### Tarea 5.1 (UI) — Crear Load Balancer HTTPS global externo

- {% include step_label.html %} En consola: en el buscador escribe **Load balancing** y da clic en el servicio.

  {% include step_image.html %}

- {% include step_label.html %} Clic en **Create load balancer**.

  {% include step_image.html %}

- {% include step_label.html %} Selecciona **Application Load Balancer (HTTP/HTTPS)**:

  > **NOTA:** Esto crea un frontend global Anycast con infraestructura de Google.
  {: .lab-note .info .compact}

  {% include step_image.html %}

- {% include step_label.html %} Clic **Next**.

- {% include step_label.html %} En **Public facing or internal** selecciona.

  - Public facing (external)

  {% include step_image.html %}

- {% include step_label.html %} Clic **Next**.

- {% include step_label.html %} En **Global or single region deployment** selecciona.

  - Best for global workloads

  {% include step_image.html %}

- {% include step_label.html %} Clic **Next**.

- {% include step_label.html %} En **Load balancer generation** selecciona.

  - Global external Application Load Balancer

  {% include step_image.html %}

- {% include step_label.html %} Clic **Next**.

- {% include step_label.html %} Clic en **Configure**.

- {% include step_label.html %} Ahora configura los siguientes datos:

  - Load Balancer name: `lab07-lb-https`

  {% include step_image.html %}

- {% include step_label.html %} En la sección de **Frontend configuration** :

  - Name: `fe-https`
  - Protocol: **HTTPS**
  - IP version: **IPv4**
  - IP address: **lab07-lb-ip**
  - Port: `443`
  - Certificate: **lab07-cert-managed**

  {% include step_image.html %}

- {% include step_label.html %} Clic en el botón **Done**

- {% include step_label.html %} En la sección **Backend configuration**:

- {% include step_label.html %} Clic en **Backend services & backend buckets** luego en **Create a backend service**.

  > **NOTA:** El backend service usa health check para decidir si puede enrutar tráfico.
  {: .lab-note .info .compact}

  {% include step_image.html %}

- {% include step_label.html %} Muy bien ahora en la ventana emergente laterar derecha configura los siguientes datos:

  - Instance group: `lab07-mig-web`
  - Backend type: **Instance group**
  - Protocol: **HTTP**
  - Named port `http`
  - Health check: **lab07-hc-http**

  {% include step_image.html %}

- {% include step_label.html %} Ahora en esa misma ventana pero en la sección de **Backends** configura lo siguiente:

  - Instance group: **lab07-mig-web**
  - Port numbers: `80`
  - Clic en **Done**

  {% include step_image.html %}

- {% include step_label.html %} Desactiva el check de **Enable Cloud CDN** y en **Cloud Armor** seleccciona **None**

  {% include step_image.html %}

- {% include step_label.html %} Clic en **Create**.

- {% include step_label.html %} Ahora en **Routing rules** verifica que este seleccionado el **Backend**.

  {% include step_image.html %}

- {% include step_label.html %} Clic en **Create**

#### Tarea 5.2 (Cloud Shell) — Evidencia de recursos del LB

- {% include step_label.html %} Lista forwarding rule y proxy/cert asociado.

  > **NOTA:** Evidencia del frontend público y el proxy de HTTPS.
  {: .lab-note .info .compact}

  ```bash
  source scripts/env.sh
  gcloud compute forwarding-rules list --global \
    --format="table(name,IPAddress,IPProtocol,portRange,target)" \
    | tee outputs/fwd_rules.txt
  ```
  ```bash
  # Target HTTPS proxy (global)
  gcloud compute target-https-proxies list --filter="name~'lab07'" --format="table(name,sslCertificates,certificateMap)" \
    | tee outputs/https_proxies.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Guarda la IP del LB (si cambió o para confirmar).

  ```bash
  export LB_IP="$(gcloud compute addresses describe "$LB_IP_NAME" --global --format='value(address)')"
  echo "LB_IP=$LB_IP" | tee outputs/lb_ip_confirm.txt
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[4] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 6. Verificar HTTPS, estado del certificado y salud del backend (Tiempo estimado: 4 min)

En esta tarea validarás:
- que el dominio resuelve a la IP,
- que el backend está HEALTHY,
- que HTTPS responde y el certificado es correcto.

#### Tarea 6.1 (Cloud Shell) — DNS, backend health y HTTPS

- {% include step_label.html %} Verifica que el dominio resuelve a la IP del LB.

  > **NOTA:** Si DNS no resuelve a la IP, el certificado no podrá validarse.
  {: .lab-note .important .compact}

  ```bash
  source scripts/env.sh
  export DOMAIN="$(cat outputs/domain.txt | cut -d= -f2)"
  export LB_IP="$(cat outputs/lb_ip.txt | cut -d= -f2)"

  echo "DOMAIN=$DOMAIN"
  echo "LB_IP=$LB_IP"
  dig +short "$DOMAIN"
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica salud del backend service (si ya existe) y del MIG.

  > **NOTA:** Si la instancia no está RUNNING, el backend fallará.
  {: .lab-note .warning .compact}

  ```bash
  source scripts/env.sh
  gcloud compute instance-groups managed list-instances "$MIG_NAME" --zone "$ZONE" \
    --format="table(instance,status)" | tee outputs/mig_instances_check.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Prueba HTTPS (cabeceras) y guarda evidencia.

  > **NOTA:** Debes ver `HTTP/2 200` (o 301 si rediriges) y cabeceras del frontend de Google.
  {: .lab-note .info .compact}

  ```bash
  curl -sS -I "https://${DOMAIN}/" | tee outputs/https_headers.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Inspecciona el certificado presentado (sin descargar nada; solo ver sujeto/issuer/fechas).

  > **NOTA:** Confirma que el cert corresponde al dominio y fue emitido por una CA válida.
  {: .lab-note .info .compact}

  ```bash
  echo | openssl s_client -servername "$DOMAIN" -connect "${DOMAIN}:443" 2>/dev/null \
    | openssl x509 -noout -subject -issuer -dates | tee outputs/cert_inspect.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Revisa el estado del certificado administrado (debe llegar a ACTIVE).

  > **NOTA:** Cuando esté **ACTIVE**, el sitio debería mostrar el candado en el navegador y HTTPS completo.
  {: .lab-note .info .compact}

  ```bash
  source scripts/env.sh
  CERT_CLASSIC_NAME="${CERT_CLASSIC_NAME:-lab07-cert-managed}"

  gcloud compute ssl-certificates describe "$CERT_CLASSIC_NAME" --global \
    --format="value(managed.status)" | tee outputs/classic_cert_status.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Obten el dominio del LB

  ```bash
  echo "DOMAIN=$DOMAIN" | tee outputs/domain_confirm.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Copia y pegalo en una pestaña de tu navegador y deberias ver el sitio con candado HTTPS.

  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[5] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 7. Limpieza (Tiempo estimado: 2–3 min)

Elimina recursos para evitar costos.

#### Tarea 7.1

- {% include step_label.html %} Elimina el Load Balancer por UI.

  > **NOTA:** La UI elimina dependencias (proxy, url map, backend service, forwarding rule) en el orden correcto.
  {: .lab-note .warning .compact}

  {% include step_image.html %}

- {% include step_label.html %} Elimina Cloud NAT en CLI:

  ```bash
  REGION="us-central1"
  NETWORK="lab07-vpc"
  ROUTER_NAME="nat-router"
  NAT_NAME="nat-config"

  gcloud compute routers nats delete "$NAT_NAME" \
    --router "$ROUTER_NAME" \
    --region "$REGION" \
    --quiet
  ```
  ```bash
  gcloud compute routers delete "$ROUTER_NAME" --region "$REGION" --quiet
  ```

- {% include step_label.html %} Elimina IP global estática.

  ```bash
  source scripts/env.sh
  LB_IP_NAME="${LB_IP_NAME:-lab07-lb-ip}"

  gcloud compute addresses delete "$LB_IP_NAME" \
    --global \
    --quiet
  ```

- {% include step_label.html %} Elimina MIG, template, health check, firewall y VPC.

  > **NOTA:** Si te aparece un error al eliminar **lab07-hc-http** puedes ignorarlo al eliminar el LB tambien borraste el HC.
  {: .lab-note .info .compact}

  ```bash
  source scripts/env.sh
  REGION="${REGION:-${ZONE%-*}}"

  # MIG (zonal)
  gcloud compute instance-groups managed delete "$MIG_NAME" \
    --zone "$ZONE" \
    --quiet

  # Instance Template (en tu lab puede ser regional, así que borra regional y global “por si acaso”)
  gcloud compute instance-templates delete "$TEMPLATE_NAME" --quiet 2>/dev/null || true
  gcloud compute instance-templates delete "$TEMPLATE_NAME" --region "$REGION" --quiet 2>/dev/null || true

  # Health check (generalmente global)
  gcloud compute health-checks delete "$HC_NAME" --quiet
  ```
  ```bash
  # Firewall rules del lab07 (borra si existen)
  for FW in lab07-fw-allow-iap-ssh lab07-fw-allow-hc-http lab07-fw-allow-lb-http; do
    gcloud compute firewall-rules delete "$FW" --quiet 2>/dev/null || true
  done

  # Subnet y VPC
  gcloud compute networks subnets delete "$SUBNET_NAME" \
    --region "$REGION" \
    --quiet

  gcloud compute networks delete "$VPC_NAME" \
    --quiet
  ```

- {% include step_label.html %} Verificación final (evidencia).

  ```bash
  source scripts/env.sh
  REGION="${REGION:-${ZONE%-*}}"
  CERT_CLASSIC_NAME="${CERT_CLASSIC_NAME:-lab07-cert-managed}"

  gcloud compute instance-groups managed describe "$MIG_NAME" --zone "$ZONE" >/dev/null 2>&1 || echo "OK: MIG eliminado"
  gcloud compute instance-templates describe "$TEMPLATE_NAME" >/dev/null 2>&1 || echo "OK: Template global eliminado"
  gcloud compute instance-templates describe "$TEMPLATE_NAME" --region "$REGION" >/dev/null 2>&1 || echo "OK: Template regional eliminado"
  gcloud compute health-checks describe "$HC_NAME" >/dev/null 2>&1 || echo "OK: Health check eliminado"

  gcloud compute addresses describe "$LB_IP_NAME" --global >/dev/null 2>&1 || echo "OK: IP global eliminada"
  gcloud compute ssl-certificates describe "$CERT_CLASSIC_NAME" --global >/dev/null 2>&1 || echo "OK: Cert clásico eliminado"

  gcloud compute routers describe "${ROUTER_NAME:-lab07-router}" --region "$REGION" >/dev/null 2>&1 || echo "OK: Router eliminado"
  gcloud compute routers nats describe "${NAT_NAME:-lab07-nat}" --router "${ROUTER_NAME:-lab07-router}" --region "$REGION" >/dev/null 2>&1 || echo "OK: NAT eliminado"

  gcloud compute networks subnets describe "$SUBNET_NAME" --region "$REGION" >/dev/null 2>&1 || echo "OK: Subnet eliminada"
  gcloud compute networks describe "$VPC_NAME" >/dev/null 2>&1 || echo "OK: VPC eliminada"
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[6] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}