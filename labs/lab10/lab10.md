---
layout: lab
title: "Práctica 10: Crear snapshot de VM y restaurarla en otra zona"
permalink: /lab10/lab10/
images_base: /labs/lab10/img
duration: "40 minutos"
objective:
  - "Crear una VM en **Compute Engine**, generar un **snapshot** de su disco de arranque y **restaurar** ese snapshot como un nuevo disco en **otra zona**, levantando una segunda VM y validando que los datos se preservan."
prerequisites:
  - "Proyecto de Google Cloud con **facturación habilitada**."
  - "Permisos para: **Compute Engine Admin** (o equivalentes para crear VM/discos/snapshots) y ver **Cloud Shell**."
  - "Acceso a **Google Cloud Console** y a **Cloud Shell** (gcloud CLI)."
introduction: |
  Los **snapshots** de Compute Engine son copias incrementales a nivel de bloque de un disco persistente, útiles para respaldo, clonación y migración. En esta práctica crearás una VM base (zona A), escribirás una “marca” en el disco, crearás un snapshot del disco de arranque, restaurarás el snapshot como un disco en otra zona (zona B) y crearás una nueva VM usando ese disco como boot disk.  

  El enfoque es efectivo y típico en entornos reales para: respaldos rápidos, replicación entre zonas, clonación para pruebas y recuperación ante fallas.
slug: lab10
lab_number: 10
final_result: >
  Al finalizar, tendrás 2 VMs: una original en la zona A y otra restaurada en la zona B creada a partir de un snapshot. Validarás que el archivo “marca” y la página web de prueba existen en la VM restaurada, y tendrás comandos/acciones de verificación y limpieza opcional.
notes:
  - "Buenas prácticas: para consistencia, **detén la VM** (o al menos sincroniza/escribe a disco) antes de capturar el snapshot del disco de arranque."
  - "Snapshots son **incrementales** (solo guardan bloques cambiados). El primer snapshot suele tardar más; los siguientes son más rápidos."
  - "**Costos:** Compute Engine (VM/CPU/RAM), discos persistentes, snapshots (almacenamiento) y tráfico de red. Elimina recursos al terminar para evitar cargos."
references:
  - text: "Compute Engine: Snapshots (conceptos y uso)"
    url: https://cloud.google.com/compute/docs/disks/snapshots
  - text: "Crear y administrar snapshots"
    url: https://cloud.google.com/compute/docs/disks/create-snapshots
  - text: "Restaurar un disco desde un snapshot"
    url: https://cloud.google.com/compute/docs/disks/restore-snapshot
  - text: "Crear VM en Compute Engine (Console)"
    url: https://cloud.google.com/compute/docs/instances/create-start-instance
  - text: "Pricing: Compute Engine"
    url: https://cloud.google.com/compute/all-pricing
  - text: "Pricing: Snapshots"
    url: https://cloud.google.com/compute/disks-image-pricing#snapshots
prev: /lab9/lab9/
next: /lab11/lab11/
---

---

### Tarea 1. Preparar el entorno y variables del laboratorio

> **Tiempo estimado:** 5 minutos
{: .lab-note .info .compact}

En esta tarea abrirás Cloud Shell, verificarás el proyecto activo y prepararás una carpeta independiente para el laboratorio, con variables consistentes (nombres y zonas) que usarás durante toda la práctica.

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

- {% include step_label.html %} Crea la carpeta del laboratorio 10 y subcarpetas estándar.

  > **NOTA:** Mantener cada práctica en su carpeta evita mezclar evidencias y comandos.
  {: .lab-note .important .compact}

  ```bash
  cd ~
  mkdir -p labs-gcp-engineer/lab10/{scripts,outputs}
  ```
  ```bash
  cd labs-gcp-engineer/lab10
  ```
  {% include step_image.html %}

- {% include step_label.html %} Crea el archivo `scripts/env.sh` con variables (zonas A y B).

  > **Nota:** Usa dos zonas de la misma región para que sea un caso real de “mover” una VM entre zonas (ej. `us-central1-a` y `us-central1-b`).
  {: .lab-note .info .compact}

  > **NOTA:** Estandarizar nombres reduce errores y facilita evidencias reproducibles.
  {: .lab-note .info .compact}

  ```bash
  cat > scripts/env.sh <<'EOF'
  # Región y Zonas
  export REGION="us-central1"
  export ZONE_A="us-central1-a"
  export ZONE_B="us-central1-b"

  # Nombres (alineados a lab10)
  export VM_A="lab10-vm-a"
  export VM_B="lab10-vm-b"
  export SNAPSHOT_NAME="lab10-snap-boot"
  export DISK_RESTORED="lab10-disk-boot-restored"
  EOF
  ```
  ```bash
  source scripts/env.sh
  echo "REGION=$REGION ZONE_A=$ZONE_A ZONE_B=$ZONE_B"
  echo "VM_A=$VM_A VM_B=$VM_B SNAPSHOT_NAME=$SNAPSHOT_NAME DISK_RESTORED=$DISK_RESTORED"
  ```
  {% include step_image.html %}

- {% include step_label.html %} Valida que tu carpeta y scripts existen (evidencia).

  > **NOTA:** Aseguras que tu trabajo está organizado y listo para continuar.
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

### Tarea 2. Habilitar Compute Engine y crear VM base en Zona A

> **Tiempo estimado:** 12 minutos
{: .lab-note .info .compact}

En esta tarea habilitarás Compute Engine (si aplica) y crearás una VM desde cero en la zona A. Luego la verificarás por UI y por CLI.

#### Tarea 2.1

- {% include step_label.html %} En la consola, abre **Compute Engine** luego **VM instances**.

  > **NOTA:** La primera vez, puede solicitar habilitar la API (Compute Engine API).
  {: .lab-note .info .compact}
  
  {% include step_image.html %}

- {% include step_label.html %} Crea la VM dando clic en **Create instance** 

  {% include step_image.html %}

- {% include step_label.html %} Configura los siguientes datos:

  > **NOTA:** Usamos una máquina pequeña para laboratorio y permitimos HTTP para validar rápido con un navegador.
  {: .lab-note .info .compact}

  - Name: `lab10-vm-a`
  - Region: **us-central1**
  - Zone: **us-central1-a**

  {% include step_image.html %}

  - Machine type: **e2-medium**
  
  {% include step_image.html %}

  - Boot disk: **Debian GNU/Linux 12 (bookworm)**
  
  {% include step_image.html %}

  - Networking/Firewall: marca **Allow HTTP traffic** (para la página de prueba)

  {% include step_image.html %}

- {% include step_label.html %} Clic en **Create** y espera a que el estado sea **Running**.

  > **NOTA:** Cuando la VM está “Running” puedes conectar por SSH y preparar datos para el snapshot.
  {: .lab-note .info .compact}

  {% include step_image.html %}

- {% include step_label.html %} Verifica por CLI que la VM existe y está en la zona A.

  > **NOTA:** CLI te da evidencia objetiva (zona, estado, disco asociado).
  {: .lab-note .info .compact}

  ```bash
  source scripts/env.sh
  gcloud compute instances describe "$VM_A" --zone "$ZONE_A" --format="yaml(name,zone,status,networkInterfaces[0].networkIP,disks[0].deviceName)"
  ```
  {% include step_image.html %}

- {% include step_label.html %} Captura el nombre del disco de arranque de la VM (lo usarás para el snapshot).

  > **NOTA:** Snapshot se crea sobre un disco; aquí tomamos el boot disk real asociado a la VM.
  {: .lab-note .info .compact}

  ```bash
  source scripts/env.sh
  export BOOT_DISK_A="$(gcloud compute instances describe "$VM_A" --zone "$ZONE_A" --format='value(disks[0].source.basename())')"
  echo "BOOT_DISK_A=$BOOT_DISK_A" | tee outputs/boot_disk_a.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Ahora crea la regla de **SSH** para que te puedas conectar a la maquina virtual

  ```bash
  source scripts/env.sh
  gcloud compute firewall-rules create default-allow-ssh \
    --network=default \
    --direction=INGRESS \
    --priority=1000 \
    --action=ALLOW \
    --rules=tcp:22 \
    --source-ranges=0.0.0.0/0 \
    --target-tags=ssh-server
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica que se haya creado correctamente la regla.

  ```bash
  source scripts/env.sh
  gcloud compute firewall-rules list \
    --filter='network:default AND name=default-allow-ssh' \
    --format="table(name,direction,priority,allowed,sourceRanges,targetTags,disabled)"
  ```
  {% include step_image.html %}

- {% include step_label.html %} Ahora agrega la etiqueta a la maquina virtual para que se pueda establecer la conexion.

  ```bash
  source scripts/env.sh
  gcloud compute instances add-tags "$VM_A" \
    --zone="$ZONE_A" \
    --tags=ssh-server
  ```
  {% include step_image.html %}

- {% include step_label.html %} Ahora verifica que realmente la maquina tenga la etiqueta.

  ```bash
  source scripts/env.sh
  gcloud compute instances describe "$VM_A" \
    --zone="$ZONE_A" \
    --format="yaml(name,tags.items)"
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[1] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 3. Preparar datos en el disco de la VM (marca + página web)

> **Tiempo estimado:** 8 minutos
{: .lab-note .info .compact}

En esta tarea crearás un archivo “marca” en el disco de la VM y configurarás una página web simple con estilo. Esto permite comprobar fácilmente que el snapshot y la restauración preservan el estado del disco.
18
#### Tarea 3.1
- {% include step_label.html %} Conéctate por **SSH** a la maquina virtual desde la UI. Da clic en **Compute Engine** luego en **VM instances** clic en el botón **SSH** de `lab10-vm-a`.

  > **NOTA:** SSH desde consola evita configuraciones locales de llaves en un laboratorio.
  {: .lab-note .important .compact}

  {% include step_image.html %}

- {% include step_label.html %} Se abrira la ventana emergente de la maquina virtual da clic en el botón **Authorize**

  {% include step_image.html %}

- {% include step_label.html %} Dentro de la VM, crea el archivo “marca” y fuerza la escritura al disco.

  > **NOTA:** `sync` ayuda a reducir el riesgo de capturar cambios aún no escritos (especialmente antes del snapshot).
  {: .lab-note .warning .compact}

  ```bash
  sudo mkdir -p /var/lab10
  echo "LAB10_MARKER: $(date -u +"%Y-%m-%dT%H:%M:%SZ") zone=${HOSTNAME}" | sudo tee /var/lab10/marker.txt
  ```
  {% include step_image.html %}
  ```bash
  sync
  ```
  ```bash
  sudo cat /var/lab10/marker.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Instala un servidor web ligero (nginx).

  ```bash
  sudo apt-get update
  sudo apt-get install -y nginx
  ```

- {% include step_label.html %} Crea una página con estilo demostrativo.

  > **NOTA:** La página es una “prueba visual” rápida, útil para mostrar a equipos no técnicos el resultado del restore.
  {: .lab-note .info .compact}

  ```bash
  cat <<'HTML' | sudo tee /var/www/html/index.html
  <!doctype html>
  <html lang="es">
  <head>
    <meta charset="utf-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1" />
    <title>LAB10 - VM Snapshot Restore</title>
    <style>
      body{font-family:system-ui,-apple-system,Segoe UI,Roboto,Arial,sans-serif;margin:0;background:#0b1220;color:#e5e7eb}
      header{padding:28px 18px;background:linear-gradient(90deg,#111827,#1f2937);border-bottom:1px solid #334155}
      h1{margin:0;font-size:22px}
      .wrap{max-width:980px;margin:0 auto;padding:18px}
      .card{background:#0f172a;border:1px solid #243244;border-radius:16px;padding:16px;margin-top:14px}
      code{background:#111827;padding:2px 6px;border-radius:8px}
      .ok{display:inline-block;padding:4px 10px;border-radius:999px;background:#064e3b;border:1px solid #10b981;color:#d1fae5;font-weight:600}
      .grid{display:grid;grid-template-columns:1fr 1fr;gap:12px}
      @media(max-width:720px){.grid{grid-template-columns:1fr}}
      footer{opacity:.8;margin-top:18px;font-size:12px}
    </style>
  </head>
  <body>
    <header>
      <div class="wrap">
        <h1>Práctica 10 — Snapshot de VM y Restauración en otra zona</h1>
      </div>
    </header>
    <div class="wrap">
      <div class="card">
        <div class="ok">OK</div>
        <p>Esta página vive en <code>/var/www/html/index.html</code> y debe preservarse tras restaurar el snapshot.</p>
      </div>
      <div class="card grid">
        <div>
          <h3>Marker</h3>
          <p>Archivo: <code>/var/lab10/marker.txt</code></p>
        </div>
        <div>
          <h3>Idea</h3>
          <p>Si la VM restaurada muestra la misma página y marker, el disco fue restaurado correctamente.</p>
        </div>
      </div>
      <footer>
        LAB10 — Compute Engine snapshots (zona A → zona B)
      </footer>
    </div>
  </body>
  </html>
  HTML
  ```

- {% include step_label.html %} Habilita el servicio **NGINX** y verifica que quedo activado.

  ```bash
  sudo systemctl enable --now nginx
  ```
  ```bash
  sudo systemctl status nginx --no-pager | sed -n '1,20p'
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica desde la VM que nginx responde localmente.

  > **NOTA:** Confirma que el servidor y contenido están correctos antes de snapshot.
  {: .lab-note .important .compact}

  ```bash
  curl -sS http://localhost | head -n 20
  ```
  {% include step_image.html %}

- {% include step_label.html %} **En Cloud Shell**, obtén la IP externa de la VM y guarda evidencia.

  > **NOTA:** Si tienes IP externa y permitiste HTTP, podrás abrir el sitio en el navegador.
  {: .lab-note .info .compact}

  ```bash
  source scripts/env.sh
  gcloud compute instances describe "$VM_A" --zone "$ZONE_A" --format="value(networkInterfaces[0].accessConfigs[0].natIP)" | tee outputs/vm_a_external_ip.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Abre el navegador con `http://<IP_EXTERNA>` y confirma que carga la página **LAB10**.

  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[2] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 4. Detener la VM y crear snapshot del disco de arranque

> **Tiempo estimado:** 7 minutos
{: .lab-note .info .compact}

En esta tarea detendrás la VM (para mejorar consistencia) y crearás el snapshot del boot disk. Luego verificarás que el snapshot existe por CLI.

> **NOTA:** Evidencia visual adicional; en entornos restringidos puede no haber IP externa.
{: .lab-note .warning .compact}

#### Tarea 4.1

- {% include step_label.html %} En UI: ve a **Compute Engine** luego **VM instances** selecciona `lab10-vm-a` y da clic en **STOP** y confirma la ventana emergente.

  > **NOTA:** Detener la VM reduce el riesgo de inconsistencias de filesystem al capturar el snapshot del disco de arranque.
  {: .lab-note .warning .compact}

  {% include step_image.html %}

- {% include step_label.html %} **En Cloud Shell**, valida que el estado de la VM cambió a `TERMINATED`.

  > **NOTA:** Confirmación objetiva antes de capturar snapshot.
  {: .lab-note .important .compact}

  ```bash
  source scripts/env.sh
  gcloud compute instances describe "$VM_A" --zone "$ZONE_A" --format="value(status)"
  ```
  {% include step_image.html %}

- {% include step_label.html %} En UI: ve a **Compute Engine** luego **Disks** y abre el disco `lab10-vm-a`.

  {% include step_image.html %}

- {% include step_label.html %} Ahora da clic en el botón **Create snapshot**

  {% include step_image.html %}

- {% include step_label.html %} Configura los siguientes datos:

  > **NOTA:** Snapshot se crea sobre el disco (no sobre la VM). Aquí usamos el boot disk de la VM A.
  {: .lab-note .info .compact}

  - Name: `lab10-snap-boot`
  - Location: **Regional**
  - Select location: **us-central1 (Iowa)**

  {% include step_image.html %}
  {% include step_image.html %}

- {% include step_label.html %} Clic en el botón **Create**

- {% include step_label.html %} **En Cloud Shell,** valida que el snapshot existe y guarda evidencia.

  > **NOTA:** Confirmas que el snapshot está READY y tienes evidencia del origen. Es normal que tarde un par de minutos.
  {: .lab-note .info .compact}

  ```bash
  source scripts/env.sh
  gcloud compute snapshots describe "$SNAPSHOT_NAME" --format="yaml(name,status,creationTimestamp,storageBytes,sourceDisk.basename())" | tee outputs/snapshot_describe.yaml
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[3] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 5. Restaurar snapshot como disco en Zona B y crear VM restaurada

> **Tiempo estimado:** 9 minutos
{: .lab-note .info .compact}

En esta tarea crearás un nuevo disco en zona B usando el snapshot y luego crearás una VM en zona B usando ese disco como boot disk. Finalmente validarás que el marker y la página web siguen presentes.

#### Tarea 5.1

- {% include step_label.html %} En UI: ve a **Compute Engine** luego **Snapshots** y da clic en el nombre `lab10-snap-boot`.

  {% include step_image.html %}

- {% include step_label.html %} Ahora da clic en el botón **Create disk**

  {% include step_image.html %}

- {% include step_label.html %} Configura los siguientes datos:

  > **NOTA:** Un snapshot es global; el disco restaurado sí vive en una zona específica.
  {: .lab-note .info .compact}

  - Name: `lab10-disk-boot-restored`
  - Region/Zone: `us-central1-b` (zona B)

  {% include step_image.html %}

- {% include step_label.html %} Clic en el botón **Create**

- {% include step_label.html %} **En Cloud Shell**, verifica que el disco restaurado existe en la zona B y guarda evidencia.

  > **NOTA:** Confirmas que el disco proviene del snapshot correcto.
  {: .lab-note .info .compact}

  ```bash
  source scripts/env.sh
  gcloud compute disks describe "$DISK_RESTORED" --zone "$ZONE_B" --format="yaml(name,zone,status,sizeGb,type,sourceSnapshot.basename())" | tee outputs/restored_disk_describe.yaml
  ```
  {% include step_image.html %}

- {% include step_label.html %} En UI: ve a **Compute Engine** luego **VM instances** y da clic en **Create instance**.

  {% include step_image.html %}

- {% include step_label.html %} Configura los siguientes datos para la maquina virtual B:

  > **NOTA:** Creamos una VM “clonada” arrancando desde el disco restaurado.
  {: .lab-note .info .compact}

  Configura:
  - Name: `lab10-vm-b`
  - Region: **us-central1 (Iowa)**
  - Zone: **us-central1-b**

  {% include step_image.html %}

  - Machine type: **e2-medium**

  {% include step_image.html %}

  - Operating system and storage: **Change**

  {% include step_image.html %}

  - Boot disk: selecciona **Existing disk** y elige **lab10-disk-boot-restored**
  - Clic **Select**

  {% include step_image.html %}

  - Networking/Firewall: marca **Allow HTTP traffic**
  - Network tags: `ssh-server`

  {% include step_image.html %}

- {% include step_label.html %} Clic en el botón **Create**.

- {% include step_label.html %} **En Cloud Shell**, valida que la VM B está en zona B y RUNNING.

  > **NOTA:** Verificas zona y disco de arranque (debe ser el restaurado).
  {: .lab-note .important .compact}

  ```bash
  source scripts/env.sh
  gcloud compute instances describe "$VM_B" --zone "$ZONE_B" --format="yaml(name,zone,status,disks[0].source.basename())" | tee outputs/vm_b_describe.yaml
  ```
  {% include step_image.html %}

- {% include step_label.html %} Conéctate por **SSH** a la VM B desde la UI.

  {% include step_image.html %}

- {% include step_label.html %} De igual maneta da clic en el botón **Authorize**.

- {% include step_label.html %} Dentro de la VM B ejecuta el siguiente comando:

  > **NOTA:** Si el archivo existe, el estado del disco se preservó tras snapshot+restore.
  {: .lab-note .info .compact}

  ```bash
  sudo cat /var/lab10/marker.txt
  ls -la /var/lab10
  ```
  {% include step_image.html %}

- {% include step_label.html %} **En la VM B**, valida que nginx existe y responde (la página debe ser la misma).

  > **NOTA:** La app (página) demuestra que el restore preservó configuración y archivos del sistema.
  {: .lab-note .info .compact}

  ```bash
  sudo systemctl status nginx --no-pager | sed -n '1,20p'
  ```
  {% include step_image.html %}
  ```bash
  curl -sS http://localhost | head -n 20
  ```
  {% include step_image.html %}

- {% include step_label.html %} **En Cloud Shell**, obten la IP externa de VM B y abre la página en el navegador.

  ```bash
  source scripts/env.sh
  gcloud compute instances describe "$VM_B" --zone "$ZONE_B" --format="value(networkInterfaces[0].accessConfigs[0].natIP)" | tee outputs/vm_b_external_ip.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Abre la página en el navegador.

  > **NOTA:** Evidencia visual adicional si tu red/política permite IP externa.
  {: .lab-note .info .compact}

  {% include step_image.html %}


{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[4] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 6. Limpieza

> **Tiempo estimado:** 4 minutos
{: .lab-note .info .compact}

En esta tarea eliminarás recursos para evitar costos: VMs, disco restaurado y snapshot. Harás verificación final por CLI.

#### Tarea 6.1

- {% include step_label.html %} En UI: elimina las VMs (recomendado para evitar errores):

  > **NOTA:** Eliminar VMs detiene el cómputo (principal fuente de costo).
  {: .lab-note .warning .compact}

  - **Compute Engine** luego **VM instances** selecciona `lab10-vm-b` → **Delete**
  
  - **Compute Engine** luego **VM instances** selecciona `lab10-vm-a` → **Delete**

  {% include step_image.html %}

- {% include step_label.html %} **En Cloud Shell**, verifica que ya no existen (describe debe fallar).

  > **NOTA:** Evidencia objetiva de limpieza.
  {: .lab-note .info .compact}

  ```bash
  source scripts/env.sh
  gcloud compute instances describe "$VM_B" --zone "$ZONE_B" >/dev/null 2>&1 || echo "OK: VM_B eliminada" | tee outputs/cleanup_vm_b.txt
  ```
  ```bash
  gcloud compute instances describe "$VM_A" --zone "$ZONE_A" >/dev/null 2>&1 || echo "OK: VM_A eliminada" | tee outputs/cleanup_vm_a.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Elimina el disco restaurado (si quedó).

  > **Nota:** Si lo creaste como boot disk con auto-delete, puede eliminarse junto con la VM. Si no, elimínalo explícitamente.
  {: .lab-note .info .compact}

  > **NOTA:** Discos pueden seguir cobrando aunque borres la VM si no tienen auto-delete.
  {: .lab-note .warning .compact}

  ```bash
  source scripts/env.sh
  gcloud compute disks delete "$DISK_RESTORED" --zone "$ZONE_B" --quiet || true
  ```
  ```bash
  gcloud compute disks describe "$DISK_RESTORED" --zone "$ZONE_B" >/dev/null 2>&1 || echo "OK: disco restaurado eliminado" | tee outputs/cleanup_disk.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Elimina el snapshot (para evitar costo por almacenamiento).

  > **NOTA:** Snapshots cobran por almacenamiento. Limpieza evita costos residuales.
  {: .lab-note .warning .compact}

  ```bash
  source scripts/env.sh
  gcloud compute snapshots delete "$SNAPSHOT_NAME" --quiet || true
  ```
  ```bash
  gcloud compute snapshots describe "$SNAPSHOT_NAME" >/dev/null 2>&1 || echo "OK: snapshot eliminado" | tee outputs/cleanup_snapshot.txt
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[5] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}