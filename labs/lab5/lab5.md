---
layout: lab
title: "Práctica 5: Implementar Cloud SQL y conectarlo con una VM de aplicación"
permalink: /lab5/lab5/
images_base: /labs/lab5/img
duration: "50 minutos"
objective:
  - "Implementar una instancia **Cloud SQL for MySQL** con **IP privada** (Private Service Access), crear una **VM de aplicación** en la misma VPC, conectar de forma segura mediante **Cloud SQL Auth Proxy**, y desplegar una **app web simple** que consulta la base de datos, validando la configuración principalmente por **interfaz gráfica** y usando **Cloud Shell** solo cuando sea necesario."
prerequisites:
  - "Proyecto de Google Cloud con permisos para **Compute Engine**, **VPC**, **Cloud SQL**, **Service Networking** e **IAM** (Owner/Editor o roles equivalentes para laboratorio)."
  - "Acceso a **Google Cloud Console** y **Cloud Shell**."
  - "Navegador con sesión activa en Google."
introduction: |
  Cloud SQL es un servicio administrado de bases de datos relacionales. En escenarios empresariales, es común desplegarlo con **IP privada**, lo que requiere **Private Service Access (Service Networking)** en la VPC usada por la aplicación. Para conectar desde una VM de forma segura y simplificar autenticación/seguridad, se recomienda usar **Cloud SQL Auth Proxy**, que expone un puerto local (por ejemplo `127.0.0.1:3306`) y se conecta a Cloud SQL usando IAM/credenciales del entorno. En esta práctica crearás toda la ruta: VPC → Private Service Access → Cloud SQL (private IP) → VM sin IP pública → Proxy → App web.
slug: lab5
lab_number: 5
final_result: >
  Al finalizar, tendrás una VPC dedicada del laboratorio con salida a internet vía Cloud NAT, una instancia Cloud SQL MySQL con IP privada, una VM de aplicación conectada por Cloud SQL Auth Proxy y una app web (Flask) que muestra registros consultados desde MySQL, con evidencias por UI y comandos.
notes:
  - "**Costos:** Cloud SQL y Compute Engine tienen costo (vCPU/RAM, almacenamiento, y tiempo de ejecución). Mantén tamaños pequeños y elimina recursos al final."
  - "Private Service Access requiere habilitar Service Networking y reservar un rango interno para servicios administrados."
  - "Para entornos reales: aplica mínimo privilegio IAM, rotación de credenciales, y monitoreo/alertas (Cloud Monitoring)."
references:
  - text: "Conectar desde Compute Engine usando Cloud SQL Auth Proxy (MySQL)"
    url: https://cloud.google.com/sql/docs/mysql/connect-compute-engine
  - text: "Acerca del Cloud SQL Auth Proxy"
    url: https://cloud.google.com/sql/docs/mysql/sql-proxy
  - text: "Conectarse con el proxy de autenticación de Cloud SQL (MySQL)"
    url: https://cloud.google.com/sql/docs/mysql/connect-auth-proxy?hl=es-419
  - text: "Configurar IP privada en Cloud SQL (MySQL)"
    url: https://cloud.google.com/sql/docs/mysql/configure-private-ip
  - text: "Configurar Private Service Access (MySQL)"
    url: https://cloud.google.com/sql/docs/mysql/configure-private-services-access
  - text: "Usar IAP para TCP forwarding (para SSH a VM sin exponer puertos)"
    url: https://cloud.google.com/iap/docs/using-tcp-forwarding
  - text: "Cloud SQL pricing"
    url: https://cloud.google.com/sql/pricing
  - text: "Compute Engine pricing"
    url: https://cloud.google.com/compute/all-pricing
  - text: "Cloud NAT overview"
    url: https://cloud.google.com/nat/docs/overview
prev: /lab4/lab4/
next: /lab6/lab6/
---

---

## Instrucciones generales

- Esta práctica es **mayormente por interfaz gráfica (UI)**:
  - Creación de VPC, Cloud NAT, Cloud SQL y VM desde la consola.
- Usaremos **Cloud Shell** para:
  - crear estructura de carpetas y guardar evidencias,
  - validar recursos con `gcloud`,
  - conectarnos por SSH vía IAP a la VM para instalar y ejecutar el proxy y la aplicación.
- Trabajaremos en una carpeta por práctica dentro de Cloud Shell:
  - `~/labs-gcp-engineer/lab05/` con `scripts/` y `outputs/`.

---

### Tarea 1. Preparar Cloud Shell y variables del laboratorio (Tiempo estimado: 5 min)

En esta tarea dejarás listo Cloud Shell, confirmarás el proyecto y crearás la estructura de carpetas con variables del laboratorio.

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

- {% include step_label.html %} Crea la carpeta del laboratorio 5.

  > **NOTA:** `scripts/` guarda archivos reproducibles (env/plantillas) y `outputs/` guarda evidencia de validación.
  {: .lab-note .info .compact}

  ```bash
  cd ~
  mkdir -p labs-gcp-engineer/lab05/{scripts,outputs}
  ```
  ```bash
  cd labs-gcp-engineer/lab05
  ```
  {% include step_image.html %}

- {% include step_label.html %} Crea el archivo `scripts/env.sh` con variables del lab (nombres alineados a **Práctica 5**).

  ```bash
  cat > scripts/env.sh <<'EOF'
  # Región / zona
  export REGION="us-central1"
  export ZONE="us-central1-a"

  # Red
  export VPC_NAME="lab05-vpc"
  export SUBNET_APP="lab05-subnet-app"
  export SUBNET_CIDR="10.50.10.0/24"

  # Private Service Access (rango para servicios administrados)
  export PSA_RANGE_NAME="lab05-psa-range"
  export PSA_RANGE_CIDR="10.50.20.0/24"

  # Cloud NAT
  export ROUTER_NAME="lab05-router"
  export NAT_NAME="lab05-nat"

  # Cloud SQL
  export SQL_INSTANCE="lab05-sql-mysql"
  export SQL_DB="appdb"
  export SQL_USER="appuser"

  # VM App
  export VM_NAME="lab05-vm-app"
  export VM_TAG_IAP="lab05-iap-ssh"
  EOF
  ```
  ```bash
  source scripts/env.sh
  ```

- {% include step_label.html %} Configura región y zona por defecto y guarda evidencia.

  > **NOTA:** Así `gcloud` asumirá región/zona en comandos posteriores; el `yaml` sirve como evidencia del contexto.
  {: .lab-note .info .compact}

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

### Tarea 2. Crear VPC (custom) + Cloud NAT para salida (Tiempo estimado: 12 min)

En esta tarea crearás una VPC dedicada al laboratorio con una subnet para la VM. Luego, crearás **Cloud NAT** para que la VM (sin IP pública) pueda instalar paquetes y descargar el Cloud SQL Auth Proxy.

#### Tarea 2.1 (UI) — Crear VPC y subnet

- {% include step_label.html %} En consola: **VPC network** luego en **VPC networks**.

  {% include step_image.html %}

- {% include step_label.html %} Ahora da clic en **Create VPC network**

  > **NOTA:** Usar una VPC dedicada evita conflictos con reglas y rutas del “default” y hace el laboratorio más claro.
  {: .lab-note .info .compact}

  {% include step_image.html %}

- {% include step_label.html %} Configura los siguientes datos:

  > **NOTA:** “Custom” te permite controlar CIDRs y regiones, como lo harías en un diseño real.
  {: .lab-note .info .compact}

  - Name: `lab05-vpc`
  - Subnet creation mode: **Custom**

  {% include step_image.html %}

- {% include step_label.html %} Crea la subnet de aplicación:

  > **NOTA:** Private Google Access ayuda a consumos de APIs/servicios de Google sin IP pública (cuando aplica).
  {: .lab-note .info .compact}

  - Subnet name: `lab05-subnet-app`
  - Region: `us-central1`
  - IPv4 range: `10.50.10.0/24`
  - Private Google Access: **ON**
  - Clic en **Done**

  {% include step_image.html %}

- {% include step_label.html %} Clic en **Create**.

- {% include step_label.html %} **En Cloud Shell**, confirma red y subnet.

  > **NOTA:** Esto valida que la VPC no sea “auto” y que el CIDR de la subnet sea el esperado.
  {: .lab-note .info .compact}

  ```bash
  source scripts/env.sh
  gcloud compute networks describe "$VPC_NAME" --format="yaml(name,autoCreateSubnetworks,subnetworks)" | tee outputs/vpc.yaml
  ```
  {% include step_image.html %}
  ```bash
  gcloud compute networks subnets describe "$SUBNET_APP" --region "$REGION" --format="yaml(name,ipCidrRange,privateIpGoogleAccess,network)" | tee outputs/subnet_app.yaml
  ```
  {% include step_image.html %}

#### Tarea 2.2 (UI) — Crear Cloud Router + Cloud NAT

- {% include step_label.html %} En consola: En el buscador escribe **Cloud NAT** y da clic en el servicio.

  {% include step_image.html %}

- {% include step_label.html %} Clic en **Get started**

  > **NOTA:** Cloud NAT da salida a internet a VMs sin IP pública (apt-get, curl, pip, descarga del proxy, etc.).
  {: .lab-note .info .compact}

  {% include step_image.html %}

- {% include step_label.html %} Configura los siguientes datos:

  > **NOTA:** Aplicar NAT solo a la subnet del lab reduce el alcance y el costo.
  {: .lab-note .info .compact}

  - Gateway name: `lab05-nat`
  - NAT type: **Public**

  {% include step_image.html %}

  - Select Cloud Router:
    - Network: **lab05-vpc**
    - Region: **us-central1 (Iowa)**
  - Cloud Router: **Create new router**
    - Router name: `lab05-router`
    - Clic **Create**
  
  {% include step_image.html %}

  - Cloud NAT mapping:
    - IPv4 subnet ranges: **Custom**
    - Subnets: **lab05-subnet-app**
    - IP ranges 1: **Selected: all**
  
  {% include step_image.html %}

  - Network Service Tier: **Standard**

  {% include step_image.html %}

- {% include step_label.html %} Clic en el botón **Create**.

- {% include step_label.html %} **En Cloud Shell**, confirma router y NAT.

  > **NOTA:** Confirma que NAT “apunta” a la subnet correcta.
  {: .lab-note .info .compact}

  ```bash
  source scripts/env.sh
  gcloud compute routers describe "$ROUTER_NAME" --region "$REGION" --format="yaml(name,network)" | tee outputs/router.yaml
  ```
  {% include step_image.html %}
  ```bash
  gcloud compute routers nats describe "$NAT_NAME" --router "$ROUTER_NAME" --router-region "$REGION" --format="yaml(name,natIpAllocateOption,sourceSubnetworkIpRangesToNat,subnetworks)" | tee outputs/nat.yaml
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[1] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 3. Configurar Private Service Access y crear Cloud SQL con IP privada (Tiempo estimado: 18 min)

En esta tarea reservarás un rango para **Private Service Access** y lo asociarás a la VPC. Luego crearás una instancia Cloud SQL MySQL con **solo IP privada**, crearás una base y un usuario de aplicación.

#### Tarea 3.1  — Habilitar APIs necesarias

- {% include step_label.html %} **En Cloud Shell** ejecuta el siguiente comando para habilitar las APIs faltantes:

  > **NOTA:** Private Service Access usa Service Networking para crear el peering administrado de servicios.
  {: .lab-note .info .compact}

  ```bash
  gcloud services enable \
    sqladmin.googleapis.com \
    servicenetworking.googleapis.com \
    compute.googleapis.com
  ```

  {% include step_image.html %}

- {% include step_label.html %} **En Cloud Shell**, confirma APIs habilitadas (evidencia).

  > **IMPORTANTE:**  Si falta una API, la UI puede fallar o pedir permisos adicionales.
  {: .lab-note .important .compact}

  ```bash
  gcloud services list --enabled --filter="name:sqladmin.googleapis.com OR name:servicenetworking.googleapis.com OR name:compute.googleapis.com" \
    --format="table(name,title)" | tee outputs/enabled_apis.txt
  ```
  {% include step_image.html %}

#### Tarea 3.2 (UI) — Private Service Access (reservar rango + conectar)

- {% include step_label.html %} En consola: **VPC network** luego en **VPC networks** da clic en tu red **lab05-vpc** finalmente en **Private service access**.

  > **NOTA:** Este paso “prepara la carretera interna” para que Cloud SQL tenga IP privada dentro de tu red.
  {: .lab-note .info .compact}

  {% include step_image.html %}

- {% include step_label.html %} En **Allocated IP ranges for services**, clic **Allocate IP range** y configura:

  > **NOTA:** Este bloque de IPs se “aparta” para servicios administrados; evita solaparse con tus subnets.
  {: .lab-note .info .compact}

  - Name: `lab05-psa-range`
  - IP range: `10.50.20.0/24`
  - Clic en **Allocate**

  {% include step_image.html %}

- {% include step_label.html %} **En Cloud Shell** ejecuta el siguiente comando para vincular el rango a `servicenetworking.googleapis.com`.

  > **NOTA:** Esto crea un peering administrado entre tu VPC y la red de servicios de Google.
  {: .lab-note .info .compact}

  ```bash
  source scripts/env.sh
  gcloud services vpc-peerings connect \
    --service=servicenetworking.googleapis.com \
    --network="$VPC_NAME" \
    --ranges="$PSA_RANGE_NAME"
  ```
  {% include step_image.html %}

- {% include step_label.html %} **En Cloud Shell**, lista peerings como evidencia (puede tardar unos segundos).

  > **NOTA:** Si no aparece el peering o está en estado incorrecto, Cloud SQL con IP privada fallará.
  {: .lab-note .info .compact}
  
  ```bash
  source scripts/env.sh
  gcloud services vpc-peerings list \
    --network="$VPC_NAME" \
    --service=servicenetworking.googleapis.com \
    --format="table(peering,network,reservedPeeringRanges.list())" \
  | tee outputs/peerings.txt
  ```
  {% include step_image.html %}

#### Tarea 3.3 (UI) — Crear Cloud SQL (MySQL) con IP privada

- {% include step_label.html %} En consola: clic en **Cloud SQL** luego en **Instances**.

  {% include step_image.html %}

- {% include step_label.html %} Da clic en **Create instance** el botón se encuentra en la parte de abajo.

  > **NOTA:** MySQL te permite un ejemplo directo con cliente estándar y librería simple.
  {: .lab-note .info .compact}

  {% include step_image.html %}

- {% include step_label.html %} Da clic en **Choose MySQL**.

  > **NOTA:** MySQL te permite un ejemplo directo con cliente estándar y librería simple.
  {: .lab-note .info .compact}

  {% include step_image.html %}

- {% include step_label.html %} Configura los siguientes datos:

  - Choose a Cloud SQL edition: **Enterprise**
  - Edition preset: **Sandbox**

  {% include step_image.html %}

  - Instance ID: `lab05-sql-mysql`
  - Password: `Pa55w.rd123456!`
  - Region: **us-central1 (Iowa)**
  - Zonal availability: **Single zone**

  {% include step_image.html %}

- {% include step_label.html %} En **Connections** clic en **Networking**

  > **NOTA:**Con esto, la base no queda expuesta por IP pública.
  {: .lab-note .info .compact}

  - Customize your instance/Connections
    - Activa **Private IP**
    - VPC Network: selecciona `lab05-vpc`

  {% include step_image.html %} 

- {% include step_label.html %} Clic **Create instance** (la provisión puede tardar varios minutos).

- {% include step_label.html %} **En Cloud Shell**, confirma que la instancia está **RUNNABLE** y captura el **Connection name** (evidencia).

  > **NOTA:** `connectionName` es el identificador que usa el proxy para llegar a tu instancia.
  {: .lab-note .info .compact}

  ```bash
  source scripts/env.sh
  gcloud sql instances describe "$SQL_INSTANCE" --format="value(state)" | tee outputs/sql_state.txt
  ```
  {% include step_image.html %}   
  ```bash
  gcloud sql instances describe "$SQL_INSTANCE" --format="value(connectionName)" | tee outputs/sql_connection_name.txt
  ```
  {% include step_image.html %}   
  ```bash
  gcloud sql instances describe "$SQL_INSTANCE" --format="yaml(name,region,state,ipAddresses,settings.ipConfiguration.privateNetwork)" \
    | tee outputs/sql_instance.yaml
  ```
  {% include step_image.html %} 

#### Tarea 3.4 (UI) — Crear DB y usuario de aplicación

- {% include step_label.html %} Clic en el nombre de la instancia `lab05-sql-mysql`:

  > **NOTA:** Separar DB por app es práctica estándar y facilita migraciones.
  {: .lab-note .info .compact}

  - Ve a **Databases** clic en **Create database**
  - Database name: `appdb`
  - Clic en **Create**

  {% include step_image.html %} 

- {% include step_label.html %} Ve a **Users** y clic en **Add user account**:

  > **NOTA:** En producción restringes host y privilegios; aquí usamos `%` para agilizar.
  {: .lab-note .info .compact}

  - Username: `appuser`
  - Password: `Pa55w.rd09876`
  - Hostname: **Allow any host (%)**
  - Clic en **Add**

  {% include step_image.html %} 

- {% include step_label.html %} **En Cloud Shell**, confirma DB y usuarios.

  > **NOTA:** Verifica que `appdb` y `appuser` estén creados antes de avanzar.
  {: .lab-note .info .compact}

  ```bash
  source scripts/env.sh
  gcloud sql databases list --instance "$SQL_INSTANCE" --format="table(name,charset,collation)" | tee outputs/sql_databases.txt
  ```
  {% include step_image.html %} 
  ```bash
  gcloud sql users list --instance "$SQL_INSTANCE" --format="table(name,host,type)" | tee outputs/sql_users.txt
  ```
  {% include step_image.html %} 

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[2] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 4. Crear VM de aplicación y conectar con Cloud SQL Auth Proxy (Tiempo estimado: 12–13 min)

En esta tarea crearás una VM en la VPC del laboratorio y conectarás a Cloud SQL usando **Cloud SQL Auth Proxy** con un **service account** que tenga el rol **Cloud SQL Client**.

#### Tarea 4.1 (UI) — Crear Service Account para la VM

- {% include step_label.html %} En consola: **IAM & Admin** luego en **Service Accounts**.  

  {% include step_image.html %}

- {% include step_label.html %} Da clic en **Create service account**.

  > **NOTA:** Este SA le da identidad a la VM; así Cloud SQL Auth Proxy puede autenticarse con IAM.
  {: .lab-note .info .compact}

  {% include step_image.html %}

- {% include step_label.html %} Configura los siguientes datos:

  - Name: `sa-lab05-app`
  - ID: `sa-lab05-app`
  - Create and continue

  {% include step_image.html %}

- {% include step_label.html %} Asigna el siguiente rol:

  > **NOTA:** `Cloud SQL Client` suele ser el mínimo típico para conectarse mediante proxy.
  {: .lab-note .info .compact}

  - Select a role: **Cloud SQL Client**

  {% include step_image.html %}

- {% include step_label.html %} Clic en **Done**.

- {% include step_label.html %} **En Cloud Shell**, verifica el service account.

  ```bash
  gcloud iam service-accounts list --filter="email:sa-lab05-app" --format="table(email,displayName)" | tee outputs/service_account.txt
  ```
  {% include step_image.html %}

#### Tarea 4.2 (UI) — Crear VM (sin IP pública) y acceso por IAP

- {% include step_label.html %} En consola: **Compute Engine** luego en **VM instances**.

- {% include step_label.html %} Clic en **Create instance**.

  {% include step_image.html %}

- {% include step_label.html %} Configura los siguientes datos:

  > **NOTA:** Sin IP pública reduces superficie de ataque; administrarás por IAP.
  {: .lab-note .info .compact}

  - Name: `lab05-vm-app`
  - Region/Zone: `us-central1-a`

  {% include step_image.html %}

  - Machine type: **e2-medium**
  - Boot disk: **Debian 12**
  - Networking
    - Network tags: `lab05-iap-ssh`
  - Networking/Network interfaces:
    - Network: `lab05-vpc`
    - Subnetwork: `lab05-subnet-app`
    - External IPv4 address: **None**
    - Clic en **Done**
  
  {% include step_image.html %}

  - Security
    - Service account: selecciona `sa-lab05-app`
    - Access scopes: **Allow full access to all Cloud APIs** (solo laboratorio)

  {% include step_image.html %}

- {% include step_label.html %} Clic en **Create**.

- {% include step_label.html %} **En Cloud Shell**, confirma que la VM existe y que no tiene IP externa (evidencia).

  > **NOTA:** Si `accessConfigs` está vacío, no tiene IP pública. Es correcto.
  {: .lab-note .info .compact}

  ```bash
  source scripts/env.sh
  gcloud compute instances describe "$VM_NAME" --zone "$ZONE" \
    --format="yaml(name,zone,status,networkInterfaces[0].networkIP,networkInterfaces[0].accessConfigs)" \
    | tee outputs/vm_describe.yaml
  ```
  {% include step_image.html %}

#### Tarea 4.3 (UI) — Firewall mínimo para IAP (SSH)

- {% include step_label.html %} En consola: **VPC network** luego en **Firewall**.

- {% include step_label.html %} Da clic en **Create firewall rule**.

  {% include step_image.html %}

- {% include step_label.html %} Configura los siguientes datos.

  > **NOTA:** IAP TCP forwarding origina conexiones desde ese rango hacia el puerto 22.
  {: .lab-note .info .compact}

  - Name: `lab05-fw-allow-iap-ssh`
  - Network: **lab05-vpc**
  - Direction: **Ingress**

  {% include step_image.html %}

  - Targets: **Specified target tags**
  - Target tags: `lab05-iap-ssh`
  - Source IPv4 ranges: `35.235.240.0/20`
  - Protocols/ports: `tcp:22`

  {% include step_image.html %}

- {% include step_label.html %} Clic en el botón **Create**

- {% include step_label.html %} **En Cloud Shell**, confirma la regla (evidencia).

  ```bash
  gcloud compute firewall-rules describe lab05-fw-allow-iap-ssh \
    --format="yaml(name,network,direction,sourceRanges,allowed,targetTags)" | tee outputs/fw_iap_ssh.yaml
  ```
  {% include step_image.html %}

#### Tarea 4.4 (Cloud Shell → SSH por IAP) — Instalar utilidades y Cloud SQL Auth Proxy

- {% include step_label.html %} Conéctate por IAP y valida egress (Cloud NAT). Si no hay salida, revisa NAT.

  > **NOTA:** Si responde con una IP, tu VM tiene salida (NAT funcionando).
  {: .lab-note .info .compact}

  ```bash
  source scripts/env.sh
  gcloud compute ssh "$VM_NAME" --zone "$ZONE" --tunnel-through-iap --command "curl -sS https://ifconfig.me || true"
  ```
  {% include step_image.html %}

- {% include step_label.html %} Instala utilidades: MySQL client, python venv y curl.

  > **NOTA:** `mysql` probará la conexión; `python3-venv` preparará la app.
  {: .lab-note .info .compact}

  ```bash
  gcloud compute ssh "$VM_NAME" --zone "$ZONE" --tunnel-through-iap --command "sudo apt-get update -y && sudo apt-get install -y default-mysql-client python3-venv curl"
  ```

- {% include step_label.html %} Descarga Cloud SQL Auth Proxy y valida versión.

  > **NOTA:** El proxy abre un puerto local y maneja autenticación/seguridad.
  {: .lab-note .info .compact}

  ```bash
  gcloud compute ssh "$VM_NAME" --zone "$ZONE" --tunnel-through-iap --command "sudo curl -fsSLo /usr/local/bin/cloud-sql-proxy https://storage.googleapis.com/cloud-sql-connectors/cloud-sql-proxy/v2.20.0/cloud-sql-proxy.linux.amd64 && sudo chmod +x /usr/local/bin/cloud-sql-proxy && /usr/local/bin/cloud-sql-proxy --version"
  ```
  {% include step_image.html %}

#### Tarea 4.5 (Cloud Shell → SSH por IAP) — Levantar proxy y probar conexión

- {% include step_label.html %} Lee `connectionName` desde `outputs/` y expórtalo.

  ```bash
  source scripts/env.sh
  export INSTANCE_CONN_NAME="$(cat outputs/sql_connection_name.txt)"
  echo "INSTANCE_CONN_NAME=$INSTANCE_CONN_NAME" | tee outputs/instance_conn_name_export.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Crea un servicio systemd del proxy dentro de la VM y valida estado.

  > **NOTA:** systemd reinicia el proxy si falla; es un patrón común en VMs.
  {: .lab-note .info .compact}

  ```bash
  cat <<EOF | gcloud compute ssh "$VM_NAME" --zone "$ZONE" --tunnel-through-iap --command "sudo bash -s"
  set -euo pipefail

  INSTANCE_CONN_NAME="${INSTANCE_CONN_NAME}"
  if [ -z "\$INSTANCE_CONN_NAME" ]; then
    echo "ERROR: INSTANCE_CONN_NAME está vacío. Ejemplo: proyecto:region:instancia"
    exit 1
  fi

  PROXY_BIN="\$(command -v cloud-sql-proxy || true)"
  [ -z "\$PROXY_BIN" ] && [ -x /usr/local/bin/cloud-sql-proxy ] && PROXY_BIN=/usr/local/bin/cloud-sql-proxy
  [ -z "\$PROXY_BIN" ] && [ -x /usr/bin/cloud-sql-proxy ] && PROXY_BIN=/usr/bin/cloud-sql-proxy
  if [ -z "\$PROXY_BIN" ]; then
    echo "ERROR: No encuentro cloud-sql-proxy en PATH (/usr/local/bin o /usr/bin)."
    exit 1
  fi

  echo "--- PRECHECK: PUERTO 3306 ---"
  if ss -lntp | grep -q ":3306"; then
    echo "ERROR: El puerto 3306 YA está en uso. Salida:"
    ss -lntp | grep ":3306" || true
    echo "Solución: detén el proceso que usa 3306 o cambia el puerto del proxy (ej. 3307)."
    exit 1
  fi

  cat > /etc/systemd/system/cloud-sql-proxy.service <<SERVICE
  [Unit]
  Description=Cloud SQL Auth Proxy (Lab05)
  After=network-online.target
  Wants=network-online.target

  [Service]
  Type=simple
  ExecStart=${PROXY_BIN} --address 127.0.0.1 --port 3306 --private-ip ${INSTANCE_CONN_NAME}
  Restart=always
  RestartSec=3
  SuccessExitStatus=143

  [Install]
  WantedBy=multi-user.target
  SERVICE

  systemctl daemon-reload
  systemctl enable --now cloud-sql-proxy
  systemctl restart cloud-sql-proxy

  echo "--- ESPERANDO LISTEN (hasta 5s) ---"
  ok=0
  for i in 1 2 3 4 5; do
    if ss -lntp | grep -q ":3306"; then ok=1; break; fi
    sleep 1
  done

  echo "--- STATUS ---"
  systemctl status cloud-sql-proxy --no-pager -l | sed -n "1,120p"

  echo "--- JOURNAL (últimos 120) ---"
  journalctl -u cloud-sql-proxy -n 120 --no-pager

  echo "--- LISTEN ---"
  ss -lntp | grep ":3306" || true

  if [ "\$ok" -ne 1 ]; then
    echo "ERROR: NO LISTEN 3306 (proxy no está escuchando)"
    exit 2
  else
    echo "OK: proxy escuchando en 127.0.0.1:3306"
  fi
  EOF
  ```

- {% include step_label.html %} Verifica que esta corriendo el servicio.

  ```bash
  gcloud compute ssh "$VM_NAME" --zone "$ZONE" --tunnel-through-iap --command \
  "sudo bash -lc 'ss -lntp | grep :3306; systemctl status cloud-sql-proxy --no-pager -l | sed -n \"1,40p\"'"
  ```
  {% include step_image.html %}

- {% include step_label.html %} Prueba conexión MySQL contra el puerto local del proxy (se te pedirá password de `appuser`).

  > **NOTA:** Si esto funciona, la ruta VM → Proxy → Cloud SQL ya está lista.
  {: .lab-note .info .compact}

  ```bash
  SQL_PRIVATE_IP="$(gcloud sql instances describe "$SQL_INSTANCE" --format="value(ipAddresses[0].ipAddress)")"
  echo "SQL_PRIVATE_IP=$SQL_PRIVATE_IP" | tee outputs/sql_private_ip.txt
  ```
  ```bash
  gcloud compute ssh "$VM_NAME" --zone "$ZONE" --tunnel-through-iap --command "mysql -h $SQL_PRIVATE_IP -P 3306 -u appuser -p -D appdb -e 'SELECT VERSION() AS mysql_version;'"
  ```
  {% include step_image.html %}

- {% include step_label.html %} Crea tabla e inserta datos ficticios (se te pedirá password). Luego consulta la tabla.

  > **NOTA:** Dataset mínimo para que la app muestre resultados reales.
  {: .lab-note .info .compact}

  ```bash
  gcloud compute ssh "$VM_NAME" --zone "$ZONE" --tunnel-through-iap --command "mysql -h $SQL_PRIVATE_IP -P 3306 -u appuser -p -D appdb -e \"
  CREATE TABLE IF NOT EXISTS visits (
    id INT AUTO_INCREMENT PRIMARY KEY,
    who VARCHAR(64) NOT NULL,
    note VARCHAR(120) NOT NULL,
    ts TIMESTAMP DEFAULT CURRENT_TIMESTAMP
  );
  INSERT INTO visits (who, note) VALUES
    ('lab05', 'primera inserción desde VM'),
    ('lab05', 'datos ficticios para validación');
  SELECT * FROM visits ORDER BY id DESC LIMIT 5;
  \""
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[3] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 5. Desplegar una app web simple (VM) que consulta Cloud SQL (Tiempo estimado: 10 min)

En esta tarea crearás una app mínima con Flask que lee/escribe registros en MySQL y los muestra en una página con estilo. Accederás con **port-forward por SSH/IAP**.

#### Tarea 5.1 (Cloud Shell → SSH por IAP) — Crear app Flask

- {% include step_label.html %} Crea el directorio de la app y un entorno virtual.

  > **NOTA:** Flask sirve el HTML; PyMySQL se conecta al MySQL local (proxy).
  {: .lab-note .info .compact}

  ```bash
  source scripts/env.sh
  gcloud compute ssh "$VM_NAME" --zone "$ZONE" --tunnel-through-iap --command "sudo bash -s" <<'EOF'
  set -euo pipefail
  export DEBIAN_FRONTEND=noninteractive

  # Pre-reqs (venv)
  apt-get update -y
  apt-get install -y python3-venv python3-pip >/dev/null

  # App dir
  mkdir -p /opt/lab05-app
  chown -R root:root /opt/lab05-app
  chmod 755 /opt/lab05-app

  # Virtualenv + deps
  python3 -m venv /opt/lab05-app/.venv
  /opt/lab05-app/.venv/bin/pip install --upgrade pip >/dev/null
  /opt/lab05-app/.venv/bin/pip install flask pymysql >/dev/null

  echo "OK: venv + deps instalados"
  EOF
  ```

- {% include step_label.html %} Crea `app.py` en `/opt/lab05-app/` dentro de la maquina virtual.

  > **NOTA:** La app toca la DB en cada carga y lista resultados: prueba end-to-end.
  {: .lab-note .info .compact}

  {% raw %}
  ```bash
  gcloud compute ssh "$VM_NAME" --zone "$ZONE" --tunnel-through-iap --command "sudo bash -s" <<'EOF'
  set -euo pipefail
  set +H 2>/dev/null || true   # evita problemas con <!doctype

  APPDIR=/opt/lab05-app

  cat > "$APPDIR/app.py" <<'PY'
  import os
  import pymysql
  from flask import Flask

  app = Flask(__name__)

  DB_HOST = os.getenv("DB_HOST", "127.0.0.1")
  DB_PORT = int(os.getenv("DB_PORT", "3306"))
  DB_NAME = os.getenv("DB_NAME", "appdb")
  DB_USER = os.getenv("DB_USER", "appuser")
  DB_PASS = os.getenv("DB_PASS", "")

  def query(sql, args=None, select=False):
      conn = pymysql.connect(
          host=DB_HOST, port=DB_PORT, user=DB_USER, password=DB_PASS,
          database=DB_NAME, cursorclass=pymysql.cursors.DictCursor,
          connect_timeout=5, read_timeout=5, write_timeout=5
      )
      try:
          with conn.cursor() as cur:
              cur.execute(sql, args or ())
              if select:
                  return cur.fetchall()
              conn.commit()
              return []
      finally:
          conn.close()

  @app.get("/")
  def home():
      ok = True
      err = ""
      rows = []
      try:
          query("INSERT INTO visits (who, note) VALUES (%s, %s)", ("lab05-app", "hit desde flask"))
          rows = query("SELECT id, who, note, ts FROM visits ORDER BY id DESC LIMIT 10", select=True)
      except Exception as e:
          ok = False
          err = str(e)

      status = "OK" if ok else "ERROR"
      pill = "#1f7a3a" if ok else "#7a1f1f"
      err_html = f"<p style='background:#7a1f1f;color:#e6eefc;padding:10px;border-radius:12px;'>Error: <code>{err}</code></p>" if not ok else ""
      trs = "".join([f"<tr><td>{r['id']}</td><td>{r['who']}</td><td>{r['note']}</td><td>{r['ts']}</td></tr>" for r in rows])

      return f"""<!doctype html>
  <html>
  <head>
    <meta charset="utf-8">
    <title>Lab05 - VM App + Cloud SQL</title>
  </head>
  <body style="font-family:system-ui,Segoe UI,Roboto,Arial;background:#0b1220;color:#e6eefc;padding:28px;">
    <div style="max-width:980px;margin:0 auto;background:rgba(255,255,255,0.06);border:1px solid rgba(255,255,255,0.12);border-radius:16px;padding:18px;">
      <h1 style="margin:0 0 10px;">Lab05: VM App + Cloud SQL</h1>
      <p style="background:{pill};color:#e6eefc;padding:6px 10px;border-radius:999px;display:inline-block;">Estado DB: <b>{status}</b></p>
      <p>Conexión: <code>{DB_USER}@{DB_HOST}:{DB_PORT}/{DB_NAME}</code></p>
      {err_html}
      <table style="width:100%;border-collapse:collapse;margin-top:12px;">
        <thead>
          <tr>
            <th style="text-align:left;border-bottom:1px solid rgba(255,255,255,0.12);padding:8px;">ID</th>
            <th style="text-align:left;border-bottom:1px solid rgba(255,255,255,0.12);padding:8px;">WHO</th>
            <th style="text-align:left;border-bottom:1px solid rgba(255,255,255,0.12);padding:8px;">NOTE</th>
            <th style="text-align:left;border-bottom:1px solid rgba(255,255,255,0.12);padding:8px;">TS</th>
          </tr>
        </thead>
        <tbody>
          {trs}
        </tbody>
      </table>
      <p style="opacity:.85;margin-top:10px;">Refresca la página para generar nuevos registros.</p>
    </div>
  </body>
  </html>"""
  PY

  cat > "$APPDIR/run.py" <<'PY'
  from app import app
  app.run(host="127.0.0.1", port=8080)
  PY

  # Validación rápida
  /opt/lab05-app/.venv/bin/python -m py_compile "$APPDIR/app.py" "$APPDIR/run.py"
  echo "OK: app.py + run.py creados y compilados"
  EOF
  ```
  {% endraw %}

- {% include step_label.html %} Crear el servicio systemd **lab05-app.service**

  ```bash
  gcloud compute ssh "$VM_NAME" --zone "$ZONE" --tunnel-through-iap --command "sudo bash -s" <<'EOF'
  set -euo pipefail

  cat > /etc/systemd/system/lab05-app.service <<'SERVICE'
  [Unit]
  Description=Lab05 Flask App (VM App + Cloud SQL)
  After=network-online.target cloud-sql-proxy.service
  Wants=network-online.target
  Requires=cloud-sql-proxy.service

  [Service]
  Type=simple
  WorkingDirectory=/opt/lab05-app
  EnvironmentFile=/opt/lab05-app/app.env
  Environment=PYTHONUNBUFFERED=1
  ExecStart=/opt/lab05-app/.venv/bin/python /opt/lab05-app/run.py
  Restart=always
  RestartSec=3

  [Install]
  WantedBy=multi-user.target
  SERVICE

  systemctl daemon-reload
  systemctl enable --now lab05-app || true   # arranca cuando exista app.env
  echo "OK: lab05-app.service creado"
  EOF
  ```

- {% include step_label.html %} Crea `app.env` edita el archivo antes de pegarlo en la terminal **pon tu password real de `appuser`**.

  > **NOTA:** Para laboratorio se acepta; en producción usa Secret Manager.
  {: .lab-note .info .compact}

  ```bash
  source scripts/env.sh
  gcloud compute ssh "$VM_NAME" --zone "$ZONE" --tunnel-through-iap --command "sudo bash -s" <<'EOF'
  set -euo pipefail

  cat > /opt/lab05-app/app.env <<'ENV'
  DB_HOST=127.0.0.1
  DB_PORT=3306
  DB_NAME=appdb
  DB_USER=appuser
  DB_PASS=CAMBIA_AQUI_PASSWORD_APPUSER
  ENV

  chmod 600 /opt/lab05-app/app.env
  systemctl restart lab05-app
  echo "OK: /opt/lab05-app/app.env creado"
  EOF
  ```
  {% include step_image.html %}

#### Tarea 5.2 (Cloud Shell → SSH por IAP) — Servicio systemd para la app

- {% include step_label.html %} Prueba local desde la VM (sin navegador) para confirmar HTML.

  > **NOTA:** Si el HTML aparece, la app corre y responde.
  {: .lab-note .info .compact}

  ```bash
  gcloud compute ssh "$VM_NAME" --zone "$ZONE" --tunnel-through-iap --command "curl -sS http://127.0.0.1:8080 | head"
  ```
  {% include step_image.html %}

#### Tarea 5.3 (Cloud Shell) — Acceso a la app con port-forward por SSH/IAP

- {% include step_label.html %} Abre un túnel SSH con forwarding del puerto 8080 (mantén esta sesión abierta).

  > **NOTA:** No abres firewall para 8080; usas SSH (IAP) para ver la app.
  {: .lab-note .info .compact}

  ```bash
  gcloud compute ssh "$VM_NAME" --zone "$ZONE" --tunnel-through-iap -- -L 8080:127.0.0.1:8080
  ```
  {% include step_image.html %}

- {% include step_label.html %} En otra terminal de Cloud Shell, valida con curl al puerto local.

  > **NOTA:** Si ves `Estado DB: OK`, la conectividad a Cloud SQL está funcionando.
  {: .lab-note .info .compact}

  ```bash
  curl -sS http://127.0.0.1:8080 | grep -E "Lab05|Estado DB|OK|ERROR" | head
  ```
  {% include step_image.html %}

- {% include step_label.html %} (Opcional) En Cloud Shell, usa **Web preview** y elige **Preview on port 8080** para ver la página.

  {% include step_image.html %}

- {% include step_label.html %} El resultado exitoso deberas ver la app con los datos de la base de datos.

  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[4] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 6. Limpieza (Tiempo estimado: 5 min)

Elimina recursos para evitar costos.

#### Tarea 6.1

- {% include step_label.html %} **Cloud Shell** Elimina **Cloud SQL** :

  - Cloud SQL instance `lab05-sql-mysql`

  ```bash
  gcloud sql instances patch lab05-sql-mysql --no-deletion-protection
  gcloud sql instances delete lab05-sql-mysql --quiet
  ```

- {% include step_label.html %} **Cloud Shell** Elimina **Compute instance** :

  - VM `lab05-vm-app`

  ```bash
  gcloud compute instances delete lab05-vm-app --zone "$ZONE" --quiet
  ```

- {% include step_label.html %} **Cloud Shell** Elimina Cloud NAT:

  ```bash
  gcloud compute routers nats delete lab05-nat \
    --router lab05-router \
    --region "$REGION" \
    --quiet
  ```
  ```bash
  gcloud compute routers delete lab05-router --region "$REGION" --quiet
  ```

- {% include step_label.html %} **Cloud Shell** Elimina firewall rule:

  ```bash
  gcloud compute firewall-rules delete lab05-fw-allow-iap-ssh --quiet
  ```

- {% include step_label.html %} **Cloud Shell** Elimina las subnets:

  ```bash
  gcloud compute networks subnets list --network lab05-vpc --format='value(name,region)' \
  | while read -r SUBNET REGION_URL; do
      REGION_NAME="${REGION_URL##*/}"
      echo "Deleting subnet $SUBNET in $REGION_NAME ..."
      gcloud compute networks subnets delete "$SUBNET" --region "$REGION_NAME" --quiet
    done
  ```

- {% include step_label.html %} **Cloud Shell** Elimina VPC `lab05-vpc` (asegúrate de que no queden dependencias).

  ```bash
  gcloud compute addresses delete lab05-psa-range --global --quiet
  ```
  ```bash
  gcloud services vpc-peerings delete \
    --service=servicenetworking.googleapis.com \
    --network=lab05-vpc \
    --quiet
  ```
  ```bash
  gcloud compute networks delete lab05-vpc --quiet
  ```

- {% include step_label.html %} **Cloud Shell** Verifica que ya no existan recursos (si los eliminaste por UI).

  ```bash
  source scripts/env.sh
  gcloud compute instances describe "$VM_NAME" --zone "$ZONE" >/dev/null 2>&1 || echo "VM eliminada"
  gcloud sql instances describe "$SQL_INSTANCE" >/dev/null 2>&1 || echo "Cloud SQL eliminada"
  gcloud compute networks describe "$VPC_NAME" >/dev/null 2>&1 || echo "VPC eliminada"
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[5] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}