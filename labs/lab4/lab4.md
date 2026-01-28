---
layout: lab
title: "Práctica 4: Crear VPC con 2 subredes, firewall y balanceo de carga interno"
permalink: /lab4/lab4/
images_base: /labs/lab4/img
duration: "45 minutos"
objective:
  - "Diseñar una **VPC en modo custom** con **2 subredes** (app + cliente), crear **reglas de firewall** mínimas (IAP/SSH, tráfico interno y health checks) y desplegar un **Internal Passthrough Network Load Balancer (TCP)** que distribuya tráfico HTTP (puerto 80) hacia un **Managed Instance Group**, validando todo principalmente por **interfaz gráfica** y con evidencias por **Cloud Shell (gcloud/gsutil)**."
prerequisites:
  - "Proyecto de Google Cloud con permisos de **Compute Engine**, **VPC** y **Cloud Load Balancing** (Compute Admin o permisos equivalentes)."
  - "Acceso a **Google Cloud Console** y **Cloud Shell**."
  - "Navegador con sesión activa en Google."
introduction: |
  Una VPC en Google Cloud define tu red privada: subredes regionales, reglas de firewall (stateful) y rutas. En una VPC **custom mode**, tú decides qué subredes existen por región.

  Para exponer un servicio **solo interno**, puedes usar un **Internal passthrough Network Load Balancer (L4)**: crea un **forwarding rule interno** (VIP privado) que distribuye tráfico hacia un **backend service** (p. ej., un managed instance group) y usa **health checks** para enviar tráfico solo a instancias saludables. 

  En esta práctica, construirás todo de forma controlada y verificable: VPC + subredes + firewall + MIG + ILB, y probarás el acceso desde una VM “cliente” dentro de la misma VPC.
slug: lab4
lab_number: 4
final_result: >
  Al finalizar, tendrás una VPC custom con dos subredes, reglas de firewall mínimas y un Internal passthrough Network Load Balancer (TCP/80) con un VIP privado que balancea tráfico hacia un Managed Instance Group de servidores web. Validarás por UI y por CLI: redes, subredes, firewall, health checks, backend health, forwarding rule y pruebas con curl desde una VM cliente (vía IAP).
notes:
  - "Costos: Compute Engine (VMs, discos) tiene costo."
  - "Cloud Load Balancing cobra por reglas de reenvío y datos procesados (según el tipo)."
  - "Una VPC y subredes no suelen tener costo directo"
  - "Las **VPC firewall rules** clásicas no cobran por regla, pero habilitar logging/telemetría puede incurrir costos."
references:
  - text: "Crear y administrar VPC networks (custom mode) (Doc oficial)"
    url: https://docs.cloud.google.com/vpc/docs/create-modify-vpc-networks
  - text: "VPC firewall rules (Doc oficial)"
    url: https://docs.cloud.google.com/firewall/docs/firewalls
  - text: "Internal passthrough Network Load Balancer: overview (Doc oficial)"
    url: https://docs.cloud.google.com/load-balancing/docs/internal
  - text: "Set up an internal passthrough Network Load Balancer (Tutorial oficial)"
    url: https://docs.cloud.google.com/load-balancing/docs/internal/setting-up-internal
  - text: "Firewall rules requeridas para Load Balancing (incluye rangos de health checks)"
    url: https://docs.cloud.google.com/load-balancing/docs/firewall-rules
  - text: "IAP TCP forwarding (rango 35.235.240.0/20 para SSH por IAP)"
    url: https://docs.cloud.google.com/iap/docs/using-tcp-forwarding
  - text: "Precios de Compute Engine"
    url: https://cloud.google.com/compute/all-pricing
  - text: "Precios de Cloud Load Balancing"
    url: https://cloud.google.com/load-balancing/pricing
  - text: "Network pricing / forwarding rules"
    url: https://cloud.google.com/vpc/network-pricing
prev: /lab3/lab3/
next: /lab5/lab5/
---

---

## Instrucciones generales

- Esta práctica es **mayormente por interfaz gráfica (UI)**; Cloud Shell se usa para:
  - guardar variables,
  - validar recursos creados (describe/list),
  - probar el balanceo desde una VM cliente.
- Trabajaremos en una carpeta por práctica dentro de Cloud Shell:
  - `~/labs-gcp-engineer/lab04/` con `scripts/`, `outputs/`.
- Recomendación para el lab (ajusta si tu curso define región):
  - Región: `us-central1`
  - Zonas: `us-central1-a` y `us-central1-b`

---

### Tarea 1. Preparar Cloud Shell y variables del laboratorio (Tiempo estimado: 5 min)

En esta tarea habilitarás Cloud Shell, confirmarás el proyecto activo y crearás la estructura de carpetas con un archivo de variables para no repetir valores a mano.

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

- {% include step_label.html %} Crea la carpeta del laboratorio 4.

  ```bash
  cd ~
  mkdir -p labs-gcp-engineer/lab04/{scripts,outputs}
  ```
  ```bash
  cd labs-gcp-engineer/lab04
  ```
  {% include step_image.html %}

- {% include step_label.html %} Crea el archivo `scripts/env.sh` con variables del laboratorio.

  ```bash
  cat > scripts/env.sh <<'EOF'
  # Región / Zonas (ajusta si tu curso lo define)
  export REGION="us-central1"
  export ZONE_A="us-central1-a"
  export ZONE_B="us-central1-b"

  # VPC y subredes
  export VPC_NAME="lab04-vpc"
  export SUBNET_APP="subnet-app"
  export SUBNET_CLIENT="subnet-client"

  # Rangos RFC1918 (ajusta si hay conflictos)
  export CIDR_APP="10.10.10.0/24"
  export CIDR_CLIENT="10.10.20.0/24"

  # Recursos del backend
  export TEMPLATE_NAME="tpl-web"
  export MIG_NAME="mig-web"
  export BACKEND_TAG="web-backend"

  # ILB (Internal TCP LB)
  export HC_NAME="hc-web-80"
  export BACKEND_SVC="bs-web-80"
  export FR_NAME="fr-ilb-web-80"
  export ILB_IP_NAME="ilb-ip"
  export ILB_PORT="80"
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
  gcloud config set compute/zone "$ZONE_A"
  ```
  ```bash
  gcloud config list --format="yaml(core.project,compute.region,compute.zone)" | tee outputs/gcloud_config.yaml
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[0] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 2. Crear VPC custom con 2 subredes (Tiempo estimado: 10 min)

Crearás una VPC en modo custom y dos subredes en la misma región: una para backends (app) y otra para clientes/pruebas.

> **IMPORTANTE:** En VPC custom, tú defines subredes por región. No podrás crear VMs en una región sin subnet definida.
{: .lab-note .important .compact}

#### Tarea 2.1 (UI) — Crear VPC y subredes

- {% include step_label.html %} En la consola: **VPC network** y luego **VPC networks**

  {% include step_image.html %}

- {% include step_label.html %} Da clic en la opción **Create VPC network**.

  > **NOTA:** La VPC es el contenedor de subredes, firewall rules y rutas.
  {: .lab-note .info .compact}

  {% include step_image.html %}

- {% include step_label.html %} Configura los siguientes datos:

  > **NOTA:** Custom te permite definir tus CIDRs y región de subredes.
  {: .lab-note .info .compact}

  - Name: `lab04-vpc`
  - Subnet creation mode: **Custom**

  {% include step_image.html %}

- {% include step_label.html %} Crea la Subnet 1 (APP):

  > **NOTA:** La subnet-app aloja los servidores web. Private Google Access permite llegar a APIs de Google sin IP pública (tema avanzado).
  {: .lab-note .info .compact}

  - Name: `subnet-app`
  - Region: `us-central1`
  - IP stack type: **IPv4 (single-stack)**
  - IPv4 range: `10.10.10.0/24`
  - Private Google Access: **ON**

  {% include step_image.html %}

- {% include step_label.html %} Agrega una nueva subred clic en el botón **Add Subnet** (CLIENT):

  > **NOTA:** La subnet-client aloja una VM “cliente” desde donde probarás el VIP interno del balanceador.
  {: .lab-note .info .compact}
  
  - Name: `subnet-client`
  - Region: `us-central1`
  - IPv4 range: `10.10.20.0/24`
  - Clic en **Done**

  {% include step_image.html %}

- {% include step_label.html %} Clic en el botón **Create** al final de la configuración.

#### Tarea 2.2 (Cloud Shell) — Confirmar VPC y subredes

- {% include step_label.html %} **Desde el Cloud Shell** verifica la VPC recien creada.

  > **NOTA:** Confirma que la red es custom (`autoCreateSubnetworks: false`).
  {: .lab-note .info .compact}

  ```bash
  source scripts/env.sh
  gcloud compute networks describe "$VPC_NAME" --format="yaml(name,subnetworks,autoCreateSubnetworks,routingConfig.routingMode)" \
    | tee outputs/vpc_describe.yaml
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica ambas subredes.

  ```bash
  gcloud compute networks subnets describe "$SUBNET_APP" --region "$REGION" \
    --format="yaml(name,region,ipCidrRange,privateIpGoogleAccess,network)" \
    | tee outputs/subnet_app.yaml
  ```
  {% include step_image.html %}
  ```bash
  gcloud compute networks subnets describe "$SUBNET_CLIENT" --region "$REGION" \
    --format="yaml(name,region,ipCidrRange,privateIpGoogleAccess,network)" \
    | tee outputs/subnet_client.yaml
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[1] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 3. Crear reglas de firewall mínimas (Tiempo estimado: 10 min)

Definirás reglas para:

- 1) permitir **SSH por IAP** a VMs sin IP pública,
- 2) permitir tráfico **cliente → backend** al puerto 80,
- 3) permitir **health checks** de Google hacia los backends (requerido por LB).

> **Rangos importantes:**
> - IAP TCP forwarding: `35.235.240.0/20` (para SSH via IAP).
> - Health checks (IPv4): `35.191.0.0/16` y `130.211.0.0/22`.
{: .lab-note .info .compact}

#### Tarea 3.1 (UI) — Regla: permitir SSH por IAP

- {% include step_label.html %} En consola da clic en: **VPC network** y luego **Firewall**. Si ya estas en el menú busca solo **Firewall**

  {% include step_image.html %}

- {% include step_label.html %} Ahora da clic en **Create firewall rule**.

  > **NOTA:** En una VPC nueva no existe allow-ssh “por defecto”. Debes crear reglas explícitas.
  {: .lab-note .info .compact}

  {% include step_image.html %}

- {% include step_label.html %} Configura los siguientes datos:

  > **NOTA:** Solo VMs con tag `iap-ssh` aceptarán SSH a través de IAP (mejor seguridad).
  {: .lab-note .info .compact}

  - Name: `fw-allow-iap-ssh`
  - Network: `lab04-vpc`
  - Direction: **Ingress**
  
  {% include step_image.html %}

  - Targets: **Specified target tags**
  - Target tags: `iap-ssh`
  - Source IPv4 ranges: `35.235.240.0/20`
  - Protocols/ports: `tcp:22`

  {% include step_image.html %}

- {% include step_label.html %} Clic en el botón **Create** al final de la pagina.

#### Tarea 3.2 (UI) — Regla: permitir tráfico del cliente al backend (HTTP 80)

- {% include step_label.html %} Crea otra firewall rule con los siguientes datos:

  > **NOTA:** Como el ILB preserva el IP del cliente (L4 passthrough), el backend debe permitir tráfico desde la subnet-client para el puerto 80.
  {: .lab-note .info .compact}

  - Name: `fw-allow-client-to-web`
  - Network: `lab04-vpc`
  - Direction: **Ingress**

  {% include step_image.html %}

  - Targets: **Specified target tags**
  - Target tags: `web-backend`
  - Source IPv4 ranges: `10.10.20.0/24` (CIDR de subnet-client)
  - Protocols/ports: `tcp:80`

  {% include step_image.html %}

- {% include step_label.html %} Clic en el botón **Create** al final de la pagina.

#### Tarea 3.3 (UI) — Regla: permitir health checks a backends

- {% include step_label.html %} Crea otra firewall rule:

  > **NOTA:** Si no permites estos rangos, el health check no llega a tus VMs y el LB marcará backends como “UNHEALTHY”.
  {: .lab-note .info .compact}

  - Name: `fw-allow-hc-to-web`
  - Network: `lab04-vpc`
  - Direction: **Ingress**

  {% include step_image.html %}

  - Targets: **Specified target tags**
  - Target tags: `web-backend`
  - Source IPv4 ranges: `35.191.0.0/16, 130.211.0.0/22`
  - Protocols/ports: `tcp:80`

  {% include step_image.html %}

- {% include step_label.html %} Clic en el botón **Create** al final de la pagina.

#### Tarea 3.4 (Cloud Shell) — Evidencias de firewall

- {% include step_label.html %} **Desde Cloud Shell** Lista las reglas del lab y guarda evidencia.

  ```bash
  source scripts/env.sh
  gcloud compute firewall-rules list \
    --filter="network:$VPC_NAME AND (name:fw-allow-iap-ssh OR name:fw-allow-client-to-web OR name:fw-allow-hc-to-web)" \
    --format="table(name,direction,network,disabled,priority,sourceRanges.list():label=SRC_RANGES,allowed[].map().firewall_rule().list():label=ALLOW,targetTags.list():label=TAGS)" \
    | tee outputs/firewall_rules_table.txt
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[2] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 4. Desplegar backends web con Managed Instance Group (Tiempo estimado: 10 min)

Crearás un **Instance Template** con un **startup script** que instala NGINX y publica una página simple con estilo. Después crearás un **Managed Instance Group** en `subnet-app`.

#### Tarea 4.1 (UI) — Crear Instance Template

- {% include step_label.html %} Primero **Dentro de Cloud Shell** crea un Cloud NAT temporal para la descarga de dependencias de la practica, ejecuta todo el codigo sisguiente

  ```bash
  REGION="us-central1"
  NETWORK="lab04-vpc"
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

- {% include step_label.html %} Ve a **Compute Engine** y luego a **Instance templates**.

  {% include step_image.html %}

- {% include step_label.html %} Da clic en **Create instance template**.

  > **NOTA:** El template define “cómo se ven” tus VMs: máquina, disco, red y startup script.
  {: .lab-note .info .compact}

  {% include step_image.html %}

- {% include step_label.html %} Configura los siguientes datos:

  - Name: `tpl-web`
  - Location: **Regional**
  - Region: **us-central1 (iowa)**

  {% include step_image.html %}

  - Machine type: **e2-medium**

  {% include step_image.html %}
  
  - Boot disk: **Debian 12**

  {% include step_image.html %}

  - Advanced options/Networking/Network interfaces:
    - Network: `lab04-vpc`
    - Subnetwork: `subnet-app`
    - External IPv4: **None** (sin IP pública)
    - Clic en **Done**

  {% include step_image.html %}

- {% include step_label.html %} En la sección **Advanced options** luego **Management** da clic en **Startup script**, pega el siguiente script:

  > **NOTA:** La página mostrara el hostname para evidenciar el balanceo.
  {: .lab-note .info .compact}

  ```bash
  #!/bin/bash
  set -euo pipefail
  export DEBIAN_FRONTEND=noninteractive

  sudo apt-get update -y
  sudo apt-get install -y nginx

  HOST="$(hostname)"
  TS="$(date -u +%FT%TZ)"

  tee /var/www/html/index.html >/dev/null <<EOF
  <!doctype html>
  <html>
    <head>
      <meta charset="utf-8">
      <title>Lab04 - Internal LB</title>
      <style>
        body { font-family: Arial, sans-serif; margin: 40px; background: #0b1220; color: #e6eefc; }
        .card { max-width: 720px; padding: 24px; border-radius: 14px; background: rgba(255,255,255,0.06); box-shadow: 0 8px 24px rgba(0,0,0,0.35); }
        h1 { margin: 0 0 12px 0; font-size: 28px; }
        .pill { display: inline-block; padding: 6px 10px; border-radius: 999px; background: rgba(126,231,135,0.18); border: 1px solid rgba(126,231,135,0.35); }
        code { background: rgba(255,255,255,0.08); padding: 2px 6px; border-radius: 6px; }
      </style>
    </head>
    <body>
      <div class="card">
        <h1>✅ Internal Load Balancer - Backend</h1>
        <p class="pill">Servidor: <b>${HOST}</b></p>
        <p>Timestamp (UTC): <code>${TS}</code></p>
        <p>Si refrescas varias veces desde el cliente, deberías ver cambios de <b>hostname</b> cuando el LB distribuya tráfico.</p>
      </div>
    </body>
  </html>
  EOF

  sudo systemctl enable --now nginx
  ```
  {% include step_image.html %}

- {% include step_label.html %} En la sección **Advanced options** **Networking** agrega las 2 etiquetas en **Networking tags**:

  - `web-backend`
  - `iap-ssh`

  {% include step_image.html %}

- {% include step_label.html %} Finalmente clic en botón **Create**.

- {% include step_label.html %} Tendras una plantilla creada llamada **tpl-web**

  {% include step_image.html %}

#### Tarea 4.2 (UI) — Crear Managed Instance Group

- {% include step_label.html %} Ve a **Compute Engine** luego en  **Instance groups** 

  {% include step_image.html %}

- {% include step_label.html %} Clic en **Create instance group**.

  {% include step_image.html %}

- {% include step_label.html %} Configura los siguientes datos:

  > **NOTA:** Un MIG te da uniformidad (misma configuración) y facilita que LB tenga backends homogéneos.
  {: .lab-note .info .compact}

  - Type: **New managed instance group (stateless)**
  - Name: `mig-web`
  - Instance template: `tpl-web`
  - Number of instances: **2**

  {% include step_image.html %}
  
  - Location: **Single zone** `us-central1-a`
  - Autoscaling: **Off: do not autoscale**
  - Minimum number of instances: **2**
  - Maximum number of instances: **3**

  {% include step_image.html %}

- {% include step_label.html %} Clic en el botón **Create** al final de la página.

- {% include step_label.html %} Tendras un **Instance groups** llamado **mig-web**

  {% include step_image.html %}

#### Tarea 4.3 (Cloud Shell) — Evidencias del MIG

- {% include step_label.html %} **Desde Cloud Shell** Lista instancias del MIG y guarda evidencia. Pueden tardar varios minutos, sino aparecen reintenta el comando.

  > **NOTA:** Confirma que el MIG creó VMs y que están “RUNNING”.
  {: .lab-note .info .compact}

  ```bash
  source scripts/env.sh
  gcloud compute instance-groups managed list-instances "$MIG_NAME" --zone "$ZONE_A" \
    --format="table(instance,status,instanceStatus,version.targetSize)" \
    | tee outputs/mig_instances.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Obtén IPs internas de las VMs.

  > **NOTA:** Las VMs viven en subnet-app y solo tienen IP privada.
  {: .lab-note .info .compact}

  ```bash
  INSTANCES="$(gcloud compute instance-groups managed list-instances "$MIG_NAME" --zone "$ZONE_A" --format="value(instance.basename())")"
  for vm in $INSTANCES; do
    gcloud compute instances describe "$vm" --zone "$ZONE_A" \
      --format="value(name,networkInterfaces[0].networkIP)" \
      | tee -a outputs/mig_vm_internal_ips.txt
  done
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[3] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 5. Crear Internal Load Balancer y probar desde VM cliente (Tiempo estimado: 8–10 min)

Configurarás un **Internal passthrough Network Load Balancer** (TCP/80), reservando un VIP interno y asociándolo al backend service con health check.

> **Concepto:** Un forwarding rule interno define el VIP/puerto; el backend service agrupa backends y health check.
{: .lab-note .info .compact}

#### Tarea 5.1 (UI) — Crear VM cliente (sin IP pública) para pruebas internas

- {% include step_label.html %} Ve a **Compute Engine** luego a **VM instances** y da clic en **Create instance**.

  > **NOTA:** Esta VM simula un “cliente interno” que consume el VIP privado del LB.
  {: .lab-note .info .compact}

  **Configura:**
  - Name: `vm-client`
  - Region/Zone: `us-central1-a`

  {% include step_image.html %}

  - Machine type: **e2-medium**
  
  {% include step_image.html %}

  - Networking:
    - Network tags: `iap-ssh`
    - Network interfaces: `lab04-vpc`
    - Subnetwork: `subnet-client`
    - External IPv4: **None**
  
  {% include step_image.html %}
  {% include step_image.html %}

- {% include step_label.html %} Clic en **Create** y espera unos segundos para el siguiente paso.

  {% include step_image.html %}

- {% include step_label.html %} **En Cloud Shell**, prueba SSH por IAP hacia la VM cliente.

  > **NOTA:** Te aseguras de poder ejecutar `curl` desde dentro de la VPC sin IP pública.
  {: .lab-note .info .compact}

  ```bash
  gcloud compute ssh vm-client --zone "$ZONE_A" --tunnel-through-iap --command "hostname && ip -br a | head"
  ```
  {% include step_image.html %}

#### Tarea 5.2 (UI) — Crear el Load Balancer interno (TCP)

- {% include step_label.html %} En el buscador escribe **Load balancing** y da clic en el servicio. 

  {% include step_image.html %}

- {% include step_label.html %} Ahora da clic en **Create load balancer**.

  {% include step_image.html %}

- {% include step_label.html %} Selecciona **Network Load Balancer (TCP/UDP/SSL)** y da clic en **Next**.

  {% include step_image.html %}

- {% include step_label.html %} Luego elige la opción **Passthrough load balancer** y da clic en **Next**.

  > **NOTA:** Passthrough NLB distribuye tráfico TCP/UDP dentro de la VPC.
  {: .lab-note .info .compact}

  {% include step_image.html %}

- {% include step_label.html %} Ahora selecciona **Internal** y da clic en **Next**.

  {% include step_image.html %}

- {% include step_label.html %} Ahora clic en **Configure**.

- {% include step_label.html %} En la configuración escribe los siguientes datos:

  > **NOTA:** Poner el VIP en subnet-client facilita que el “cliente” esté en el mismo segmento lógico, pero cualquier subnet de la VPC puede alcanzarlo.
  {: .lab-note .info .compact}

  - Name: `ilb-web-80` (nombre del LB en UI)
  - Region: `us-central1`
  - Network: **lab04-vpc**

  {% include step_image.html %}

  - Backend configuration/Backend service:
    - Backend type: **Instance group**
    - Protocol: **TCP**
    - Health check/Create a health check:
      - Name: `hc-web-80`
      - Protocol: **TCP**
      - Port: `80`
      - Clic en **Create**

    {% include step_image.html %}
    
  - Backend configuration/Backends:
    - Instance group: **mig-web**
    - Clic en **Done**

  {% include step_image.html %}

  - Frontend configuration:
    - Subnetwork: `subnet-client` (donde vivirá el VIP)
    - IP address: **Create IP address** (nómbrala `ilb-ip`)
    - Ports: `80`

  {% include step_image.html %}

- {% include step_label.html %} Finalmente clic en el botón **Create**

#### Tarea 5.3 (Cloud Shell) — Evidencias del LB y estado de health

- {% include step_label.html %} **Desde Cloud Shell** Obtén el VIP interno del forwarding rule (guárdalo en variable).

  ```bash
  source scripts/env.sh
  # Lista forwarding rules internas en la región (encuentra la del lab)
  gcloud compute forwarding-rules list --regions "$REGION" \
    --format="table(name,loadBalancingScheme,IPAddress,IPProtocol,ports,network,subnetwork)" \
    | tee outputs/forwarding_rules_region.txt
  ```
  {% include step_image.html %}
  ```bash
  export FR_NAME_ACTUAL="$(gcloud compute forwarding-rules list --regions "$REGION" \
    --filter="loadBalancingScheme=INTERNAL AND name~'(ilb|web)'" \
    --format="value(name)" | head -n 1)"

  echo "FR_NAME_ACTUAL=$FR_NAME_ACTUAL" | tee outputs/fr_name_actual.txt
  ```
  {% include step_image.html %}
  ```bash
  ILB_VIP="$(gcloud compute forwarding-rules describe "$FR_NAME_ACTUAL" --region "$REGION" --format='value(IPAddress)')"
  echo "ILB_VIP=$ILB_VIP" | tee outputs/ilb_vip.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Revisa la salud del backend service (debe marcar instancias como HEALTHY).

  > **IMPORTANTE:** Si todo está bien, verás `HEALTHY`. Si sale `UNHEALTHY`, revisa firewall (health check ranges) y NGINX.
  {: .lab-note .important .compact}

  ```bash
  # Guarda la respuesta COMPLETA
  gcloud compute backend-services get-health "$BS_NAME_ACTUAL" --region "$BS_REGION" --format=json \
  | tee outputs/backend_health.json
  ```
  ```bash
  # Muestra solo lo importante (HEALTHY/UNHEALTHY) sin depender de campos exactos
  grep -Eo '"healthState"\s*:\s*"[^"]+"' outputs/backend_health.json | head -n 20
  ```
  {% include step_image.html %}

#### Tarea 5.4 (Cloud Shell → VM cliente) — Probar balanceo con curl

- {% include step_label.html %} Ejecuta un loop de `curl` desde `vm-client` hacia el VIP para ver cambios de hostname.

  > **NOTA:** Si tienes 2 instancias en el MIG, deberías observar alternancia del “Servidor: <hostname>” (no necesariamente 50/50 en 8 requests, pero sí cambios).
  {: .lab-note .info .compact}

  ```bash
  source scripts/env.sh
  ILB_VIP="$(gcloud compute forwarding-rules describe ilb-web-80-forwarding-rule --region "$REGION" --format='value(IPAddress)' | tr -d '\r')"
  echo "ILB_VIP=$ILB_VIP" | tee outputs/ilb_vip.txt
  gcloud compute ssh vm-client --zone "$ZONE_A" --tunnel-through-iap --command "for i in \$(seq 1 8); do echo \"--- Request \$i\"; curl -sS http://$ILB_VIP | grep -E \"Servidor\" || true; sleep 1; done"
  ```
  {% include step_image.html %}

- {% include step_label.html %} (Opcional) Prueba conectividad directa a una VM backend (solo para troubleshooting), usando su IP interna.

  > **IMPORTANTE:** Si esto funciona pero el VIP no, el problema está en LB/health checks/firewall.
  {: .lab-note .important .compact}

  ```bash
  gcloud compute instance-groups managed list-instances mig-web --zone us-central1-a \
  --format="value(instance.basename())" \
  | while read -r VM; do
    IP="$(gcloud compute instances describe "$VM" --zone us-central1-a --format='value(networkInterfaces[0].networkIP)')"
    echo "$VM $IP"
  done | tee outputs/mig_vm_internal_ips.txt
  ```
  {% include step_image.html %}
  ```bash
  BACKEND_IP="$(awk 'NR==1{print $2}' outputs/mig_vm_internal_ips.txt)"
  echo "BACKEND_IP=$BACKEND_IP" | tee outputs/backend_ip_chosen.txt
  ```
  {% include step_image.html %}
  ```bash
  gcloud compute ssh vm-client --zone "$ZONE_A" --tunnel-through-iap --command \
  "curl -m 5 -sS -D- http://$BACKEND_IP/ | head -n 20"
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[4] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 6. Limpieza (Tiempo estimado: 3–5 min)

Elimina recursos para evitar costos.

#### Tarea 6.1 (UI / Cloud Shell) — Borrar Load Balancer y recursos

- {% include step_label.html %} (UI recomendado) En **Network services** luego **Load balancing**, elimina el LB creado (incluye forwarding rule / backend service / health check si aplica).

  > **NOTA:** La UI suele eliminar dependencias en orden correcto.
  {: .lab-note .info .compact}

  {% include step_image.html %}

- {% include step_label.html %} **Desde Cloud Shell** Verifica que ya no existan forwarding rules del lab.

  ```bash
  gcloud compute forwarding-rules list --regions "$REGION" \
    --format="table(name,IPAddress,ports,loadBalancingScheme)" | tee outputs/forwarding_rules_after_cleanup.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} (UI) Elimina los siguientes recursos **Compute engine** luego **VM instances**:

  - VM `vm-client`
  - Managed Instance Group `mig-web`
  - Instance template `tpl-web`

  {% include step_image.html %}

- {% include step_label.html %} (UI) Elimina firewall rules del lab **VPC Network** luego **Firewall**:

  - `fw-allow-iap-ssh`
  - `fw-allow-client-to-web`
  - `fw-allow-hc-to-web`

  {% include step_image.html %}

- {% include step_label.html %} (UI) Elimina el grupo de VMs **Compute Engine** luego **Instance Groups**

  > **NOTA:** Puede tardar un par de minutos. Espera a que se elimine.
  {: .lab-note .info .compact}

  {% include step_image.html %}

- {% include step_label.html %} (UI) Elimina la plantilla de VMs **Compute Engine** luego **Instance templates**

  {% include step_image.html %}

- {% include step_label.html %} Ahora **Dentro de Cloud Shell** ejecuta el siguiente comando para eliminar el **Cloud NAT**

  ```bash
  gcloud compute routers nats delete "$NAT_NAME" \
    --router="$ROUTER_NAME" \
    --region="$REGION" -q

  gcloud compute routers delete "$ROUTER_NAME" \
    --region="$REGION" -q 
  ```
  {% include step_image.html %}

- {% include step_label.html %} (UI) Elimina la VPC `lab04-vpc` **VPC Network** luego **VPC Networks**.

  > **NOTA:** La VPC no se puede borrar si aún hay dependencias (subnets, reglas, instancias, forwarding rules).
  {: .lab-note .info .compact}

  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[5] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}