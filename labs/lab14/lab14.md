---
layout: lab
title: "Práctica 14: Configurar Workload Identity en GKE para evitar uso de claves"
permalink: /lab14/lab14/
images_base: /labs/lab14/img
duration: "45 minutos"
objective:
  - "Crear un clúster **GKE Autopilot**, configurar **Workload Identity Federation for GKE** para que una aplicación en Kubernetes acceda a **Cloud Storage** usando **identidad** (KSA→GSA) **sin llaves JSON**, validando la identidad efectiva dentro del Pod y la lectura de un objeto desde un bucket con permisos de **privilegio mínimo**."
prerequisites:
  - "Proyecto de Google Cloud con **facturación habilitada**."
  - "Permisos para: **Kubernetes Engine Admin**, **Service Account Admin**, **Project IAM Admin** y permisos sobre **Cloud Storage** (crear bucket y administrar IAM del bucket)."
  - "Acceso a **Google Cloud Console** y a **Cloud Shell (gcloud/kubectl)**."
introduction: |
  **Workload Identity Federation for GKE** es el método recomendado para que workloads en GKE se autentiquen contra APIs de Google Cloud **sin** administrar claves de Service Accounts.  
  La idea es simple:
  - Tu **Pod** corre con un **Kubernetes Service Account (KSA)**.
  - Ese KSA se “mapea” a un **Google Service Account (GSA)** con un binding IAM (`roles/iam.workloadIdentityUser`).
  - GKE provee credenciales temporales vía el **metadata server**, y tu app usa esas credenciales para llamar APIs (por ejemplo, leer objetos de Cloud Storage).  

  En esta práctica implementarás el mapeo KSA→GSA y comprobarás que el Pod obtiene identidad y permisos correctos **sin** montar ningún archivo de credenciales.
slug: lab14
lab_number: 14
final_result: >
  Al finalizar, tendrás un clúster GKE Autopilot con un namespace `lab14`, un GSA con permisos mínimos sobre un bucket, un KSA anotado para impersonar ese GSA, y un Pod que valida su identidad vía metadata server y lee un objeto desde Cloud Storage sin usar claves.
notes:
  - "**Costo:** GKE cobra una **cuota de administración por clúster** y además recursos usados (Autopilot cobra por Pods). Cloud Storage cobra por almacenamiento/operaciones. Elimina recursos al finalizar."
  - "No crees ni descargues **keys JSON** para workloads en GKE. Workload Identity reduce riesgo de fuga de credenciales y facilita rotación automática."
  - "Si tu organización usa **Organization Policies** (por ejemplo, restringir creación de buckets o clusters), podría requerir ajustes."
references:
  - text: "Autenticación a Google Cloud APIs desde workloads en GKE (Workload Identity Federation for GKE)"
    url: https://docs.cloud.google.com/kubernetes-engine/docs/how-to/workload-identity?hl=es-419
  - text: "GKE pricing (cluster management fee y costos)"
    url: https://cloud.google.com/kubernetes-engine/pricing
  - text: "Cloud Storage IAM (roles a nivel bucket, ejemplo roles/storage.objectViewer)"
    url: https://docs.cloud.google.com/storage/docs/access-control/iam
prev: /lab13/lab13/
next: /lab15/lab15/
---

---

### Tarea 1. Preparar el entorno y variables del laboratorio

> **Tiempo estimado:** 6 minutos
{: .lab-note .info .compact}

En esta tarea iniciarás Cloud Shell, validarás el proyecto activo, habilitarás APIs necesarias y crearás una carpeta independiente para el laboratorio con variables estándar (cluster, namespace, KSA/GSA y bucket).

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

- {% include step_label.html %} Crea la estructura del laboratorio y entra al directorio.

  > **NOTA:** Una carpeta por práctica mantiene evidencias y manifiestos organizados.
  {: .lab-note .info .compact}

  ```bash
  cd ~
  mkdir -p labs-gcp-engineer/lab14/{manifests,scripts,outputs}
  ```
  ```bash
  cd labs-gcp-engineer/lab14
  ```

  {% include step_image.html %}

- {% include step_label.html %} Habilita APIs necesarias (si tu proyecto es nuevo o no están habilitadas).

  > **NOTA:** Sin estas APIs puedes fallar al crear GKE, service accounts o buckets.
  {: .lab-note .warning .compact}

  ```bash
  gcloud services enable container.googleapis.com iam.googleapis.com storage.googleapis.com
  ```
  ```bash
  gcloud services list --enabled \
    --filter="name:(container.googleapis.com OR iam.googleapis.com OR storage.googleapis.com)" \
    --format="table(name)" | tee outputs/enabled_apis.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Crea `scripts/env.sh` con variables del laboratorio y cárgalas.

  > **NOTA:** Puedes ajustar región si tu organización lo requiere.
  {: .lab-note .info .compact}

  ```bash
  cat > scripts/env.sh <<'EOF'
  export REGION="us-central1"
  export ZONE="us-central1-a"

  export CLUSTER_NAME="lab14-gke"
  export NAMESPACE="lab14"

  # Identidades
  export KSA_NAME="lab14-ksa"
  export GSA_NAME="lab14-gsa"   # nombre (no email)
  # El email se calcula después

  # Bucket: nombre globalmente único (se calcula después)
  EOF
  ```
  ```bash
  source scripts/env.sh
  sed -n '1,220p' scripts/env.sh | tee outputs/env_sh.txt
  ```

- {% include step_label.html %} Calcula un nombre de bucket único (sin asumir) y persístelo en `scripts/env.sh`.

  > **NOTA:** Cloud Storage exige unicidad global; generar el nombre evita colisiones y hace el lab independiente.
  {: .lab-note .important .compact}

  ```bash
  source scripts/env.sh
  # Bucket rules: lowercase, números y guiones; 3-63 chars; global unique
  PROJ_SHORT="$(echo "$PROJECT_ID" | tr '[:upper:]' '[:lower:]' | tr '_' '-' | cut -c1-18)"
  SUFFIX="$(date +%s | tail -c 6)"
  export BUCKET_NAME="lab14-wi-${PROJ_SHORT}-${SUFFIX}"
  ```
  ```bash
  # Persistir al env.sh
  echo "export BUCKET_NAME=\"$BUCKET_NAME\"" >> scripts/env.sh
  echo "BUCKET_NAME=$BUCKET_NAME" | tee outputs/bucket_name.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Valida estructura del laboratorio y guarda evidencia.

  > **NOTA:** Esta evidencia básica ayuda a auditoría y troubleshooting.
  {: .lab-note .info .compact}

  ```bash
  pwd | tee outputs/pwd.txt
  ```
  ```bash
  ls -la | tee outputs/ls_root.txt
  ```
  ```bash
  ls -la manifests scripts outputs | tee outputs/ls_subfolders.txt
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[0] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 2. Crear clúster GKE Autopilot y verificar Workload Identity

> **Tiempo estimado:** 14 minutos
{: .lab-note .info .compact}

En esta tarea crearás un clúster GKE Autopilot. Luego verificarás que el clúster usa el **workload pool** `PROJECT_ID.svc.id.goog` (base de Workload Identity Federation for GKE) y configurarás `kubectl` desde la UI.

#### Tarea 2.1

- {% include step_label.html %} En la consola, ve a **Kubernetes Engine** luego **Clusters**.

  {% include step_image.html %}

- {% include step_label.html %} Haz clic en **Create**

  {% include step_image.html %}

- {% include step_label.html %} Da clic en **Configure** de la sección **Autopilot: Google manages your cluster (Recommended)**.

  > **NOTA:** Autopilot acelera la creación y delega el manejo de nodos a Google (ideal para laboratorios).
  {: .lab-note .info .compact}

  {% include step_image.html %}

- {% include step_label.html %} Configura los siguientes datos:

  - Name: `lab14-gke`
  - Region: **us-central1**

  {% include step_image.html %}

- {% include step_label.html %} Luego clic en **Create**

  > **NOTA:** En Autopilot, Workload Identity Federation for GKE suele estar integrado por defecto; aun así lo verificarás por CLI.
  {: .lab-note .important .compact}

- {% include step_label.html %} Cuando el clúster esté listo (STATUS: RUNNING), conecta `kubectl` desde la UI con **Connect**.

  > **NOTA:** La UI genera el comando `get-credentials` correcto para región/proyecto sin errores de ubicación.
  {: .lab-note .info .compact}

  {% include step_image.html %}

- {% include step_label.html %} Ahora da clic **Run in Cloud Shell**.

  > **NOTA:** La UI genera el comando `gcloud container clusters get-credentials` exacto para tu clúster.
  {: .lab-note .info .compact}

  {% include step_image.html %}
  {% include step_image.html %}

- {% include step_label.html %} Verifica el contexto de `kubectl`.

  > **NOTA:** Confirma conectividad y salud básica del clúster antes de configurar identidades.
  {: .lab-note .info .compact}

  ```bash
  source scripts/env.sh
  kubectl config current-context | tee outputs/kubectl_context.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica por CLI que el clúster tiene configurado el **workload pool**.

  > **NOTA:** El `workloadPool` normalmente es `PROJECT_ID.svc.id.goog` y es el dominio de confianza para mapear KSA→GSA.
  {: .lab-note .important .compact}

  ```bash
  source scripts/env.sh
  gcloud container clusters describe "$CLUSTER_NAME" --region "$REGION" \
    --format="yaml(name,location,autopilot.enabled,workloadIdentityConfig.workloadPool)" \
    | tee outputs/cluster_wi.yaml
  ```
  {% include step_image.html %}

- {% include step_label.html %} Crea el namespace `lab14` y valida que existe.

  > **NOTA:** Separar por namespace facilita filtrar recursos y limpiar el laboratorio al final.
  {: .lab-note .info .compact}

  ```bash
  source scripts/env.sh
  kubectl create ns "$NAMESPACE" 2>/dev/null || true
  ```
  {% include step_image.html %}
  ```bash
  kubectl get ns "$NAMESPACE" -o wide | tee outputs/namespace.txt
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[1] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 3. Crear bucket y Google Service Account con permisos mínimos

> **Tiempo estimado:** 8 minutos
{: .lab-note .info .compact}

En esta tarea crearás un bucket de Cloud Storage y un Google Service Account (GSA) que tendrá **solo** permisos para **leer** objetos del bucket (privilegio mínimo). Subirás un archivo de prueba al bucket con tu usuario (admin) para que el Pod lo lea después.

#### Tarea 3.1

- {% include step_label.html %} En la consola, ve a **Cloud Storage** luego **Buckets**.

  > **NOTA:** Si encuentras que hay mas buckets puedes ignorarlos, solo trabajaras con el que vas a crear.
  {: .lab-note .important .compact}

  {% include step_image.html %}

- {% include step_label.html %} Ahora da clic en **Create**.

  > **NOTA:** El bucket será el recurso objetivo de acceso desde GKE **sin claves**.
  {: .lab-note .info .compact}

  {% include step_image.html %}

- {% include step_label.html %} Crea el bucket con los siguientes datos:

  > **NOTA:** Configuración segura por defecto: evita exposición pública accidental. **El valor del nombre de la imagen NO lo copies**
  {: .lab-note .important .compact}

  - Name: usa el valor de `BUCKET_NAME` (en `outputs/bucket_name.txt`)
  - Clic en **Continue**

  {% include step_image.html %}

  - Location type: **Region**
  - Location: **us-central1**
  - Clic en **Continue**

  {% include step_image.html %}

  - Set a default class: **Standard**
  - Clic en **Continue**

  {% include step_image.html %}

- {% include step_label.html %} Da clic en **Create**

- {% include step_label.html %} Verifica por CLI que el bucket existe y guarda evidencia.

  > **NOTA:** Confirma el bucket real antes de asignar permisos o desplegar Pods.
  {: .lab-note .info .compact}

  ```bash
  source scripts/env.sh
  gcloud storage buckets describe "gs://$BUCKET_NAME" \
    --format="yaml(name,location,uniformBucketLevelAccess)" \
    | tee outputs/bucket_describe.yaml
  ```
  {% include step_image.html %}

- {% include step_label.html %} Crea el Google Service Account (GSA) para el workload.

  > **NOTA:** El GSA representa la identidad IAM “real” que tendrá permisos en Google Cloud.
  {: .lab-note .important .compact}

  ```bash
  export PROJECT_ID="$(gcloud config get-value project)"
  echo "PROJECT_ID=$PROJECT_ID" | tee outputs/project_id.txt
  ```
  ```bash
  source scripts/env.sh
  gcloud iam service-accounts create "$GSA_NAME" \
    --display-name="LAB14 GSA for Workload Identity (no keys)"
  export GSA_EMAIL="${GSA_NAME}@${PROJECT_ID}.iam.gserviceaccount.com"
  ```
  ```bash
  echo "GSA_EMAIL=$GSA_EMAIL" | tee outputs/gsa_email.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Asigna permisos mínimos al GSA sobre el bucket (solo lectura de objetos).

  > **NOTA:** `roles/storage.objectViewer` a nivel bucket permite leer/listar objetos, sin administrar el bucket.
  {: .lab-note .info .compact}

  ```bash
  source scripts/env.sh
  export GSA_EMAIL="$(sed 's/^GSA_EMAIL=//' outputs/gsa_email.txt)"

  gcloud storage buckets add-iam-policy-binding "gs://$BUCKET_NAME" \
    --member="serviceAccount:$GSA_EMAIL" \
    --role="roles/storage.objectViewer"
  ```
  {% include step_image.html %}
  ```bash
  gcloud storage buckets get-iam-policy "gs://$BUCKET_NAME" \
    --format="yaml(bindings)" | tee outputs/bucket_iam.yaml
  ```
  {% include step_image.html %}

- {% include step_label.html %} Sube un archivo de prueba al bucket con tu identidad (admin) y valida que existe.

  > **NOTA:** Tu Pod solo **leerá**. Aquí tú (admin) cargas el archivo para comprobar acceso sin llaves.
  {: .lab-note .info .compact}

  ```bash
  source scripts/env.sh
  echo "LAB14 Workload Identity OK - $(date -u +%Y-%m-%dT%H:%M:%SZ)" > outputs/hello.txt
  ```
  {% include step_image.html %}
  ```bash
  gcloud storage cp outputs/hello.txt "gs://$BUCKET_NAME/hello.txt"
  ```
  {% include step_image.html %}
  ```bash
  gcloud storage ls "gs://$BUCKET_NAME/" | tee outputs/bucket_ls.txt
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[2] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 4. Configurar mapeo KSA→GSA y probar acceso desde un Pod sin claves

> **Tiempo estimado:** 13 minutos
{: .lab-note .info .compact}

En esta tarea crearás un Kubernetes Service Account (KSA), le darás permiso de impersonar al GSA (binding IAM), anotarás el KSA, y desplegarás un Pod que:

- 1) Consulta el metadata server para ver **qué GSA está usando**  
- 2) Obtiene un token temporal  
- 3) Lee `hello.txt` desde el bucket, **sin** usar ninguna key.

#### Tarea 4.1

- {% include step_label.html %} Crea el KSA en el namespace y verifica su existencia.

  > **NOTA:** El KSA es la identidad “dentro” de Kubernetes; será el puente hacia el GSA.
  {: .lab-note .info .compact}

  ```bash
  source scripts/env.sh
  kubectl -n "$NAMESPACE" create serviceaccount "$KSA_NAME" 2>/dev/null || true
  ```
  {% include step_image.html %}
  ```bash
  kubectl -n "$NAMESPACE" get sa "$KSA_NAME" -o yaml | tee outputs/ksa.yaml
  ```
  {% include step_image.html %}

- {% include step_label.html %} Otorga al KSA permiso para impersonar el GSA (binding `roles/iam.workloadIdentityUser`).

  > **NOTA:** Este binding es el “permiso de suplantación”: el KSA puede actuar como el GSA sin llaves.
  {: .lab-note .important .compact}

  ```bash
  source scripts/env.sh
  export GSA_EMAIL="$(sed 's/^GSA_EMAIL=//' outputs/gsa_email.txt)"

  gcloud iam service-accounts add-iam-policy-binding "$GSA_EMAIL" \
    --role="roles/iam.workloadIdentityUser" \
    --member="serviceAccount:${PROJECT_ID}.svc.id.goog[${NAMESPACE}/${KSA_NAME}]"
  ```
  {% include step_image.html %}
  ```bash
  gcloud iam service-accounts get-iam-policy "$GSA_EMAIL" \
    --format="yaml(bindings)" | tee outputs/gsa_iam.yaml
  ```
  {% include step_image.html %}

- {% include step_label.html %} Anota el KSA para indicar qué GSA debe usar (`iam.gke.io/gcp-service-account`).

  > **NOTA:** La anotación conecta el KSA con el GSA específico (como “usar esta identidad en Google Cloud”).
  {: .lab-note .info .compact}

  ```bash
  source scripts/env.sh
  export GSA_EMAIL="$(sed 's/^GSA_EMAIL=//' outputs/gsa_email.txt)"

  kubectl -n "$NAMESPACE" annotate serviceaccount "$KSA_NAME" \
    "iam.gke.io/gcp-service-account=$GSA_EMAIL" --overwrite
  ```
  {% include step_image.html %}
  ```bash
  kubectl -n "$NAMESPACE" get sa "$KSA_NAME" -o yaml | tee outputs/ksa_annotated.yaml
  ```
  {% include step_image.html %}

- {% include step_label.html %} Crea el manifiesto del Pod de prueba `manifests/wi-test-pod.yaml`.

  > **NOTA:** Este Pod valida “quién soy” y luego consume Cloud Storage usando credenciales temporales (ADC) **sin** `GOOGLE_APPLICATION_CREDENTIALS`.
  {: .lab-note .important .compact}

  ```bash
  source scripts/env.sh
  export GSA_EMAIL="$(sed 's/^GSA_EMAIL=//' outputs/gsa_email.txt)"

  cat > manifests/wi-test-pod.yaml <<EOF
  apiVersion: v1
  kind: Pod
  metadata:
    name: wi-test
    namespace: ${NAMESPACE}
    labels:
      app: wi-test
  spec:
    serviceAccountName: ${KSA_NAME}
    restartPolicy: Never
    containers:
    - name: tester
      image: gcr.io/google.com/cloudsdktool/cloud-sdk:slim
      env:
      - name: PROJECT_ID
        value: "${PROJECT_ID}"
      - name: BUCKET_NAME
        value: "${BUCKET_NAME}"
      - name: EXPECTED_GSA
        value: "${GSA_EMAIL}"
      command: ["/bin/bash","-lc"]
      args:
        - |
          set -euo pipefail

          echo "== 0) Confirmar que NO hay keys montadas =="
          echo "GOOGLE_APPLICATION_CREDENTIALS=\${GOOGLE_APPLICATION_CREDENTIALS:-<vacío>}"
          test "\${GOOGLE_APPLICATION_CREDENTIALS:-}" = ""
          ls -la /var/secrets 2>/dev/null || true

          echo "== 1) Validar identidad efectiva (metadata server) =="
          echo "Esperado GSA: \${EXPECTED_GSA}"
          EMAIL="\$(curl -s -H 'Metadata-Flavor: Google' \
            http://metadata.google.internal/computeMetadata/v1/instance/service-accounts/default/email)"
          echo "GSA efectivo: \${EMAIL}"
          test "\${EMAIL}" = "\${EXPECTED_GSA}"

          echo "== 2) Obtener token temporal (sin llaves) =="
          curl -s -H 'Metadata-Flavor: Google' \
            http://metadata.google.internal/computeMetadata/v1/instance/service-accounts/default/token \
            | head -c 220; echo

          echo "== 3) Leer objeto desde Cloud Storage (ADC) =="
          gcloud config set core/project "\${PROJECT_ID}" >/dev/null
          gcloud storage ls "gs://\${BUCKET_NAME}/"
          gcloud storage cat "gs://\${BUCKET_NAME}/hello.txt" | head -n 5

          echo "OK: acceso a Cloud Storage sin llaves (Workload Identity)"
  EOF

  sed -n '1,260p' manifests/wi-test-pod.yaml | tee outputs/wi_test_manifest.txt
  ```

- {% include step_label.html %} Aplica el manifiesto, espera la ejecución y revisa logs del Pod.

  > **NOTA:** El resultado correcto es que los logs muestren el email del GSA, token y el contenido de `hello.txt`.
  {: .lab-note .info .compact}

  ```bash
  source scripts/env.sh
  kubectl apply -f manifests/wi-test-pod.yaml
  ```
  {% include step_image.html %}
  ```bash
  kubectl -n "$NAMESPACE" get pods -o wide | tee outputs/pods_list.txt
  ```
  {% include step_image.html %}
  ```bash
  # Espera que termine (Succeeded) o al menos que el contenedor ejecute y escriba logs
  kubectl -n "$NAMESPACE" wait --for=jsonpath='{.status.phase}'=Succeeded pod/wi-test --timeout=240s \
    || kubectl -n "$NAMESPACE" get pod/wi-test -o wide
  ```
  {% include step_image.html %}
  ```bash
  kubectl -n "$NAMESPACE" logs pod/wi-test | tee outputs/wi_test_logs.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl -n "$NAMESPACE" get pod/wi-test -o jsonpath='{.status.phase}{"\n"}' | tee outputs/wi_test_phase.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} **Prueba negativa:** Pod SIN mapeo (debe fallar al leer el objeto).

  > **NOTA:** Si esta prueba **no** falla, es señal de que existe otro rol amplio heredado (por ejemplo, al “default” o por política organizacional). En ese caso, úsala como hallazgo de seguridad: “el entorno no está en privilegio mínimo”.
  {: .lab-note .warning .compact}

  ```bash
  source scripts/env.sh
  cat > manifests/no-wi-pod.yaml <<EOF
  apiVersion: v1
  kind: Pod
  metadata:
    name: no-wi
    namespace: ${NAMESPACE}
  spec:
    serviceAccountName: default
    restartPolicy: Never
    containers:
    - name: tester
      image: gcr.io/google.com/cloudsdktool/cloud-sdk:slim
      env:
      - name: PROJECT_ID
        value: "${PROJECT_ID}"
      - name: BUCKET_NAME
        value: "${BUCKET_NAME}"
      command: ["/bin/bash","-lc"]
      args:
        - |
          set -euo pipefail
          gcloud config set core/project "\${PROJECT_ID}" >/dev/null
          echo "Intento leer objeto sin mapeo KSA->GSA (debe fallar con 403)"
          gcloud storage cat "gs://\${BUCKET_NAME}/hello.txt"
  EOF
  ```
  ```bash
  kubectl apply -f manifests/no-wi-pod.yaml
  ```
  {% include step_image.html %}
  ```bash
  kubectl -n "$NAMESPACE" logs pod/no-wi --follow --tail=80 | tee outputs/no_wi_logs.txt || true
  ```
  {% include step_image.html %}
  ```bash
  kubectl -n "$NAMESPACE" get pod/no-wi -o jsonpath='{.status.phase}{"\n"}' | tee outputs/no_wi_phase.txt
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[3] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 5. Limpieza de recursos

> **Tiempo estimado:** 4 minutos
{: .lab-note .info .compact}

En esta tarea eliminarás recursos para evitar costos residuales: Pods/namespace, bucket, service account y clúster. Harás verificaciones rápidas por CLI.

#### Tarea 5.1

- {% include step_label.html %} Elimina recursos de Kubernetes (namespace completo).

  > **NOTA:** Borrar el namespace elimina pods y service accounts del laboratorio en un solo paso.
  {: .lab-note .info .compact}

  ```bash
  source scripts/env.sh
  kubectl delete ns "$NAMESPACE" --ignore-not-found
  ```
  ```bash
  kubectl get ns | grep -n "$NAMESPACE" || echo "OK: namespace eliminado" | tee outputs/cleanup_ns.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Elimina el bucket.

  > **NOTA:** Cloud Storage cobra por almacenamiento/operaciones; elimina el bucket para evitar cargos residuales.
  {: .lab-note .warning .compact}

  ```bash
  source scripts/env.sh
  gcloud storage rm -r "gs://$BUCKET_NAME" || true
  ```
  ```bash
  gcloud storage buckets describe "gs://$BUCKET_NAME" >/dev/null 2>&1 || echo "OK: bucket eliminado" | tee outputs/cleanup_bucket.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Elimina el Google Service Account (GSA).

  > **NOTA:** Evitas identidades huérfanas y reduces riesgo de asignaciones accidentales en el futuro.
  {: .lab-note .info .compact}

  ```bash
  source scripts/env.sh
  export GSA_EMAIL="$(sed 's/^GSA_EMAIL=//' outputs/gsa_email.txt)"
  gcloud iam service-accounts delete "$GSA_EMAIL" --quiet || true
  ```
  ```bash
  gcloud iam service-accounts describe "$GSA_EMAIL" >/dev/null 2>&1 || echo "OK: GSA eliminado" | tee outputs/cleanup_gsa.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Elimina el clúster de Kubernetes.

  > **NOTA:** En GKE, el clúster es el costo principal (cuota por clúster y recursos Autopilot).
  {: .lab-note .warning .compact}

  ```bash
  source scripts/env.sh
  gcloud container clusters delete "$CLUSTER_NAME" --region "$REGION" --quiet || true
  ```

- {% include step_label.html %} Verifica que el cluster este correctamente eliminado.

  ```bash
  source scripts/env.sh

  # Verificación: debe FALLAR y entonces imprimimos OK
  gcloud container clusters describe "$CLUSTER_NAME" --region "$REGION" >/dev/null 2>&1 \
    && echo "WARN: el clúster aún existe" \
    || echo "OK: el clúster ya no existe" | tee outputs/cluster_deleted_check.txt
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[4] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}