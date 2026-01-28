---
layout: lab
title: "Práctica 2: Crear VM en Compute Engine y conectarla por IAP (Identity-Aware Proxy)"
permalink: /lab2/lab2/
images_base: /labs/lab2/img
duration: "45 minutos"
objective:
  - "Crear una VM **privada (sin IP pública)** en **Compute Engine** y conectarte de forma segura mediante **IAP TCP forwarding** (SSH por túnel), validando IAM + firewall requeridos. Finalmente desplegarás un **contenedor web** con una página con estilo y la visualizarás usando **port-forwarding por IAP**."
prerequisites:
  - "Proyecto de Google Cloud con permisos para **Compute Engine** e **IAM** (idealmente rol Owner/Editor en un sandbox)."
  - "Acceso a **Google Cloud Console** y **Cloud Shell (CLI)**."
  - "Conocer zona/región donde crearás la VM (ej. `us-central1-a`)."
introduction: |
  **Identity-Aware Proxy (IAP) TCP forwarding** permite acceder a servicios administrativos como **SSH** en VMs **sin exponerlas a Internet** (sin IP externa). El acceso pasa por autenticación/autorización (IAM) y el tráfico se encapsula de forma segura, reduciendo el riesgo de abrir puertos al mundo. En esta práctica crearás una VM privada, habilitarás el acceso por IAP (IAM + firewall), te conectarás por SSH, y luego ejecutarás un contenedor con una página web “bonita” que abrirás mediante un túnel IAP a un puerto TCP.
slug: lab2
lab_number: 2
final_result: >
  Al finalizar, tendrás una VM privada en Compute Engine accesible por IAP (SSH sin IP pública), con reglas de firewall limitadas al rango de IAP. Además, la VM ejecutará un contenedor web con una página estilizada y podrás verla desde tu navegador mediante un túnel IAP (port-forward) a un puerto local.
notes:
  - "Compute Engine **tiene costo** (VM/discos/red). IAP no sustituye los costos de cómputo: sigues pagando por la VM y recursos asociados."
  - "Si la VM tiene **IP externa**, la conexión SSH puede preferir la IP pública. Para forzar IAP usa `--tunnel-through-iap` o crea la VM **sin IP externa**."
  - "IAP TCP forwarding requiere firewall que permita tráfico desde `35.235.240.0/20` hacia los puertos que vayas a tunelar (ej. 22, 8080)."
  - "Evita reglas tipo `default-allow-ssh` abiertas a `0.0.0.0/0` si quieres que SOLO se conecten por IAP."
  - "En entornos empresariales, es común usar **OS Login**; si tu organización lo exige, ajusta el método de SSH según la política."
references:
  - text: "Usar IAP para TCP forwarding (pasos, firewall, roles IAM, comportamiento con IP externa)"
    url: https://docs.cloud.google.com/iap/docs/using-tcp-forwarding
  - text: "Conectar a VMs Linux usando IAP (guía específica)"
    url: https://docs.cloud.google.com/compute/docs/connect/ssh-using-iap
  - text: "TCP forwarding overview (concepto y cómo funciona)"
    url: https://docs.cloud.google.com/iap/docs/tcp-forwarding-overview
  - text: "Cloud Shell: cómo funciona (5 GB persistentes en $HOME)"
    url: https://docs.cloud.google.com/shell/docs/how-cloud-shell-works
  - text: "Compute Engine pricing"
    url: https://cloud.google.com/compute/all-pricing
  - text: "Free Tier / Always Free (Compute Engine e2-micro, condiciones)"
    url: https://docs.cloud.google.com/free/docs/free-cloud-features
prev: /lab1/lab1/
next: /lab3/lab3/
---

---

## Instrucciones generales

- La práctica es **mayormente por UI**, con comandos puntuales en **Cloud Shell** para confirmar configuración y crear túneles.
- Mantendremos la estructura de carpetas del curso en Cloud Shell:
  - `~/labs-gcp-engineer/lab02/` con subcarpetas `scripts/`, `outputs/`, `notes/`.
- Convención recomendada:
  - **Región/Zona:** `us-central1` / `us-central1-a`.
  - **Nombre VM:** `iap-vm-web-01`
  - **Red:** `default`

---

### Tarea 1. Preparar Cloud Shell + estructura de carpetas (Tiempo estimado: 6 min)

En esta tarea activarás Cloud Shell, confirmarás el proyecto activo y crearás la carpeta de la práctica con scripts/evidencias.

> **IMPORTANTE:** Cloud Shell tiene `$HOME` persistente (5 GB), ideal para tus laboratorios.
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

- {% include step_label.html %} Crea la carpeta del curso:

  ```bash
  cd ~
  mkdir -p labs-gcp-engineer/lab02/{scripts,outputs,notes}
  ```
  ```bash
  cd labs-gcp-engineer/lab02
  ```
  {% include step_image.html %}

- {% include step_label.html %} Define variables base del lab y guárdalas en `scripts/env.sh`:

  > **IMPORTANTE:** Edita el codigo antes de ejecutarlo. Cambia las letras **xx** al final del nombre de la VM para que sea unica.
  {: .lab-note .important .compact}

  ```bash
  cat > scripts/env.sh << 'EOF'
  # Ajusta a tu zona
  export ZONE="us-central1-a"
  export REGION="us-central1"

  # Nombre de VM
  export VM_NAME="iap-vm-web-xx"

  # Red (por simplicidad del lab)
  export VPC_NETWORK="default"
  EOF
  ```
  ```bash
  source scripts/env.sh
  ```

- {% include step_label.html %} Comprueba que las variables quedaron correctas (y guarda evidencia).

  ```bash
  echo "PROJECT=$(gcloud config get-value project)"
  echo "ZONE=$ZONE"
  echo "REGION=$REGION"
  echo "VM_NAME=$VM_NAME"
  echo "VPC_NETWORK=$VPC_NETWORK"
  ```
  {% include step_image.html %}
  ```bash
  gcloud version | head -n 1 | tee outputs/gcloud_version.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Confirma que la estructura de carpetas está completa.

  > **NOTA:**
  - Separar `env.sh` te permite repetir comandos sin errores y mantener consistencia.
  - Guardar evidencias en `outputs/` facilita auditoría del laboratorio.
  {: .lab-note .info .compact}

  ```bash
  ls -la | tee outputs/lab02_tree_root.txt
  ```
  ```bash
  ls -la scripts outputs notes | tee outputs/lab02_tree_subdirs.txt
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[0] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 2. Preparar el proyecto para IAP (APIs + IAM) (Tiempo estimado: 10 min)

En esta tarea habilitarás los APIs necesarios y asignarás permisos IAM para usar túneles IAP hacia VMs.

> **IMPORTANTE:** IAP controla **quién** puede tunelar (IAM), pero también necesitas firewall para permitir el tráfico **desde** IAP al puerto destino (ej. 22, 8080).
{: .lab-note .important .compact}

#### Tarea 2.1 (UI) — Habilitar APIs

- {% include step_label.html %} Da clic en el menu lateral izquierdo y ve a **APIs & Services**  y luego **Library**.

  {% include step_image.html %}

- {% include step_label.html %} Busca la API llamada **`Compute Engine API`**

  {% include step_image.html %}

- {% include step_label.html %} Ahora da clic en el botón **Enable**.

  {% include step_image.html %}

- {% include step_label.html %} En la ventana emergente selecciona tu proyecto **¡CUIDADO!** de no seleccionar otro.

  {% include step_image.html %}

- {% include step_label.html %} Ahora da clic nuevamente en la sección **Library** del menu lateral izquierdo.

  {% include step_image.html %}

- {% include step_label.html %} Repite la busqueda para la API llamada **`Cloud Identity-Aware Proxy API`**

  {% include step_image.html %}

- {% include step_label.html %} Ahora da clic en el botón **Manage**.

  {% include step_image.html %}

- {% include step_label.html %} **(Desde Cloud Shell)** Confirma que las APIs quedaron habilitados y guarda evidencia.

  ```bash
  gcloud services list --enabled | egrep -i "compute|iap" | tee outputs/enabled_apis.txt
  ```
  {% include step_image.html %}

#### Tarea 2.2 (UI) — Asignar IAM (IAP Tunnel User)

- {% include step_label.html %} En el menú lateral izquierdo ve a **IAM & Admin** y luego **IAM**.

  {% include step_image.html %}

- {% include step_label.html %} Identifica tu usuario (tu email) y da clic en el **Lapíz** de edición.

  {% include step_image.html %}

- {% include step_label.html %} Si no aparece el rol **IAP-Secured Tunnel User** agregalo. Si ya aparece avanza al siguiente paso.

  {% include step_image.html %}

- {% include step_label.html %} Tambien agrega el rol **Compute Instance Admin (v1)** de la sección **Compute Engine**. Da clic en el botón **Add another role** para agregar uno nuevo.

  {% include step_image.html %}

- {% include step_label.html %} Deberas tener minimo **2 roles** haz caso omiso de los demas y da clic en el botón **Save**

  > **Nota:**
  - Sin el rol `roles/iap.tunnelResourceAccessor`, el túnel no inicia.
  - Sin permisos sobre la VM (p. ej. `Compute Instance Admin (v1)` en un lab), puedes fallar al inyectar llaves SSH / obtener metadatos.
  {: .lab-note .info .compact}

  {% include step_image.html %}

- {% include step_label.html %} **(Desde Cloud Shell)** Obtén tu identidad activa para comparar con lo asignado en IAM.

  ```bash
  gcloud auth list --filter=status:ACTIVE --format="value(account)" | tee outputs/active_account.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} **(Desde Cloud Shell)** Lista bindings IAM del proyecto filtrando por tu usuario (si tienes permisos para leer IAM).

  > **Nota:** Si no tienes permisos para `get-iam-policy`, valida únicamente en la UI.
  {: .lab-note .info .compact}

  ```bash
  gcloud projects get-iam-policy "$(gcloud config get-value project)" \
    --flatten="bindings[].members" \
    --filter="bindings.members:user:${ACTIVE_ACCOUNT}" \
    --format="table(bindings.role)" \
  | tee outputs/iam_roles_for_active_account.txt || true
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[1] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 3. Crear una VM privada en Compute Engine (Tiempo estimado: 12 min)

Crearás una VM **sin IP externa** (solo IP interna) para que el acceso SSH ocurra vía IAP.

> **IMPORTANTE:** Una VM sin IP externa reduce superficie de ataque; IAP te permite administrarla sin exponer SSH público.
{: .lab-note .important .compact}

#### Tarea 3.1 (UI) — Crear VM

- {% include step_label.html %} Ve al menu lateral izquiero y slecciona  **Compute Engine** luego **VM instances**.

  {% include step_image.html %} 

- {% include step_label.html %}Ahora da clic en el botón **Create instance**

  {% include step_image.html %} 

- {% include step_label.html %} Configura los siguientes datos para crear la maquina virtual.

  > **IMPORTANTE:** El resto de los valores se quedara **por defecto**. Coloca el mismo nombre que definiste en la tarea 1 sino fallaran las siguientes tareas.
  {: .lab-note .important .compact}

  - **Name:** `iap-vm-web-xx` (Mismo nombre que el que definiste en las variables **Tarea 1**)
  - **Region/Zone:** `us-central1-a`

    {% include step_image.html %}

  - **Machine type (recomendado):** `e2-medium` (para fluidez en Docker)

  {% include step_image.html %}

  - **OS and Storage/Image:** Debian 12 (o Ubuntu LTS)

  {% include step_image.html %} 

- {% include step_label.html %} En la sección de **Networking** y **Network interfaces** verifica los siguientes datos. Clic en el botón **Done**

  - **Network:** `default`
  - **Subnetwork:** La que recomiende
  - **External IPv4:** Selecciona **None** (sin IP externa)

  {% include step_image.html %}

- {% include step_label.html %} Finalmente da clic en el botón **Create**

  {% include step_image.html %}

- {% include step_label.html %} Da clic en el nombre de la maquina virtual.

  {% include step_image.html %} 

- {% include step_label.html %} Clic en **Edit**.

  {% include step_image.html %} 

- {% include step_label.html %} Luego en la opción **Network tags** casi llegando a la mitad de la pantalla: 

  {% include step_image.html %}

- {% include step_label.html %} Agrega las siguientes etiquetas.
  
  - `iap-ssh`
  - `iap-web`

  {% include step_image.html %}

- {% include step_label.html %} Clic en **Save**.

  {% include step_image.html %}

#### Tarea 3.2 Confirmar propiedades clave

- {% include step_label.html %} **En la UI**, abre la VM y confirma visualmente:

  - **Status:** Running

  {% include step_image.html %}

  - **External IP:** none
  - **Internal IP:** asignada
  - **Network tags:** `iap-ssh`, `iap-web`

  {% include step_image.html %}

- {% include step_label.html %} **(Desde Cloud Shell)** Consulta la VM por CLI y guarda evidencia (IP externa debe salir vacía).

  ```bash
  source ~/labs-gcp-engineer/lab02/scripts/env.sh
  ```
  ```bash
  gcloud compute instances describe "$VM_NAME" --zone "$ZONE" --format="yaml(name,status,networkInterfaces[0].networkIP,networkInterfaces[0].accessConfigs,labels,tags.items)" | tee outputs/vm_describe.yaml
  ```
  {% include step_image.html %}

- {% include step_label.html %} **(Desde Cloud Shell)** Extrae un resumen rápido de los datos configurados.

  > **Nota:** 
  - El archivo `vm_external_ip.txt` debería estar vacío.
  - Sin IP externa, evitas abrir SSH al internet.
  - Los tags permiten aplicar firewall **solo** a esta VM (en lugar de “toda la red”).
  {: .lab-note .info .compact}

  ```bash
  gcloud compute instances describe "$VM_NAME" --zone "$ZONE" --format="value(status)" | tee outputs/vm_status.txt
  ```
  ```bash
  gcloud compute instances describe "$VM_NAME" --zone "$ZONE" --format="value(networkInterfaces[0].accessConfigs[0].natIP)" | tee outputs/vm_external_ip.txt
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[2] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 4. Configurar firewall para IAP (Tiempo estimado: 7 min)

Crearás reglas de firewall que permitan tráfico **solo desde IAP** (rango `35.235.240.0/20`) hacia los puertos que necesitas: **22** (SSH) y **8080** (web del contenedor).

> **IMPORTANTE:** Si tu proyecto tiene `default-allow-ssh` (0.0.0.0/0), considera deshabilitarla/eliminarla si quieres que SOLO sea posible entrar por IAP.
{: .lab-note .important .compact}

#### Tarea 4.1 (UI) — Firewall para SSH por IAP

- {% include step_label.html %} Ve a **VPC network** y luego **Firewall**

  {% include step_image.html %}

- {% include step_label.html %} Ahora da clic en **Create firewall rule**.

   {% include step_image.html %}
   
- {% include step_label.html %} Configura los siguientes datos:

  - **Name:** `allow-iap-ssh`
  - **Direction:** Ingress

  {% include step_image.html %}
  
  - **Targets:** Specified target tags
  - **Target tags:** `iap-ssh`
  - **Source IPv4 ranges:** `35.235.240.0/20`
  - **Protocols and ports:** `tcp:22`

  {% include step_image.html %}

  - Clic en el botón **Create**

  {% include step_image.html %}

#### Tarea 4.2 (UI) — Firewall para Web (puerto 8080) por IAP

- {% include step_label.html %} Clic en **Create firewall rule**.

  {% include step_image.html %}

- {% include step_label.html %} Crea otra regla con los siguientes datos:

  - **Name:** `allow-iap-web-8080`
  - **Direction:** Ingress

  {% include step_image.html %}

  - **Targets:** Specified target tags
  - **Target tags:** `iap-web`
  - **Source IP ranges:** `35.235.240.0/20`
  - **Protocols and ports:** `tcp:8080`
  
  {% include step_image.html %}

  - Clic en **Create**

  {% include step_image.html %}

#### Tarea 4.3 (Cloud Shell) — Comprobar reglas y “ver” el rango de IAP aplicado

- {% include step_label.html %} **Desde Cloud Shell** lista ambas reglas y guarda evidencia:

  > **NOTA:**
  - IAP actúa como “puerta” de entrada autenticada, pero el firewall aún debe permitir que el tráfico **desde IAP** llegue al puerto destino.
  - Separar reglas (22 vs 8080) te da control fino.
  {: .lab-note .info .compact}

  ```bash
  gcloud compute firewall-rules list --filter="name=('allow-iap-ssh' OR 'allow-iap-web-8080')" --format="table(name,network,direction,targetTags.list():label=tags,sourceRanges.list():label=source,allowed.list():label=ports)" | tee outputs/firewall_iap_rules.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Revisa el contenido del archivo de evidencia:

  ```bash
  sed -n '1,120p' outputs/firewall_iap_rules.txt
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[3] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 5. Conectarte a la VM por IAP (Tiempo estimado: 6 min)

En esta tarea te conectarás por SSH vía IAP y confirmarás que el acceso funciona sin IP pública.

#### Tarea 5.1 (UI) — SSH en navegador (preferido para lab UI)

- {% include step_label.html %} Ve a **Compute Engine** y luego a **VM instances**.

- {% include step_label.html %} En la fila de la VM `iap-vm-web-xx`, clic en **SSH** y luego **Open in browser window**.

  {% include step_image.html %}

- {% include step_label.html %} Autoriza las ventanas emergentes con tu usuario.

- {% include step_label.html %} Dentro de la VM, confirma identidad y red:

  ```bash
  echo "User=$(whoami)"
  ```
  ```bash
  echo "Host=$(hostname)"
  ```
  ```bash
  ip -br a | sed -n '1,10p'
  ```
  {% include step_image.html %}

- {% include step_label.html %} Comprueba la conectividad local a puerto 22 (solo observación) y registra evidencia de sistema:

  ```bash
  uname -a | tee ~/uname.txt
  ```
  ```bash
  cat /etc/os-release | sed -n '1,12p' | tee ~/os-release.txt
  ```
  {% include step_image.html %}

#### Tarea 5.2 SSH por IAP desde CLI (forzando túnel)

- {% include step_label.html %} **Desde Cloud Shell**, abre SSH forzando IAP:

  ```bash
  source ~/labs-gcp-engineer/lab02/scripts/env.sh
  ```
  ```bash
  gcloud compute ssh "$VM_NAME" --zone "$ZONE" --tunnel-through-iap --quiet
  ```
  {% include step_image.html %}

- {% include step_label.html %} **Dentro de la VM (EN EL CLOUD SHELL)**, confirma que la sesión funciona y captura una evidencia rápida:

  > **NOTA:**
  - Si la VM no tiene IP externa, el acceso se vuelve “administrado” por identidad (IAP + IAM) en lugar de “por red pública”.
  - `--tunnel-through-iap` es el “switch” que fuerza IAP incluso si hubiera IP externa.
  {: .lab-note .info .compact}

  ```bash
  date | tee -a ~/iap_ssh_session.txt
  ```
  ```bash
  echo "Connected via IAP: OK" | tee -a ~/iap_ssh_session.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Sal de la VM:

  ```bash
  exit
  ```

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[4] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 6. Ejecutar contenedor web y ver por túnel IAP (Tiempo estimado: 10–12 min)

Crearás una página HTML con CSS, la servirás con un contenedor NGINX en la VM y la abrirás desde tu navegador usando un túnel IAP hacia el puerto **8080**.

> **IMPORTANTE:** No usaremos IP pública. Accederás vía **IAP port-forward**.
{: .lab-note .important .compact}

#### Tarea 6.1 (SSH dentro de la VM) — Instalar Docker y confirmarlo

- {% include step_label.html %} **Dentro de Cloud Shell** crea un Cloud NAT temporal para la descarga de dependencias de la practica, ejecuta todo el codigo sisguiente

  ```bash
  ROUTER_NAME="nat-router"
  NAT_NAME="nat-config"

  NETWORK="$(gcloud compute instances describe "$VM_NAME" --zone "$ZONE" --format='value(networkInterfaces[0].network)' | awk -F/ '{print $NF}')"

  gcloud compute routers create "$ROUTER_NAME" \
    --region="$REGION" \
    --network="$NETWORK" || true

  gcloud compute routers nats create "$NAT_NAME" \
    --router="$ROUTER_NAME" \
    --region="$REGION" \
    --nat-all-subnet-ip-ranges \
    --auto-allocate-nat-external-ips
  ```
  {% include step_image.html %}

- {% include step_label.html %} Conéctate a la VM (UI o CLI) 

- {% include step_label.html %} Ejecuta los siguientes comandos para preparar la aplicación

  ```bash
  sudo apt-get update -y
  ```
  {% include step_image.html %}
  ```bash
  sudo apt-get install -y docker.io
  ```
  {% include step_image.html %}
  ```bash
  sudo systemctl enable --now docker
  ```
  {% include step_image.html %}
  ```bash
  sudo usermod -aG docker "$USER"
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica que el servicio Docker quedó activo:

  ```bash
  sudo systemctl status docker --no-pager | sed -n '1,12p'
  ```
  {% include step_image.html %}

- {% include step_label.html %} Cierra sesión y vuelve a entrar (necesario para aplicar el grupo `docker` al usuario).

  ```bash
  exit
  ```

- {% include step_label.html %} Reconecta a la VM y confirma que ya puedes ejecutar Docker sin `sudo`:

  ```bash
  docker version
  docker ps
  ```
  {% include step_image.html %}

#### Tarea 6.2 (SSH dentro de la VM) — Crear la web “estilizada”

- {% include step_label.html %} Crea la carpeta **web**.

  ```bash
  mkdir -p ~/web
  cd ~/web
  ```

- {% include step_label.html %} Ahora copia y pega el siguiente codigo que crea el archivo index.html.

  ```bash
  cat > index.html <<'EOF'
  <!doctype html>
  <html lang="es">
  <head>
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width,initial-scale=1" />
    <title>IAP Web Demo</title>
    <link rel="stylesheet" href="styles.css" />
  </head>
  <body>
    <main class="wrap">
      <section class="card">
        <header class="hdr">
          <div class="dot"></div>
          <div>
            <h1>Compute Engine + IAP</h1>
            <p class="sub">VM privada • SSH por túnel • Web por port-forward</p>
          </div>
        </header>

        <div class="grid">
          <div class="pill">
            <span class="k">VM</span>
            <span class="v" id="vm">—</span>
          </div>
          <div class="pill">
            <span class="k">Zona</span>
            <span class="v" id="zone">—</span>
          </div>
          <div class="pill">
            <span class="k">Puerto</span>
            <span class="v">8080</span>
          </div>
          <div class="pill">
            <span class="k">Acceso</span>
            <span class="v">IAP TCP forwarding</span>
          </div>
        </div>

        <hr class="sep" />

        <p class="txt">
          Si ves esta página desde tu navegador, significa que el túnel IAP está funcionando
          y que tu VM no requirió IP pública.
        </p>

        <div class="cta">
          <a class="btn" href="#" onclick="location.reload()">Refrescar</a>
          <span class="hint">Tip: valida con <code>curl http://localhost:8080</code> en la VM</span>
        </div>
      </section>

      <footer class="ft">
        <span>© Lab02 • GCP engineer</span>
        <span class="muted" id="ts">—</span>
      </footer>
    </main>

    <script>
      document.getElementById('vm').textContent = (location.hostname || 'vm');
      document.getElementById('zone').textContent = 'ZONE_PLACEHOLDER';
      document.getElementById('ts').textContent = new Date().toLocaleString();
    </script>
  </body>
  </html>
  EOF

  cat > styles.css <<'EOF'
  :root{
    --bg1:#0b1020;
    --bg2:#111a33;
    --card:#0f172a;
    --txt:#e5e7eb;
    --muted:#a1a1aa;
    --accent:#22c55e;
    --border:rgba(255,255,255,.10);
  }
  *{box-sizing:border-box}
  body{
    margin:0;
    font-family: ui-sans-serif, system-ui, -apple-system, Segoe UI, Roboto, Arial, "Apple Color Emoji","Segoe UI Emoji";
    color:var(--txt);
    min-height:100vh;
    background:
      radial-gradient(900px 600px at 10% 10%, rgba(34,197,94,.18), transparent 60%),
      radial-gradient(900px 600px at 90% 20%, rgba(59,130,246,.18), transparent 55%),
      linear-gradient(160deg, var(--bg1), var(--bg2));
  }
  .wrap{
    max-width:920px;
    margin:0 auto;
    padding:40px 18px;
    display:flex;
    flex-direction:column;
    gap:18px;
  }
  .card{
    border:1px solid var(--border);
    background:rgba(15,23,42,.85);
    backdrop-filter: blur(10px);
    border-radius:18px;
    padding:22px;
    box-shadow: 0 18px 60px rgba(0,0,0,.35);
  }
  .hdr{display:flex;align-items:center;gap:12px}
  .dot{
    width:12px;height:12px;border-radius:999px;
    background:var(--accent);
    box-shadow:0 0 0 6px rgba(34,197,94,.12);
  }
  h1{margin:0;font-size:22px;letter-spacing:.2px}
  .sub{margin:4px 0 0;color:var(--muted);font-size:13px}
  .grid{
    margin-top:16px;
    display:grid;
    grid-template-columns: repeat(2, minmax(0, 1fr));
    gap:10px;
  }
  .pill{
    border:1px solid var(--border);
    border-radius:14px;
    padding:12px 12px;
    display:flex;
    justify-content:space-between;
    align-items:center;
    background:rgba(255,255,255,.04);
  }
  .k{color:var(--muted);font-size:12px}
  .v{font-weight:600;font-size:13px}
  .sep{border:0;border-top:1px solid var(--border);margin:16px 0}
  .txt{margin:0;color:#d1d5db;line-height:1.5}
  .cta{margin-top:14px;display:flex;gap:12px;align-items:center;flex-wrap:wrap}
  .btn{
    display:inline-block;
    background:var(--accent);
    color:#04110a;
    text-decoration:none;
    padding:10px 14px;
    border-radius:12px;
    font-weight:700;
  }
  .hint{color:var(--muted);font-size:12px}
  code{background:rgba(0,0,0,.25);padding:2px 6px;border-radius:8px}
  .ft{
    display:flex;
    justify-content:space-between;
    color:var(--muted);
    font-size:12px;
    padding:0 4px;
  }
  .muted{opacity:.9}
  @media (max-width:640px){
    .grid{grid-template-columns:1fr}
  }
  EOF
  ```

- {% include step_label.html %} Sustituye `ZONE_PLACEHOLDER` por tu zona (`us-central1-a`)

  ```bash
  sed -i "s/ZONE_PLACEHOLDER/${ZONE}/g" index.html
  ```

- {% include step_label.html %} Confirma que el archivo cambió:

  ```bash
  sed -i "s/ZONE_PLACEHOLDER/us-central1-a/g" index.html
  grep -n "ZONE_PLACEHOLDER" -n index.html || echo "OK: placeholder reemplazado"
  ```
  {% include step_image.html %}

- {% include step_label.html %} Confirma que los archivos existen y tienen contenido:

  ```bash
  ls -la ~/web
  ```
  ```bash
  sed -n '1,25p' ~/web/index.html
  sed -n '1,25p' ~/web/styles.css
  ```

#### Tarea 6.3 (SSH dentro de la VM) — Levantar el contenedor NGINX y probar localmente

- {% include step_label.html %} Ejecuta NGINX sirviendo `~/web` en el puerto 8080:

  ```bash
  cd ~/web
  docker rm -f iap-web >/dev/null 2>&1 || true
  ```
  ```bash
  docker run -d --name iap-web -p 8080:80 -v "$PWD":/usr/share/nginx/html:ro nginx:alpine
  ```
  {% include step_image.html %}

- {% include step_label.html %} Comprueba que el contenedor está “Up” y que el puerto está publicado:

  {% raw %}
  ```bash
  docker ps --format "table {{.Names}} {{.Status}}	{{.Ports}}"
  ```
  {% endraw %}
  {% include step_image.html %}

- {% include step_label.html %} Prueba la página desde la VM (debe devolver HTML):

  ```bash
  curl -sS http://localhost:8080 | head -n 30
  ```
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

#### Tarea 6.4 Crear un túnel IAP al puerto 8080 y abrir en navegador

- {% include step_label.html %} **En Cloud Shell** (**NO** dentro de la VM), abre el túnel IAP al puerto 8080:

  > **Nota:** Este comando se queda corriendo para mantener el túnel. Déjalo abierto.
  {: .lab-note .info .compact}

  ```bash
  source ~/labs-gcp-engineer/lab02/scripts/env.sh
  ```
  ```bash
  gcloud compute start-iap-tunnel "$VM_NAME" 8080 --zone "$ZONE" --local-host-port=localhost:8080 --quiet
  ```
  {% include step_image.html %}

- {% include step_label.html %} En Cloud Shell, usa **Web Preview** y luego **Preview on port 8080**.

  {% include step_image.html %}

- {% include step_label.html %} Debes ver la página web de ejemplo con estilos

  > **NOTA:**
  - El contenedor vive dentro de una VM privada; el mundo no puede alcanzarlo porque no hay IP pública.
  - IAP crea un “pasillo” autenticado hacia un puerto TCP específico; es como “conectar un cable virtual” desde tu máquina a la VM sin exponer el servicio.
  {: .lab-note .info .compact}

  {% include step_image.html %}

- {% include step_label.html %} En una segunda pestaña de Cloud Shell (o nueva ventana), confirma que el puerto local responde:

  ```bash
  curl -sS http://localhost:8080 | head -n 15
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[5] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 7. Limpieza (Tiempo estimado: 4–6 min)

Elimina recursos para evitar costos innecesarios.

#### Tarea 7.1 (UI) — Borrar VM

- {% include step_label.html %} Ve a **Compute Engine** y luego **VM instances**.

- {% include step_label.html %} Selecciona `iap-vm-web-xx` da clic en la opción **Delete**.

  > **NOTA:** Confirma la ventana emergente para borrar.
  {: .lab-note .info .compact}

  {% include step_image.html %}

- {% include step_label.html %} **Desde Cloud Shell** Confirma que ya no existe:

  ```bash
  source ~/labs-gcp-engineer/lab02/scripts/env.sh
  ```
  ```bash
  gcloud compute instances describe "$VM_NAME" --zone "$ZONE" >/dev/null 2>&1 || echo "OK: VM eliminada"
  ```
  {% include step_image.html %}

#### Tarea 7.2 Borrar reglas de firewall

- {% include step_label.html %} En UI: clic en **VPC network** y luego **Firewall** elimina:

  - `allow-iap-ssh`
  - `allow-iap-web-8080`

  {% include step_image.html %}

- {% include step_label.html %} Confirma que ya no aparecen:

  ```bash
  gcloud compute firewall-rules list --filter="name~'allow-iap-(ssh|web-8080)'" --format="table(name)" || echo "OK: Reglas eliminadas"
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[6] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}