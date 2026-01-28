---
layout: lab
title: "Práctica 13: Crear rol personalizado con privilegio mínimo y asignar a un usuario"
permalink: /lab13/lab13/
images_base: /labs/lab13/img
duration: "45 minutos"
objective:
  - "Crear un **rol personalizado** (Custom Role) a nivel **proyecto** con **privilegio mínimo** para **ver** instancias de Compute Engine (sin crear/modificar), y asignarlo a un **usuario** (principal) validando el acceso con **Policy Troubleshooter** y/o prueba en ventana incógnita."
prerequisites:
  - "Proyecto de Google Cloud con **facturación habilitada** (solo si creas la VM de prueba)."
  - "Permisos para: **IAM Role Administrator** (crear roles) y **Project IAM Admin** (asignar roles)."
  - "Acceso a **Google Cloud Console** y a **Cloud Shell (gcloud)**."
introduction: |
  El principio de **menor privilegio** significa otorgar **solo** los permisos necesarios para realizar una tarea específica, y nada más.

  En Google Cloud, puedes lograrlo con:
  - **Predefined roles** (recomendado cuando encaja).
  - **Custom roles** (cuando necesitas un “traje a la medida” con permisos puntuales).

  En esta práctica crearás un rol personalizado a nivel proyecto que **solo** permite **listar y ver detalles** de VMs en Compute Engine. Luego lo asignarás a un usuario y comprobarás (sin asumir nada) si ese usuario **puede ver** las instancias pero **no puede** crearlas ni modificarlas.
slug: lab13
lab_number: 13
final_result: >
  Al finalizar, el proyecto contará con un rol personalizado (privilegio mínimo) y un usuario con ese rol asignado. Podrás demostrar el acceso con Policy Troubleshooter y, opcionalmente, validarlo en una sesión separada (incógnito) verificando que el usuario puede listar VMs pero no crear ni administrar recursos. La limpieza quedará como una tarea separada (opcional) para eliminar lo creado (VM/binding/rol) y dejar el proyecto “limpio”.
notes:
  - "**Costo:** Crear roles IAM y asignarlos **no tiene costo directo**. Si creas una VM para pruebas, esa VM y su disco sí generan costo; elimínalos al final."
  - "En entornos corporativos, tu organización puede imponer políticas (Organization Policies) que afecten creación de recursos o asignación de IAM."
  - "Un rol personalizado solo puede incluir **permisos soportados** para custom roles; si incluyes permisos no soportados, fallará la creación."
references:
  - text: "Crear y administrar roles personalizados (Custom Roles)"
    url: https://cloud.google.com/iam/docs/creating-custom-roles
  - text: "Descripción general de roles de IAM (predefinidos vs personalizados)"
    url: https://cloud.google.com/iam/docs/roles-overview
  - text: "Otorgar roles con la consola (IAM)"
    url: https://cloud.google.com/iam/docs/grant-role-console
  - text: "Troubleshoot IAM permissions (Policy Troubleshooter)"
    url: https://cloud.google.com/policy-intelligence/docs/troubleshoot-access
  - text: "Compute Engine IAM roles and permissions"
    url: https://cloud.google.com/compute/docs/access/iam
prev: /lab12/lab12/
next: /lab14/lab14/
---

---

## Instrucciones generales

- Esta práctica es **mayormente por interfaz gráfica (UI)** para crear el rol y asignarlo.
- Usaremos **Cloud Shell** para:
  - Validar APIs habilitadas,
  - Guardar evidencias (`outputs/`),
  - Verificar el rol y el binding por `gcloud`,
  - Comparar el alcance con un rol **predefinido**,
  - Verificación del acceso (Troubleshooter),
  - **Limpieza** para eliminar lo creado.

Carpeta de trabajo (Cloud Shell):
- `~/labs-gcp-engineer/lab13/` con `scripts/` y `outputs/`.

---

### Tarea 1. Preparar el entorno y variables del laboratorio (Usuario: Admin)

> **Tiempo estimado:** 7 minutos
{: .lab-note .info .compact}

En esta tarea iniciarás Cloud Shell, validarás el proyecto activo, crearás la carpeta del laboratorio y definirás variables (zona, VM opcional, rol y usuario objetivo).

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

- {% include step_label.html %} Crea la carpeta del laboratorio 13 y subcarpetas estándar.

  > **NOTA:** Cada práctica en su carpeta (manifests/scripts/outputs) mantiene el trabajo ordenado y reproducible.
  {: .lab-note .info .compact}

  ```bash
  cd ~
  mkdir -p labs-gcp-engineer/lab13/{scripts,outputs}
  ```
  ```bash
  cd labs-gcp-engineer/lab13
  ```
  {% include step_image.html %}

- {% include step_label.html %} Habilita APIs necesarias (si tu proyecto es nuevo o no estaban habilitadas) y guarda evidencia.

  > **NOTA:** IAM/Resource Manager son necesarios para roles/bindings. Compute solo es necesario si crearás la VM de prueba.
  {: .lab-note .info .compact}

  ```bash
  gcloud services enable iam.googleapis.com cloudresourcemanager.googleapis.com compute.googleapis.com
  ```
  {% include step_image.html %}
  ```bash
  gcloud services list --enabled \
    --filter="name:(iam.googleapis.com OR cloudresourcemanager.googleapis.com OR compute.googleapis.com)" \
    --format="table(name,title)" | tee outputs/enabled_apis.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Crea `scripts/env.sh` con variables del laboratorio y cárgalas.

  > **IMPORTANTE:** Debes editar `TARGET_USER_EMAIL` con el email real del usuario al que asignarás el rol (sin ese dato no se puede validar).
  {: .lab-note .important .compact}

  ```bash
  cat > scripts/env.sh <<'EOF'
  export REGION="us-central1"
  export ZONE="us-central1-a"

  # Recurso opcional de prueba (VM) para validar visualmente "read-only"
  export VM_NAME="lab13-test-vm"
  export VM_TYPE="e2-micro"

  # Rol personalizado (ID único dentro del proyecto)
  export ROLE_ID="lab13LeastPrivilegeComputeViewer"
  export ROLE_TITLE="LAB13 Least Privilege - Compute Viewer"
  export ROLE_DESC="Permite listar y ver detalles de VMs en Compute Engine (sin crear/modificar)."
  export ROLE_STAGE="GA"

  # Usuario objetivo (solo email; en CLI se usa user:EMAIL)
  export TARGET_USER_EMAIL="REEMPLAZA_CON_TU_USUARIO@EJEMPLO.COM"
  EOF
  ```
  ```bash
  source scripts/env.sh
  sed -n '1,200p' scripts/env.sh | tee outputs/env_sh.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica que `TARGET_USER_EMAIL` fue actualizado (no debe contener “REEMPLAZA”).

  > **NOTA:** Esto evita avanzar y fallar más adelante al asignar el rol.
  {: .lab-note .warning .compact}

  ```bash
  source scripts/env.sh
  echo "TARGET_USER_EMAIL=$TARGET_USER_EMAIL" | tee outputs/target_user_email.txt
  echo "$TARGET_USER_EMAIL" | grep -q "REEMPLAZA" \
    && echo "ERROR: actualiza TARGET_USER_EMAIL en scripts/env.sh" && false \
    || echo "OK: email configurado"
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica estructura del laboratorio y guarda evidencia.

  > **NOTA:** Evidencia base útil para auditoría o troubleshooting posterior.
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

### Tarea 2. Crear una VM de prueba para validar el acceso del rol (Usuario: Admin)

> **Tiempo estimado:** 10 minutos
{: .lab-note .info .compact}

En esta tarea crearás una VM pequeña para que el usuario objetivo tenga un recurso real que “ver”. Si tu organización restringe la creación de VMs, omite esta tarea y valida solo con Policy Troubleshooter.

#### Tarea 2.1

- {% include step_label.html %} En la consola, ve a **Compute Engine** luego **VM instances**.

  {% include step_image.html %}

- {% include step_label.html %} Clic en **Create instance**

  > **NOTA:** El rol mínimo que crearás se enfocará en permisos de “vista” sobre VMs.
  {: .lab-note .info .compact}

  {% include step_image.html %}

- {% include step_label.html %} Configura los siguientes datos:

  > **NOTA:** Usa una VM pequeña para reducir costo. No necesitas abrir puertos ni instalar software.
  {: .lab-note .info .compact}

  - Name: `lab13-test-vm`
  - Region: **us-central1**
  - Zone: **us-central1-a**

  {% include step_image.html %}

  - Machine type: **e2-medium**

  {% include step_image.html %}

  - Boot disk: **Debian GNU/Linux 12 (bookworm)**
  
  {% include step_image.html %}

- {% include step_label.html %} Clic en el botón **Create**

- {% include step_label.html %} Confirma por CLI que la VM quedó `RUNNING` y guarda evidencia.

  > **NOTA:** Esto asegura que el recurso existe antes de validar IAM.
  {: .lab-note .important .compact}

  ```bash
  source scripts/env.sh
  gcloud compute instances list --filter="name=($VM_NAME)" \
    --format="table(name,zone,status)" | tee outputs/vm_list.txt
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[1] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 3. Diseñar privilegio mínimo, validar permisos soportados y crear el rol personalizado (Usuario: Admin)

> **Tiempo estimado:** 14 minutos
{: .lab-note .info .compact}

En esta tarea harás 3 cosas prácticas para “privilegio mínimo real”:
- 1) Validar que los permisos elegidos son **testables y soportados** para custom roles.  
- 2) Crear el rol en UI con **solo** esos permisos.  
- 3) Comparar (ejemplo) contra un rol predefinido para ver por qué a veces un custom role es necesario.

#### Tarea 3.1

- {% include step_label.html %} Define **PROJECT_ID** desde la config actual

  ```bash
  PROJECT_ID="$(gcloud config get-value project)"
  echo "PROJECT_ID=$PROJECT_ID"
  ```
  {% include step_image.html %}

- {% include step_label.html %} Obtén el **PROJECT_NUMBER**

  ```bash
  PROJECT_NUMBER="$(gcloud projects describe "$PROJECT_ID" --format='value(projectNumber)')"
  echo "PROJECT_NUMBER=$PROJECT_NUMBER"
  ```
  {% include step_image.html %}

- {% include step_label.html %} Valida por CLI que los permisos que quieres usar son **testables** sobre el proyecto.

  > **NOTA:** Esto te ayuda a evitar “permisos no válidos” antes de crear el rol (muy útil en entornos corporativos).
  {: .lab-note .important .compact}

  ```bash
  source scripts/env.sh
  gcloud iam list-testable-permissions \
    "//cloudresourcemanager.googleapis.com/projects/$PROJECT_NUMBER" \
    --filter='name:compute.instances.list OR name:compute.instances.get OR name:compute.zones.list OR name:compute.projects.get' \
    --format='table(name,title)' \
  | tee outputs/testable_permissions.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} (Ejemplo) Compara el alcance de un rol predefinido vs tu custom role.

  > **NOTA:** `roles/compute.viewer` suele incluir **muchos más permisos** que solo `instances.list/get`. Esto ilustra por qué el “predefinido” a veces no encaja para mínimo privilegio.
  {: .lab-note .info .compact}

  ```bash
  gcloud iam roles describe roles/compute.viewer \
    --format="yaml(name,title,description,stage,includedPermissions)" \
    | head -n 80 | tee outputs/predefined_compute_viewer_head.yaml
  echo "TIP: si quieres el listado completo, quita el head y guarda todo el YAML en outputs/."
  ```
  {% include step_image.html %}

- {% include step_label.html %} En la consola, ve a **IAM & Admin** luego **Roles**.

  {% include step_image.html %}

- {% include step_label.html %} Haz clic en **Create role**.

  > **NOTA:** Los roles personalizados se gestionan desde “Roles” (no desde la pantalla de usuarios IAM).
  {: .lab-note .info .compact}

  {% include step_image.html %}

- {% include step_label.html %} Configura los datos del **Custom role**.

  > **NOTA:** El **ID** debe ser único dentro del proyecto. El “stage” ayuda a clasificar madurez (GA/BETA/ALPHA/DEPRECATED).
  {: .lab-note .info .compact}

  - Title: `LAB13 Least Privilege - Compute Viewer`
  - Description: `Permite listar y ver detalles de VM (sin crear/modificar)`
  - ID: `lab13LeastPrivilegeComputeViewer`
  - Role launch stage: `General Availability`

  {% include step_image.html %}

- {% include step_label.html %} Da clic en **Add permissions** y agrega **exactamente** los siguientes permisos:

  > **NOTA:** Estos permisos permiten **ver** instancias y obtener contexto del proyecto/zona. No incluyas `create/update/delete` para mantener privilegio mínimo.
  {: .lab-note .important .compact}

  - `compute.instances.list`
  - `compute.instances.get`
  - `compute.zones.list`
  - `compute.projects.get`
  - `resourcemanager.projects.get`

  {% include step_image.html %}

- {% include step_label.html %} Clic en **Create** para crear el rol.

  > **NOTA:** El rol quedará como `projects/PROJECT_ID/roles/ROLE_ID`.
  {: .lab-note .info .compact}

  {% include step_image.html %}

- {% include step_label.html %} Valida por Cloud Shell que el rol existe y lista sus permisos (evidencia).

  > **NOTA:** Esta validación asegura que el rol quedó con **los permisos correctos** (y solo esos).
  {: .lab-note .important .compact}

  ```bash
  source scripts/env.sh
  gcloud iam roles describe "$ROLE_ID" --project "$PROJECT_ID" \
    --format="yaml(name,title,description,stage,includedPermissions)" \
    | tee outputs/custom_role_describe.yaml
  ```
  {% include step_image.html %}

- {% include step_label.html %} Guarda una “plantilla” YAML del rol (para reproducibilidad / IaC).

  > **NOTA:** En equipos reales, este YAML se versiona en Git y se aprueba por Change Management.
  {: .lab-note .info .compact}

  ```bash
  source scripts/env.sh
  cat > scripts/role.yaml <<EOF
  title: "${ROLE_TITLE}"
  description: "${ROLE_DESC}"
  stage: "${ROLE_STAGE}"
  includedPermissions:
  - compute.instances.list
  - compute.instances.get
  - compute.zones.list
  - compute.projects.get
  - resourcemanager.projects.get
  EOF

  sed -n '1,200p' scripts/role.yaml | tee outputs/role_yaml.txt
  ```

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[2] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 4. Asignar el rol personalizado a un usuario (Usuario: Admin)

> **Tiempo estimado:** 9 minutos
{: .lab-note .info .compact}

En esta tarea asignarás el rol personalizado al usuario objetivo y validarás la política IAM del proyecto para confirmar el binding.

#### Tarea 4.1

- {% include step_label.html %} En la consola, ve a **IAM & Admin** luego en **IAM**.

  > **NOTA:** Aquí se gestionan los bindings (quién tiene qué rol) a nivel proyecto.
  {: .lab-note .info .compact}

  {% include step_image.html %}

- {% include step_label.html %} Clic en **Grant access**.

  {% include step_image.html %}

- {% include step_label.html %} Configura los siguientes datos:

  > **NOTA:** En UI no necesitas el prefijo `user:`; solo pega el email del principal.
  {: .lab-note .info .compact}

  - New principals: **(pega `TARGET_USER_EMAIL`)**
  - Select a role: **Custom** y **LAB13 Least Privilege - Compute Viewer**

  {% include step_image.html %}

- {% include step_label.html %} Clic en **Save**

- {% include step_label.html %} Agrega un segundo rol llamado **Viewer** de la sección **Basic**

- {% include step_label.html %} Verifica por Cloud Shell que el binding existe (sin imprimir toda la política).

  > **NOTA:** Este filtro confirma el binding exacto (rol + usuario) sin mostrar toda la política del proyecto.
  {: .lab-note .important .compact}

  ```bash
  source scripts/env.sh
  gcloud projects get-iam-policy "$PROJECT_ID" \
    --flatten="bindings[].members" \
    --filter="bindings.role:projects/$PROJECT_ID/roles/$ROLE_ID AND bindings.members:user:$TARGET_USER_EMAIL" \
    --format="table(bindings.role,bindings.members)" \
    | tee outputs/iam_binding_check.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Exporta un snapshot parcial de la política IAM (útil para auditoría).

  > **NOTA:** La política completa puede ser grande; en laboratorio suele bastar con una vista parcial.
  {: .lab-note .info .compact}

  ```bash
  source scripts/env.sh
  gcloud projects get-iam-policy "$PROJECT_ID" --format=json \
    | head -n 120 | tee outputs/iam_policy_head.json
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[3] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 5. Verificar acceso (Troubleshooter) y pruebas negativas

> **Tiempo estimado:** 5 minutos
{: .lab-note .info .compact}

En esta tarea comprobarás que el usuario **sí** tiene el permiso `compute.instances.list` y que el rol **no** otorga acciones administrativas. Aquí **NO** se elimina nada: la limpieza queda en la **Tarea 6** (separada).

#### Tarea 5.1 (Usuario: Admin)

- {% include step_label.html %} Da clic en **Policy Troubleshooter** del menu lateral izquierdo.

  {% include step_image.html %}

- {% include step_label.html %} **Prueba positiva #1:** Configura los siguientes datos par averificar que el usuario puede **obtener** VMs.

  > **NOTA:** Policy Troubleshooter explica por qué un usuario tiene o no un permiso, sin necesidad de “loguearte” como ese usuario.
  {: .lab-note .important .compact}

  - Principal email: **TARGET_USER_EMAIL**
  - Resource: **TU_PROYECTO** - **Compute** y selecciona la maquina del laboratorio.
  - Clic **Select**

  {% include step_image.html %}

  - Permission: `compute.instances.get`

  {% include step_image.html %}

- {% include step_label.html %} Clic en **Check access**

  {% include step_image.html %}

- {% include step_label.html %} **Prueba negativa #1:** verifica que el usuario **NO** puede crear VMs.

  > **NOTA:** Esta prueba debe resultar **DENIED** si tu rol realmente es de privilegio mínimo.
  {: .lab-note .important .compact}

  - Permission: `compute.instances.createTagBinding`

  {% include step_image.html %}

- {% include step_label.html %} Clic en **Check access**

  {% include step_image.html %}

- {% include step_label.html %} **Prueba negativa #2:** verifica que el usuario **NO** puede modificar metadatos de una VM.

  > **NOTA:** Esta prueba debe resultar **DENIED**. Es un ejemplo realista de “acción peligrosa” (podría inyectar startup scripts).
  {: .lab-note .important .compact}

  - Permission: `compute.instances.setMetadata`

  {% include step_image.html %}

- {% include step_label.html %} Clic en **Check access**

  {% include step_image.html %}

#### Tarea 5.2 (Usuario: Temporal)

- {% include step_label.html %} **(Opcional)** Prueba real en sesión separada (incógnito/InPrivate).

  > **NOTA:** Validación visual: el usuario debería poder **ver** la lista, pero no debería poder crear/editar.
  {: .lab-note .info .compact}

  - Abre ventana **Incógnito**
  - Inicia sesión con [**AQUÍ**](https://cloud.google.com/cloud-console).
  - Coloca tu correo **TARGET_USER_EMAIL**
  - Selecciona tu proyecto.
  - Abre **Compute Engine** luego **VM instances**
  - Verifica que puede **ver** la lista (incluida `lab13-test-vm`).

  {% include step_image.html %}

- {% include step_label.html %} Ahora selecciona la instancia y da clic en **Delete** o **Stop**. No deberias poder eliminarla o detenerla.

  {% include step_image.html %}

#### Tarea 5.3 (Usuario: Admin)

- {% include step_label.html %} Evidencia extra por CLI: guarda el “resumen” del rol y el binding en un solo archivo.

  > **NOTA:** Este archivo suele servir como evidencia para un reporte de seguridad o auditoría interna.
  {: .lab-note .info .compact}

  ```bash
  source scripts/env.sh
  {
    echo "=== ROLE ==="
    gcloud iam roles describe "$ROLE_ID" --project "$PROJECT_ID" --format="yaml(name,title,stage,includedPermissions)"
    echo
    echo "=== BINDING ==="
    gcloud projects get-iam-policy "$PROJECT_ID" \
      --flatten="bindings[].members" \
      --filter="bindings.role:projects/$PROJECT_ID/roles/$ROLE_ID AND bindings.members:user:$TARGET_USER_EMAIL" \
      --format="yaml(bindings.role,bindings.members)"
  } | tee outputs/summary_role_and_binding.yaml
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[4] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 6. Limpieza (Usuario: Admin)

> **Tiempo estimado:** 6 minutos
{: .lab-note .warning .compact}

Esta tarea **revierte** el laboratorio. La limpieza se hace en este orden para evitar errores:

1) **Eliminar VM** (si se creó) para evitar costos.  
2) **Remover binding** (revocar acceso del usuario).  
3) **Eliminar rol personalizado** (evitar dejar roles huérfanos).  

> **NOTA:** Ejecuta esta limpieza con tu usuario administrador del laboratorio (no con `TARGET_USER_EMAIL`).
{: .lab-note .important .compact}

#### Tarea 6.1 — Eliminar la VM de prueba

- {% include step_label.html %} Limpieza de VM.

  > **NOTA:** Esto elimina costo residual de Compute Engine (VM + disco).
  {: .lab-note .info .compact}

  ```bash
  source scripts/env.sh

  # Intenta eliminar (no falla el lab si no existe)
  gcloud compute instances delete "$VM_NAME" --zone "$ZONE" --quiet || true

  # Evidencia: confirmar que ya no existe
  if gcloud compute instances describe "$VM_NAME" --zone "$ZONE" >/dev/null 2>&1; then
    echo "WARN: la VM aún existe (revisa permisos/region/zona)." | tee outputs/cleanup_vm.txt
  else
    echo "OK: VM eliminada o no existía." | tee outputs/cleanup_vm.txt
  fi
  ```
  {% include step_image.html %}

#### Tarea 6.2 — Remover el binding IAM (revocar el rol al usuario)

- {% include step_label.html %} Revoca el rol (binding) por CLI y valida que ya no aparece.

  > **NOTA:** Esto regresa el proyecto a su estado previo (usuario sin el rol del lab).
  {: .lab-note .info .compact}

  ```bash
  source scripts/env.sh

  remove_if_exists () {
    local ROLE="$1"
    gcloud projects remove-iam-policy-binding "$PROJECT_ID" \
      --member="user:$TARGET_USER_EMAIL" \
      --role="$ROLE" \
      --quiet >/dev/null 2>&1 || true
  }

  remove_if_exists "projects/$PROJECT_ID/roles/$ROLE_ID"
  remove_if_exists "roles/viewer"

  OUT="$(gcloud projects get-iam-policy "$PROJECT_ID" \
    --flatten="bindings[].members" \
    --filter="bindings.members:user:$TARGET_USER_EMAIL AND (bindings.role:projects/$PROJECT_ID/roles/$ROLE_ID OR bindings.role:roles/viewer)" \
    --format="value(bindings.role)")"

  if [[ -z "$OUT" ]]; then
    echo "OK: roles removidos (custom + viewer)"
  else
    echo "WARN: aún quedan roles:"
    echo "$OUT"
  fi | tee outputs/iam_bindings_after_remove_check.txt
  ```
  {% include step_image.html %}

#### Tarea 6.3 — Eliminar el rol personalizado

- {% include step_label.html %} Elimina el rol personalizado por CLI y valida que ya no existe.

  > **NOTA:** Esto evita dejar roles “huérfanos” que podrían asignarse accidentalmente en el futuro.
  {: .lab-note .warning .compact}

  ```bash
  source scripts/env.sh

  gcloud iam roles delete "$ROLE_ID" --project "$PROJECT_ID" --quiet || true

  # Evidencia: confirmar que ya no existe
  if gcloud iam roles describe "$ROLE_ID" --project "$PROJECT_ID" >/dev/null 2>&1; then
    echo "WARN: el rol aún existe (revisa permisos o si hay conflicto)." | tee outputs/cleanup_role.txt
  else
    echo "OK: rol eliminado o no existía." | tee outputs/cleanup_role.txt
  fi
  ```
  {% include step_image.html %}

- {% include step_label.html %} Evidencia final “estado limpio”: lista recursos del lab (deberían no aparecer).

  > **NOTA:** Puede que aparezca el role, es normal ya que queda en un estado **Soft Delete** desaparecera solo en un periodo aproximado de **7 dias**.
  {: .lab-note .warning .compact}

  ```bash
  source scripts/env.sh
  {
    echo "=== CHECK VM (should be empty) ==="
    gcloud compute instances list --filter="name=($VM_NAME)" --format="table(name,zone,status)" || true
    echo
    echo "=== CHECK BINDING (should be empty) ==="
    gcloud projects get-iam-policy "$PROJECT_ID" \
      --flatten="bindings[].members" \
      --filter="bindings.role:projects/$PROJECT_ID/roles/$ROLE_ID AND bindings.members:user:$TARGET_USER_EMAIL" \
      --format="table(bindings.role,bindings.members)" || true
    echo
    echo "=== CHECK ROLE (should fail/empty) ==="
    gcloud iam roles describe "$ROLE_ID" --project "$PROJECT_ID" --format="value(name)" || true
  } | tee outputs/cleanup_final_state.txt
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[5] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}