---
layout: lab
title: "Práctica 15: Proteger datos sensibles con VPC Service Controls y validar acceso"
permalink: /lab15/lab15/
images_base: /labs/lab15/img
duration: "30 minutos"
objective:
  - "Configurar **VPC Service Controls** (VPC-SC) para proteger datos sensibles en **Cloud Storage** dentro de un **perímetro de servicio**, generar evidencias de violación en **Dry run**, aplicar el perímetro en modo **Enforced** para bloquear accesos no confiables y finalmente permitir acceso controlado mediante un **Access Level** (allowlist por IP), validando todo con pruebas y logs."
prerequisites:
  - "**Proyecto** de Google Cloud con **facturación habilitada**."
  - "**Organización** (Cloud Identity / Google Workspace) asociada al proyecto (VPC-SC es control a nivel organización)."
  - "Permisos mínimos recomendados en la **organización**: **Access Context Manager Admin** (`roles/accesscontextmanager.policyAdmin`) y acceso para ver la organización (ej. **Organization Viewer**)."
  - "Permisos en el **proyecto** para crear bucket/objeto y ver logs (ej. **Storage Admin/Creator** y **Logs Viewer**)."
  - "Acceso a **Google Cloud Console** y **Cloud Shell**."
introduction: |
  **VPC Service Controls** añade una capa de seguridad “tipo perímetro” para servicios multi-tenant (Cloud Storage, BigQuery, etc.) y ayuda a mitigar **exfiltración de datos** si un usuario/servicio intenta acceder desde ubicaciones o contextos no confiables.

  En esta práctica crearás un bucket con un archivo “sensible” y luego:
  - Activarás un perímetro en **Dry run** para **ver** violaciones sin bloquear.
  - Cambiarás a **Enforced** para **bloquear** accesos no confiables.
  - Agregarás un **Access Level** para permitir acceso controlado (allowlist por IP) y comprobar que el acceso vuelve a funcionar.
slug: lab15
lab_number: 15
final_result: >
  Al finalizar, tendrás un perímetro VPC-SC protegiendo Cloud Storage en tu proyecto, evidencias de violación (Dry run) en Cloud Logging, bloqueo real en Enforced (403 con identificador VPC-SC) y una excepción controlada mediante Access Level que permite el acceso desde una IP autorizada.
notes:
  - "**Costo:** VPC Service Controls **no tiene cargo adicional**, pero Cloud Storage cobra por almacenamiento/operaciones (mínimo en este laboratorio)."
  - "Este laboratorio es **independiente**: crea bucket, access level y perímetro desde cero."
  - "La UI y terminología pueden variar ligeramente según tu organización y políticas (Org Policies)."
references:
  - text: "Create a service perimeter (VPC Service Controls)"
    url: https://docs.cloud.google.com/vpc-service-controls/docs/create-service-perimeters
  - text: "Access control with IAM (roles requeridos para VPC-SC)"
    url: https://docs.cloud.google.com/vpc-service-controls/docs/access-control?hl=es-419
  - text: "VPC Service Controls audit logging (campos vpcServiceControlsUniqueId, dryRun, etc.)"
    url: https://docs.cloud.google.com/vpc-service-controls/docs/audit-logging
  - text: "Retrieve VPC Service Controls errors from audit logs"
    url: https://docs.cloud.google.com/vpc-service-controls/docs/retrieve-troubleshoot-errors
  - text: "VPC Service Controls pricing (sin cargo adicional)"
    url: https://cloud.google.com/vpc-service-controls/pricing
prev: /lab14/lab14/
next: /lab16/lab16/
---

---

### Tarea 1. Preparar entorno, verificar organización y crear carpeta del laboratorio (UI + Cloud Shell)

> **Tiempo estimado:** 5 minutos
{: .lab-note .info .compact}

En esta tarea iniciarás Cloud Shell, verificarás que tu proyecto pertenece a una **organización**, validarás que puedes ver/usar **Access Context Manager**, habilitarás APIs necesarias y crearás la carpeta del laboratorio con variables en `scripts/env.sh`.

#### Tarea 1.1

- {% include step_label.html %} Abre **Google Cloud Console** y lanza **Cloud Shell** (ícono `>_`).

  > **NOTA:** Cloud Shell ya trae `gcloud` y autenticación lista; úsalo para ejecutar pasos repetibles y guardar evidencias.
  {: .lab-note .info .compact}

  {% include step_image.html %}

- {% include step_label.html %} Verifica el proyecto activo y expórtalo como variable.

  > **NOTA:** Esto evita crear perímetros/políticas en el proyecto equivocado.
  {: .lab-note .warning .compact}

  ```bash
  gcloud config get-value project
  export PROJECT_ID="$(gcloud config get-value project)"
  echo "PROJECT_ID=$PROJECT_ID"
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica que el proyecto está asociado a una **organización** (requisito clave para VPC-SC).

  > **NOTA:** Si el `ORG_ID` queda vacío, este proyecto **no** puede completar VPC-SC (es control a nivel organización).
  {: .lab-note .important .compact}

  ```bash
  # Obtiene el ORGANIZATION_ID (si existe)
  export ORG_ID="$(gcloud projects get-ancestors "$PROJECT_ID" \
    --filter="type:organization" --format="value(ID)" | head -n 1)"
  echo "ORG_ID=$ORG_ID"
  ```
  {% include step_image.html %}

- {% include step_label.html %} Si `ORG_ID` salió vacío, confirma en UI:
  - **IAM & Admin → Settings** y revisa el campo **Organization**.

  > **NOTA:** Si no hay organización, detén aquí y usa un proyecto dentro de una org (Cloud Identity / Workspace).
  {: .lab-note .important .compact}

  {% include step_image.html %}

- {% include step_label.html %} Habilita APIs necesarias (si el proyecto es nuevo o no están habilitadas).

  > **NOTA:** Sin Access Context Manager no podrás crear **Access Levels** ni **Service Perimeters**.
  {: .lab-note .important .compact}

  ```bash
  gcloud services enable \
    accesscontextmanager.googleapis.com \
    storage.googleapis.com \
    logging.googleapis.com

  gcloud services list --enabled \
    --filter="name:(accesscontextmanager.googleapis.com OR storage.googleapis.com OR logging.googleapis.com)" \
    --format="table(name)"
  ```
  {% include step_image.html %}

- {% include step_label.html %} Crea la estructura de carpetas de la práctica y entra al directorio.

  > **NOTA:** Mantener `scripts/` y `outputs/` por práctica facilita evidencias y limpieza.
  {: .lab-note .info .compact}

  ```bash
  cd ~
  mkdir -p labs-gcp-finops/lab15/{scripts,outputs}
  cd labs-gcp-finops/lab15
  ```
  {% include step_image.html %}

- {% include step_label.html %} Crea `scripts/env.sh` con variables base del laboratorio y cárgalo.

  > **NOTA:** Algunas variables se completan después (Access Policy, bucket único, etc.).
  {: .lab-note .info .compact}

  ```bash
  cat > scripts/env.sh <<'EOF'
  export REGION="us-central1"

  # Org / Access Context (se descubre)
  export ORG_ID=""
  export POLICY_NAME=""     # formato: accessPolicies/123456789

  # Recursos
  export BUCKET_NAME=""

  # Access Context / VPC-SC
  export ACCESS_LEVEL_ID="lab15_trusted_ip"     # ID técnico (sin espacios)
  export ACCESS_LEVEL_TITLE="LAB15 Trusted IP"
  export PERIMETER_NAME="lab15-perimeter"

  # Servicio restringido
  export RESTRICTED_SERVICE="storage.googleapis.com"
  EOF

  source scripts/env.sh
  sed -n '1,220p' scripts/env.sh | tee outputs/env_sh.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica estructura del laboratorio y guarda evidencia.

  > **NOTA:** Esta evidencia es útil si algo falla más adelante (paths/archivos).
  {: .lab-note .info .compact}

  ```bash
  pwd | tee outputs/pwd.txt
  ls -la | tee outputs/ls_root.txt
  ls -la scripts outputs | tee outputs/ls_subfolders.txt
  ```
  {% include step_image.html %}

{% assign results = site.data["task-results"][page.slug].results %}
{% capture r1 %}{{ results[0] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 2. Crear “datos sensibles” en Cloud Storage (UI + Cloud Shell)

> **Tiempo estimado:** 5 minutos
{: .lab-note .info .compact}

En esta tarea crearás un bucket (seguro por defecto) y subirás un archivo `sensitive.txt` para simular un dato sensible. Esto será el objetivo para probar el perímetro.

#### Tarea 2.1

- {% include step_label.html %} Genera un nombre de bucket globalmente único y persístelo en `scripts/env.sh` (sin asumir).

  > **NOTA:** Cloud Storage exige **unicidad global**; este método reduce colisiones y hace el lab independiente.
  {: .lab-note .info .compact}

  ```bash
  source scripts/env.sh
  export PROJECT_ID="$(gcloud config get-value project)"

  PROJ_SHORT="$(echo "$PROJECT_ID" | tr '[:upper:]' '[:lower:]' | tr '_' '-' | cut -c1-18)"
  SUFFIX="$(date +%s | tail -c 6)"
  export BUCKET_NAME="lab15-vpcsc-${PROJ_SHORT}-${SUFFIX}"

  # Persistir en env.sh (y recargar)
  grep -q '^export BUCKET_NAME=' scripts/env.sh && sed -i 's|^export BUCKET_NAME=.*|export BUCKET_NAME="'"$BUCKET_NAME"'"|' scripts/env.sh \
    || echo 'export BUCKET_NAME="'"$BUCKET_NAME"'"' >> scripts/env.sh
  source scripts/env.sh

  echo "BUCKET_NAME=$BUCKET_NAME" | tee outputs/bucket_name.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} En la consola, ve a **Cloud Storage → Buckets → Create**.

  > **NOTA:** Usaremos la UI para que el flujo sea fácil de replicar por participantes.
  {: .lab-note .info .compact}

  {% include step_image.html %}

- {% include step_label.html %} Crea el bucket con:
  - Name: el valor `BUCKET_NAME` (en `outputs/bucket_name.txt`)
  - Location type: **Region**
  - Location: `us-central1` (o tu región)
  - Default storage class: **Standard**
  - Public access prevention: **Enforced**
  - Uniform bucket-level access: **Enabled**
  - Click **Create**

  > **NOTA:** “Uniform access” evita ACLs dispersas; IAM centraliza permisos y reduce errores.
  {: .lab-note .important .compact}

  {% include step_image.html %}

- {% include step_label.html %} Verifica por CLI que el bucket existe y registra evidencia.

  > **NOTA:** Confirma el bucket real antes de aplicar perímetros.
  {: .lab-note .info .compact}

  ```bash
  source scripts/env.sh
  gcloud storage buckets describe "gs://$BUCKET_NAME" \
    --format="yaml(name,location,uniformBucketLevelAccess,publicAccessPrevention)" \
    | tee outputs/bucket_describe.yaml
  ```
  {% include step_image.html %}

- {% include step_label.html %} Crea el archivo “sensible” y súbelo al bucket.

  > **NOTA:** Este archivo será el objetivo para probar si VPC-SC bloquea/permite lectura.
  {: .lab-note .info .compact}

  ```bash
  source scripts/env.sh
  cat > outputs/sensitive.txt <<'EOF'
  CONFIDENCIAL: Datos de ejemplo para laboratorio VPC-SC.
  No compartir fuera del perímetro.
  EOF

  gcloud storage cp outputs/sensitive.txt "gs://$BUCKET_NAME/sensitive.txt"
  gcloud storage ls "gs://$BUCKET_NAME/" | tee outputs/bucket_ls_before_vpcsc.txt
  ```
  {% include step_image.html %}

{% assign results = site.data["task-results"][page.slug].results %}
{% capture r2 %}{{ results[1] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r2 %}

---

### Tarea 3. Identificar Access Policy y crear Access Level (allowlist por IP) (UI + Cloud Shell)

> **Tiempo estimado:** 6 minutos
{: .lab-note .info .compact}

En esta tarea identificarás (o crearás si tu entorno lo permite) una **Access Policy** de Access Context Manager y crearás un **Access Level** basado en la IP pública desde la que harás pruebas (allowlist). Ese Access Level será la “excepción controlada” en Enforced.

#### Tarea 3.1

- {% include step_label.html %} Registra `ORG_ID` en `scripts/env.sh` y valida (no asumas).

  > **NOTA:** Esto evita tener que re-calcularlo en pasos posteriores.
  {: .lab-note .info .compact}

  ```bash
  source scripts/env.sh
  export PROJECT_ID="$(gcloud config get-value project)"
  ORG_ID_FOUND="$(gcloud projects get-ancestors "$PROJECT_ID" --filter="type:organization" --format="value(ID)" | head -n 1)"
  echo "ORG_ID_FOUND=$ORG_ID_FOUND" | tee outputs/org_id.txt

  grep -q '^export ORG_ID=' scripts/env.sh && sed -i 's|^export ORG_ID=.*|export ORG_ID="'"$ORG_ID_FOUND"'"|' scripts/env.sh \
    || echo 'export ORG_ID="'"$ORG_ID_FOUND"'"' >> scripts/env.sh
  source scripts/env.sh
  ```
  {% include step_image.html %}

- {% include step_label.html %} Lista Access Policies disponibles para la organización y guarda el `POLICY_NAME`.

  > **NOTA:** En organizaciones reales normalmente existe **una** Access Policy corporativa; úsala (no crees otra).
  {: .lab-note .important .compact}

  ```bash
  source scripts/env.sh
  gcloud access-context-manager policies list --organization "$ORG_ID" \
    --format="table(name,title)" | tee outputs/policies_list.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Si tu organización **NO** tiene ninguna Access Policy (caso raro), crea una (solo si tienes permisos).

  > **NOTA:** Si ya existe una policy corporativa, **omite** este paso para evitar duplicados.
  {: .lab-note .warning .compact}

  ```bash
  source scripts/env.sh
  POLICY_COUNT="$(gcloud access-context-manager policies list --organization "$ORG_ID" --format="value(name)" | wc -l | tr -d ' ')"
  if [ "$POLICY_COUNT" = "0" ]; then
    gcloud access-context-manager policies create \
      --organization "$ORG_ID" \
      --title="Access Policy (LAB15)" \
      --format="value(name)" | tee outputs/policy_created.txt
  else
    echo "OK: Ya existe al menos una Access Policy, no se crea otra." | tee outputs/policy_create_skipped.txt
  fi
  ```
  {% include step_image.html %}

- {% include step_label.html %} Selecciona el `POLICY_NAME` efectivo y persístelo en `scripts/env.sh` (sin asumir).

  > **NOTA:** Si tu organización tiene varias policies, usa la que te indiquen (Security/Governance). Para el lab, tomamos la primera por defecto.
  {: .lab-note .warning .compact}

  ```bash
  source scripts/env.sh
  POLICY_NAME_FOUND="$(gcloud access-context-manager policies list --organization "$ORG_ID" --format="value(name)" | head -n 1)"
  echo "POLICY_NAME_FOUND=$POLICY_NAME_FOUND" | tee outputs/policy_name.txt

  grep -q '^export POLICY_NAME=' scripts/env.sh && sed -i 's|^export POLICY_NAME=.*|export POLICY_NAME="'"$POLICY_NAME_FOUND"'"|' scripts/env.sh \
    || echo 'export POLICY_NAME="'"$POLICY_NAME_FOUND"'"' >> scripts/env.sh
  source scripts/env.sh

  grep -n '^export POLICY_NAME=' scripts/env.sh | tee outputs/policy_env_line.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Obtén tu IP pública actual (desde Cloud Shell) y crea una subnet `/32` para allowlist.

  > **NOTA:** Este lab valida principalmente desde **Cloud Shell**; por eso usamos la IP que ve Cloud Shell como origen.
  {: .lab-note .info .compact}

  ```bash
  source scripts/env.sh
  MY_IP="$(
    curl -s https://ifconfig.me 2>/dev/null \
    || curl -s https://api.ipify.org 2>/dev/null \
    || curl -s https://checkip.amazonaws.com 2>/dev/null \
    || true
  )"
  echo "MY_IP=$MY_IP" | tee outputs/my_ip.txt

  MY_IP_CIDR="${MY_IP}/32"
  echo "MY_IP_CIDR=$MY_IP_CIDR" | tee outputs/my_ip_cidr.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} En la consola, ve a **Security → Access Context Manager → Access Levels → New**.

  > **NOTA:** El Access Level será la excepción controlada para permitir acceso desde tu IP en modo Enforced.
  {: .lab-note .important .compact}

  {% include step_image.html %}

- {% include step_label.html %} Crea el Access Level con:
  - Name/ID: `lab15_trusted_ip`
  - Title: `LAB15 Trusted IP`
  - Condition: **IP subnetworks** → agrega `MY_IP_CIDR`
  - Save

  > **NOTA:** Si tu IP cambia (VPN/red distinta), tendrás que actualizar el Access Level para que el “allow” funcione.
  {: .lab-note .warning .compact}

  {% include step_image.html %}

- {% include step_label.html %} Verifica por CLI que el Access Level existe y contiene tu IP.

  > **NOTA:** El nombre completo queda como: `accessPolicies/XYZ/accessLevels/lab15_trusted_ip`.
  {: .lab-note .info .compact}

  ```bash
  source scripts/env.sh
  gcloud access-context-manager levels list --policy "$POLICY_NAME" \
    --format="table(name,title)" | tee outputs/access_levels_list.txt

  LEVEL_FULL_NAME="$(gcloud access-context-manager levels list --policy "$POLICY_NAME" \
    --filter="name~lab15_trusted_ip OR title:LAB15 Trusted IP" \
    --format="value(name)" | head -n 1)"
  echo "LEVEL_FULL_NAME=$LEVEL_FULL_NAME" | tee outputs/access_level_full_name.txt

  gcloud access-context-manager levels describe "$LEVEL_FULL_NAME" --policy "$POLICY_NAME" \
    --format="yaml(name,title,basic.conditions)" | tee outputs/access_level_describe.yaml
  ```
  {% include step_image.html %}

{% assign results = site.data["task-results"][page.slug].results %}
{% capture r3 %}{{ results[2] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r3 %}

---

### Tarea 4. Crear perímetro VPC-SC en Dry run y observar violaciones en logs (UI + Cloud Shell)

> **Tiempo estimado:** 7 minutos
{: .lab-note .info .compact}

En esta tarea crearás un **service perimeter** en modo **Dry run** (no bloquea, solo registra). Luego leerás el archivo sensible para generar evidencias en Cloud Logging (`dryRun=true` + `vpcServiceControlsUniqueId`).

#### Tarea 4.1

- {% include step_label.html %} En la consola, ve a **Security → VPC Service Controls**.
  - Selecciona tu **Access Policy** (la del `POLICY_NAME`) si te lo solicita.

  > **NOTA:** VPC-SC “vive” dentro de la Access Policy (Access Context Manager).
  {: .lab-note .info .compact}

  {% include step_image.html %}

- {% include step_label.html %} Crea un perímetro nuevo en **Dry run**:
  - Mode: **Dry run**
  - Click **New perimeter**
  - Name: `lab15-perimeter`
  - Title: `LAB15 Perimeter (Dry run)`
  - Projects: agrega tu proyecto `PROJECT_ID`
  - Restricted services: agrega **Cloud Storage API** (`storage.googleapis.com`)
  - Access levels: **NO agregues ninguno** todavía

  > **NOTA:** En Dry run puedes “ver qué se rompería” antes de bloquear en Enforced.
  {: .lab-note .important .compact}

  {% include step_image.html %}

- {% include step_label.html %} Genera el evento de acceso (en Dry run debe **funcionar** y generar evidencia de violación).

  > **NOTA:** Si el comando falla aquí, revisa IAM del bucket/objeto o que el perímetro quedó en Dry run (no Enforced).
  {: .lab-note .warning .compact}

  ```bash
  source scripts/env.sh
  gcloud storage cat "gs://$BUCKET_NAME/sensitive.txt" | tee outputs/sensitive_read_dryrun.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Abre **Logging → Logs Explorer** y ejecuta este filtro para encontrar violaciones en Dry run:

  > **NOTA:** Busca `dryRun="true"` y el identificador `vpcServiceControlsUniqueId`.
  {: .lab-note .important .compact}

  ```text
  log_id("cloudaudit.googleapis.com/policy")
  protoPayload.metadata.dryRun="true"
  protoPayload.metadata.vpcServiceControlsUniqueId:*
  ```
  {% include step_image.html %}

- {% include step_label.html %} (Alternativa CLI) Recupera evidencia del evento por `gcloud logging read`.

  > **NOTA:** Este archivo JSON es excelente evidencia para reporte/entrega.
  {: .lab-note .info .compact}

  ```bash
  gcloud logging read \
    'log_id("cloudaudit.googleapis.com/policy") AND protoPayload.metadata.dryRun="true" AND protoPayload.metadata.vpcServiceControlsUniqueId:*' \
    --limit=10 --format="json" | tee outputs/vpcsc_dryrun_logs.json
  ```
  {% include step_image.html %}

- {% include step_label.html %} Extrae el `vpcServiceControlsUniqueId` del log y guárdalo como evidencia.

  > **NOTA:** Si no aparecen logs, ajusta el rango de tiempo en Logs Explorer (Last 30 minutes) y reintenta la lectura del archivo.
  {: .lab-note .warning .compact}

  ```bash
  python3 - <<'PY'
import json
p="outputs/vpcsc_dryrun_logs.json"
try:
    data=json.load(open(p))
    if not data:
        print("NO_LOGS_FOUND")
    else:
        md=data[0].get("protoPayload",{}).get("metadata",{})
        print(md.get("vpcServiceControlsUniqueId","NO_UNIQUE_ID"))
except Exception as e:
    print("PARSE_ERROR:", e)
PY
  ```
  {% include step_image.html %}

{% assign results = site.data["task-results"][page.slug].results %}
{% capture r4 %}{{ results[3] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r4 %}

---

### Tarea 5. Cambiar a Enforced, validar bloqueo y habilitar acceso controlado con Access Level (UI + Cloud Shell)

> **Tiempo estimado:** 7 minutos
{: .lab-note .info .compact}

En esta tarea aplicarás el perímetro en **Enforced** para provocar un bloqueo real (403). Después, agregarás el **Access Level** `lab15_trusted_ip` dentro del perímetro para permitir acceso controlado desde tu IP, y confirmarás que la lectura vuelve a funcionar.

#### Tarea 5.1

- {% include step_label.html %} En la consola, abre el perímetro `lab15-perimeter` y haz clic en **Edit**.

  > **NOTA:** Realizarás dos cambios: 1) Enforced y 2) agregar Access Level permitido.
  {: .lab-note .info .compact}

  {% include step_image.html %}

- {% include step_label.html %} Cambia el modo a **Enforced** y guarda cambios.

  > **NOTA:** Enforced **bloquea** solicitudes que violan el perímetro (ya no solo registra).
  {: .lab-note .important .compact}

  {% include step_image.html %}

- {% include step_label.html %} Prueba el acceso desde Cloud Shell (debe **fallar** con 403 por VPC-SC).

  > **NOTA:** Si NO se bloquea, revisa que el perímetro esté realmente Enforced y que aún NO agregaste Access Levels al perímetro.
  {: .lab-note .warning .compact}

  ```bash
  source scripts/env.sh
  set +e
  gcloud storage cat "gs://$BUCKET_NAME/sensitive.txt" 2>&1 | tee outputs/sensitive_read_enforced_attempt.txt
  set -e
  ```
  {% include step_image.html %}

- {% include step_label.html %} Busca el evento de bloqueo en Logs Explorer (Enforced).

  > **NOTA:** Debe aparecer `dryRun="false"` y un `vpcServiceControlsUniqueId`.
  {: .lab-note .important .compact}

  ```text
  log_id("cloudaudit.googleapis.com/policy")
  protoPayload.metadata.dryRun="false"
  protoPayload.metadata.vpcServiceControlsUniqueId:*
  ```
  {% include step_image.html %}

- {% include step_label.html %} Agrega el Access Level `lab15_trusted_ip` en el perímetro (sección **Access levels / Allowed access levels**) y guarda.

  > **NOTA:** Esto crea la excepción controlada: desde tu IP allowlist, el acceso vuelve a funcionar.
  {: .lab-note .important .compact}

  {% include step_image.html %}

- {% include step_label.html %} Espera propagación (1–3 min) y reintenta leer el archivo sensible (ahora debe **funcionar**).

  > **NOTA:** Si falla, confirma que tu IP actual coincide con la allowlist (`outputs/my_ip_cidr.txt`) y que el Access Level está asociado al perímetro.
  {: .lab-note .warning .compact}

  ```bash
  source scripts/env.sh
  gcloud storage cat "gs://$BUCKET_NAME/sensitive.txt" | tee outputs/sensitive_read_enforced_allowed.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} (Validación extra) Lista el bucket para confirmar acceso controlado.

  > **NOTA:** VPC-SC complementa IAM: aquí no cambiaste IAM del bucket, solo el **contexto** permitido.
  {: .lab-note .info .compact}

  ```bash
  source scripts/env.sh
  gcloud storage ls "gs://$BUCKET_NAME/" | tee outputs/bucket_ls_enforced_allowed.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} (Si algo no cuadra) Guarda evidencia del “policy log” más reciente por CLI.

  > **NOTA:** En incidentes reales, el `vpcServiceControlsUniqueId` y/o el `troubleshootToken` son la mejor evidencia.
  {: .lab-note .important .compact}

  ```bash
  gcloud logging read \
    'log_id("cloudaudit.googleapis.com/policy") AND protoPayload.metadata.vpcServiceControlsUniqueId:*' \
    --limit=5 --format="json" | tee outputs/vpcsc_latest_policy_logs.json
  ```
  {% include step_image.html %}

{% assign results = site.data["task-results"][page.slug].results %}
{% capture r5 %}{{ results[4] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r5 %}

---

### Tarea 6. Limpieza (opcional) para evitar costos y dejar el entorno estable (UI + Cloud Shell)

> **Tiempo estimado:** 3 minutos
{: .lab-note .info .compact}

En esta tarea (opcional) eliminarás el bucket y retirarás el perímetro/Access Level para dejar el entorno como estaba.

#### Tarea 6.1

- {% include step_label.html %} Elimina el bucket (UI recomendado) o por CLI.

  > **NOTA:** Storage cobra por almacenamiento/operaciones; elimina el bucket para evitar cargos residuales.
  {: .lab-note .warning .compact}

  **UI:** Cloud Storage → Buckets → `BUCKET_NAME` → Delete

  ```bash
  source scripts/env.sh
  gcloud storage rm -r "gs://$BUCKET_NAME" || true
  gcloud storage buckets describe "gs://$BUCKET_NAME" >/dev/null 2>&1 || echo "OK: bucket eliminado" | tee outputs/cleanup_bucket.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} En **Security → VPC Service Controls**, elimina el perímetro `lab15-perimeter` (o desactívalo si tu org no permite borrar).

  > **NOTA:** Un perímetro activo puede afectar integraciones futuras; en laboratorios es mejor retirarlo.
  {: .lab-note .important .compact}

  {% include step_image.html %}

- {% include step_label.html %} En **Security → Access Context Manager → Access Levels**, elimina `lab15_trusted_ip` si fue solo para el lab.

  > **NOTA:** Mantén el set de Access Levels limpio para evitar reglas confusas a futuro.
  {: .lab-note .info .compact}

  {% include step_image.html %}

- {% include step_label.html %} Verifica por CLI que el Access Level ya no exista (opcional, evidencia).

  > **NOTA:** Si no tienes permisos para listar, toma evidencia con capturas en UI.
  {: .lab-note .warning .compact}

  ```bash
  source scripts/env.sh
  gcloud access-context-manager levels list --policy "$POLICY_NAME" \
    --filter="name~lab15_trusted_ip OR title:LAB15 Trusted IP" \
    --format="value(name)" | tee outputs/access_level_post_cleanup.txt
  ```
  {% include step_image.html %}

{% assign results = site.data["task-results"][page.slug].results %}
{% capture r6 %}{{ results[5] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r6 %}

---

## Resultado final (de toda la práctica)

- Bucket `BUCKET_NAME` creado con objeto `sensitive.txt`.
- Access Policy identificada/creada y registrada (`POLICY_NAME`).
- Access Level `lab15_trusted_ip` creado con allowlist por IP.
- Perímetro `lab15-perimeter` creado en **Dry run** y evidencias de violación consultadas en logs (`dryRun=true`).
- Perímetro cambiado a **Enforced** y bloqueo real validado (403 / VPC-SC, `dryRun=false`).
- Excepción aplicada con Access Level y acceso restaurado de forma controlada.

---

## Notas y/o Consideraciones

- **VPC-SC complementa IAM**, no lo sustituye: IAM define “quién”, VPC-SC define “desde dónde/en qué contexto”.
- Empieza con **Dry run** en entornos reales: observa violaciones y ajusta Access Levels / reglas antes de Enforced.
- Si tu IP cambia (VPN/red), actualiza el Access Level. Para entornos corporativos, considera condiciones adicionales (dispositivo/identidad).
- Si el bloqueo no se reproduce desde Cloud Shell en tu organización, valida desde una red/IP no allowlist y usa los logs con `vpcServiceControlsUniqueId` como evidencia principal.

---

## URLS de referencia

- https://docs.cloud.google.com/vpc-service-controls/docs/create-service-perimeters
- https://docs.cloud.google.com/vpc-service-controls/docs/access-control?hl=es-419
- https://docs.cloud.google.com/vpc-service-controls/docs/audit-logging
- https://docs.cloud.google.com/vpc-service-controls/docs/retrieve-troubleshoot-errors
- https://cloud.google.com/vpc-service-controls/pricing

---

## Task results (obligatorio) — pega esto en `_data/task-results/lab15.yml`

> **IMPORTANTE:** Este bloque alimenta `site.data["task-results"][page.slug].results` y corresponde a los índices usados en cada tarea (0..5).
{: .lab-note .info .compact}

```yaml
results:
  - "✅ Tarea 1 completada: Cloud Shell activo, proyecto verificado, organización validada, APIs (Access Context Manager/Storage/Logging) habilitadas, carpeta `~/labs-gcp-finops/lab15` creada con `scripts/` y `outputs/`, y variables base definidas en `scripts/env.sh`."
  - "✅ Tarea 2 completada: Bucket de Cloud Storage creado con configuración segura (Uniform access + Public access prevention) y objeto `sensitive.txt` cargado y validado mediante `gcloud storage ls/cat`."
  - "✅ Tarea 3 completada: Access Policy identificada/creada y registrada (`POLICY_NAME`); IP pública obtenida y convertida a `/32`; Access Level `lab15_trusted_ip` creado (allowlist IP) y validado por CLI con `levels describe`."
  - "✅ Tarea 4 completada: Perímetro `lab15-perimeter` creado en **Dry run** restringiendo `storage.googleapis.com`; lectura del objeto generó evidencia en logs (audit log policy) con `dryRun=true` y `vpcServiceControlsUniqueId` recuperados por UI/CLI."
  - "✅ Tarea 5 completada: Perímetro cambiado a **Enforced** y bloqueo real validado (403 por VPC-SC); luego se agregó `lab15_trusted_ip` como Access Level permitido y se verificó que la lectura del objeto volvió a funcionar de forma controlada."
  - "✅ Tarea 6 completada: Limpieza opcional realizada (bucket eliminado y configuración de perímetro/access level retirada según políticas de la organización) con verificaciones y evidencias guardadas en `outputs/`."
```
