---
layout: lab
title: "Práctica 8: Desplegar app contenedorizada en GKE y exponer con Ingress"
permalink: /lab8/lab8/
images_base: /labs/lab8/img
duration: "60 minutos"
objective:
  - "Desplegar una **aplicación contenedorizada** en **Google Kubernetes Engine (GKE)** usando **Artifact Registry** como repositorio de imágenes, crear **Deployment + Service**, y exponerla a internet con un **Ingress (GCE)** que aprovisiona un **HTTP(S) Load Balancer** global, validando el flujo end-to-end con evidencias por **UI** y comandos en **Cloud Shell**."
prerequisites:
  - "Proyecto de Google Cloud con permisos para **GKE**, **Compute Engine**, **VPC**, **Artifact Registry**, **Cloud Build**, **Load Balancing** e **IAM**."
  - "Acceso a **Google Cloud Console** y **Cloud Shell**."
  - "Navegador con sesión activa en Google."
introduction: |
  En GKE, una aplicación contenedorizada típicamente se despliega con un **Deployment** (réplicas, rollout) y se publica dentro del clúster con un **Service**. Para exponerla a internet con una IP pública, se suele usar **Ingress** (controlador **GCE Ingress**), que crea automáticamente un **HTTP(S) Load Balancer** y conecta tu tráfico a los backends del clúster.

  En esta práctica crearás:
  - Un repositorio **Artifact Registry (Docker)**,
  - Una imagen de contenedor con una página web con estilo (Nginx),
  - Un clúster **GKE Autopilot** (enfocado en rapidez y mínima administración),
  - Un **Deployment + Service (NodePort)**,
  - Un **Ingress** que expone la app con una **IP global** (opcionalmente estática),
  - Evidencias operativas (pods listos, service, ingress con IP, respuesta HTTP).
slug: lab8
lab_number: 8
final_result: >
  Al finalizar, tendrás una imagen publicada en Artifact Registry, una app desplegada en GKE (Deployment+Service) y expuesta con Ingress (GCE) mediante un HTTP(S) Load Balancer, accediendo desde internet por una IP pública y validando salud y disponibilidad con comandos y UI.
notes:
  - "**Costos:** GKE Autopilot cobra por recursos consumidos (pods) y el **HTTP(S) Load Balancer** creado por Ingress tiene costo. Artifact Registry cobra por almacenamiento. Elimina recursos al terminar."
  - "El aprovisionamiento del Load Balancer y la asignación de IP pueden tardar varios minutos."
  - "Para HTTPS con dominio y certificado administrado, revisa la Práctica 7; aquí nos enfocamos en **Ingress HTTP** (y el LB que se crea)."
references:
  - text: "Ingress en GKE (GCE Ingress) y exposición de servicios"
    url: https://cloud.google.com/kubernetes-engine/docs/concepts/ingress
  - text: "Configurar HTTP(S) Load Balancing con Ingress en GKE"
    url: https://cloud.google.com/kubernetes-engine/docs/how-to/ingress-features
  - text: "Artifact Registry (repositorios Docker)"
    url: https://cloud.google.com/artifact-registry/docs/docker/quickstart
  - text: "Cloud Build: construir y subir imágenes"
    url: https://cloud.google.com/build/docs/building/build-containers
  - text: "GKE Autopilot: visión general"
    url: https://cloud.google.com/kubernetes-engine/docs/concepts/autopilot-overview
  - text: "Precios GKE"
    url: https://cloud.google.com/kubernetes-engine/pricing
  - text: "Precios Cloud Load Balancing"
    url: https://cloud.google.com/load-balancing/pricing
  - text: "Precios Artifact Registry"
    url: https://cloud.google.com/artifact-registry/pricing
prev: /lab7/lab7/
next: /lab9/lab9/
---

---

## Instrucciones generales

- Esta práctica es **mayormente por interfaz gráfica (UI)**:
  - habilitar APIs, crear repositorio, crear clúster GKE, revisar Workloads/Services/Ingress.
- Usaremos **Cloud Shell** cuando se requiera:
  - crear estructura de carpetas del lab,
  - construir y subir la imagen a Artifact Registry con Cloud Build,
  - aplicar manifiestos Kubernetes (`kubectl`),
  - validar Ingress y probar la URL/IP con `curl`.

Carpeta de trabajo (Cloud Shell):
- `~/labs-gcp-engineer/lab08/` con `scripts/`, `k8s/` y `outputs/`.

---

### Tarea 1. Preparar Cloud Shell y variables del laboratorio (Tiempo estimado: 5 min)

En esta tarea crearás la estructura del lab y definirás variables alineadas a **Práctica 8**.

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

- {% include step_label.html %} Crea la carpeta del laboratorio 8.

  > **NOTA:** `app/` contiene el código del contenedor; `k8s/` manifiestos; `outputs/` evidencias.
  {: .lab-note .info .compact}

  ```bash
  cd ~
  mkdir -p labs-gcp-engineer/lab08/{scripts,k8s,outputs,app}
  ```
  ```bash
  cd labs-gcp-engineer/lab08
  ```
  {% include step_image.html %}


- {% include step_label.html %} Crea `scripts/env.sh` con nombres alineados a lab08.

  > **NOTA:** Estandarizar nombres evita errores y facilita revisar el lab.
  {: .lab-note .important .compact}

  ```bash
  cat > scripts/env.sh <<'EOF'
  # Región/zona (GKE)
  export REGION="us-central1"

  # Artifact Registry
  export AR_LOCATION="us-central1"
  export AR_REPO="lab08-repo"
  export IMAGE_NAME="lab08-web"
  export IMAGE_TAG="v1"

  # GKE
  export CLUSTER_NAME="lab08-gke"
  export NAMESPACE="lab08"
  export DEPLOY_NAME="lab08-web"
  export SVC_NAME="lab08-web-svc"
  export INGRESS_NAME="lab08-web-ing"
  export INGRESS_IP_NAME="lab08-ingress-ip"
  EOF
  ```
  ```bash
  source scripts/env.sh
  ```

- {% include step_label.html %} Configura región por defecto para `gcloud` y guarda evidencia.

  > **NOTA:** Reduce parámetros repetidos.
  {: .lab-note .info .compact}

  ```bash
  gcloud config set compute/region "$REGION"
  ```
  ```bash
  gcloud config list --format="yaml(core.project,compute.region,compute.zone)" | tee outputs/gcloud_config.yaml
  ```
  {% include step_image.html %}


{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[0] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 2. Habilitar APIs y crear repositorio Artifact Registry (Tiempo estimado: 10 min)

En esta tarea habilitarás APIs y crearás un repositorio Docker para almacenar la imagen.

#### Tarea 2.1 (UI) — Habilitar APIs necesarias

- {% include step_label.html %} **En Cloud Shell** ejecuta el siguiente comando para habilitar las APIs faltantes.

  > **NOTA:** Sin estas APIs, el clúster, el repositorio o el build fallarán.
  {: .lab-note .important .compact}

  - **Kubernetes Engine API**
  - **Artifact Registry API**
  - **Cloud Build API**
  - **Compute Engine API** (si no estaba)

  ```bash
  gcloud services enable \
    container.googleapis.com \
    artifactregistry.googleapis.com \
    cloudbuild.googleapis.com \
    compute.googleapis.com
  ```
  {% include step_image.html %}

- {% include step_label.html %} En Cloud Shell, confirma APIs habilitadas y guarda evidencia.

  > **NOTA:** Evidencia rápida de pre-requisitos habilitados.
  {: .lab-note .info .compact}

  ```bash
  gcloud services list --enabled \
    --filter="name:container.googleapis.com OR name:artifactregistry.googleapis.com OR name:cloudbuild.googleapis.com OR name:compute.googleapis.com" \
    --format="table(name,title)" | tee outputs/enabled_apis.txt
  ```
  {% include step_image.html %}


#### Tarea 2.2 (UI) — Crear repositorio Docker en Artifact Registry

- {% include step_label.html %} En consola: en el buscador escribe **Artifact Registry** y da clic en el servicio.

  {% include step_image.html %}

- {% include step_label.html %} Clic en **Create repository**

  {% include step_image.html %}

- {% include step_label.html %} Configura los siguientes datos:

  > **NOTA:** Artifact Registry es el repositorio recomendado para imágenes en Google Cloud.
  {: .lab-note .info .compact}
  
  - Name: `lab08-repo`
  - Format: **Docker**
  - Mode: **Standard**
  - Location type: **Region**
  - Region: `us-central1`

  {% include step_image.html %}

- {% include step_label.html %} Clic en **Create**

- {% include step_label.html %} En Cloud Shell, valida el repositorio (evidencia).

  > **NOTA:** Verifica nombre, formato Docker y región.
  {: .lab-note .important .compact}

  ```bash
  source scripts/env.sh
  gcloud artifacts repositories describe "$AR_REPO" --location "$AR_LOCATION" \
    --format="yaml(name,format,location,state)" | tee outputs/ar_repo.yaml
  ```
  {% include step_image.html %}


{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[1] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 3. Crear app contenedorizada y subir imagen a Artifact Registry (Tiempo estimado: 10 min)

En esta tarea crearás una app web simple (Nginx + página con estilo), construirás la imagen con Cloud Build y la subirás a Artifact Registry.

#### Tarea 3.1 — Crear archivos de la app (Dockerfile + HTML)

- {% include step_label.html %} Crea el archivo `app/index.html` con estilo.

  > **NOTA:** Página con estilo para validar visualmente que el tráfico llega al backend.
  {: .lab-note .info .compact}

  ```bash
  cd ~/labs-gcp-engineer/lab08
  cat > app/index.html <<'EOF'
  <!doctype html>
  <html>
    <head>
      <meta charset="utf-8"/>
      <title>Lab08 - GKE + Ingress</title>
      <style>
        body { font-family: system-ui, -apple-system, Segoe UI, Roboto, Arial; background: #0b1220; color: #e6eefc; padding: 28px; }
        .wrap { max-width: 980px; margin: 0 auto; }
        .card { background: rgba(255,255,255,0.06); border: 1px solid rgba(255,255,255,0.12); border-radius: 16px; padding: 18px; box-shadow: 0 10px 28px rgba(0,0,0,0.35); }
        h1 { margin: 0 0 10px; font-size: 26px; }
        .pill { display:inline-block; padding: 6px 10px; border-radius: 999px; background: rgba(34,197,94,0.18); border: 1px solid rgba(34,197,94,0.35); }
        .muted { opacity: .85; }
        code { background: rgba(255,255,255,0.08); padding: 2px 6px; border-radius: 6px; }
        .grid { display:grid; grid-template-columns: 1fr 1fr; gap: 12px; margin-top: 12px; }
        .box { border: 1px solid rgba(255,255,255,0.12); border-radius: 14px; padding: 12px; background: rgba(255,255,255,0.04); }
        @media (max-width: 780px){ .grid { grid-template-columns: 1fr; } }
      </style>
    </head>
    <body>
      <div class="wrap">
        <div class="card">
          <h1>Lab08: App contenedorizada en GKE</h1>
          <p class="pill">Nginx OK</p>
          <p class="muted">Esta página se sirve desde un contenedor y se expone con <b>Ingress</b>.</p>

          <div class="grid">
            <div class="box">
              <b>Componentes</b>
              <ul>
                <li>Artifact Registry (imagen)</li>
                <li>GKE (Deployment + Service)</li>
                <li>Ingress (GCE) → HTTP(S) Load Balancer</li>
              </ul>
            </div>
            <div class="box">
              <b>Tips</b>
              <ul>
                <li>Revisa el estado de Ingress con <code>kubectl describe ingress</code></li>
                <li>Si no hay IP, espera unos minutos y vuelve a consultar</li>
              </ul>
            </div>
          </div>

          <p class="muted" style="margin-top:12px;">
            Endpoint de salud: <code>/healthz</code>
          </p>
        </div>
      </div>
    </body>
  </html>
  EOF
  ```

- {% include step_label.html %} Crea el archivo `app/Dockerfile` (Nginx + endpoint `/healthz`).

  > **NOTA:** `/healthz` sirve para health checks y validaciones rápidas.
  {: .lab-note .info .compact}

  ```bash
  cat > app/Dockerfile <<'EOF'
  FROM nginx:alpine
  COPY index.html /usr/share/nginx/html/index.html
  RUN sh -c "echo ok > /usr/share/nginx/html/healthz"
  EOF
  ```

- {% include step_label.html %} Verifica los archivos creados.

  ```bash
  ls -la app | tee outputs/app_ls.txt
  sed -n '1,40p' app/Dockerfile | tee outputs/dockerfile_preview.txt
  ```
  {% include step_image.html %}

#### Tarea 3.2 — Build + Push con Cloud Build

- {% include step_label.html %} Carga las variables, arma la URI de la imagen y guárdala.

  > **NOTA:** Esta URI se usará en el Deployment de Kubernetes.
  {: .lab-note .important .compact}

  ```bash
  source scripts/env.sh

  PROJECT_ID="$(gcloud config get-value project)"
  PROJECT_NUMBER="$(gcloud projects describe "$PROJECT_ID" --format='value(projectNumber)')"
  IMAGE_URI="${AR_LOCATION}-docker.pkg.dev/${PROJECT_ID}/${AR_REPO}/${IMAGE_NAME}:${IMAGE_TAG}"

  echo "PROJECT_ID=$PROJECT_ID"
  echo "PROJECT_NUMBER=$PROJECT_NUMBER"
  echo "IMAGE_URI=$IMAGE_URI" | tee outputs/image_uri.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Asegura que la API este activada.

  ```bash
  gcloud services enable artifactregistry.googleapis.com cloudbuild.googleapis.com
  ```
  {% include step_image.html %} 

- {% include step_label.html %} Asegura que el repo exista

  ```bash
  gcloud artifacts repositories describe "$AR_REPO" --location "$AR_LOCATION"
  ```
  {% include step_image.html %}

- {% include step_label.html %} Permisos: dale Writer a los 2 posibles SAs que Cloud Build puede usar
  ```bash
  CB_LEGACY="${PROJECT_NUMBER}@cloudbuild.gserviceaccount.com"
  CB_COMPUTE="${PROJECT_NUMBER}-compute@developer.gserviceaccount.com"
  ```

- {% include step_label.html %} Agrega los permisos al service account.

  ```bash
  gcloud artifacts repositories add-iam-policy-binding "$AR_REPO" \
    --location "$AR_LOCATION" \
    --member="serviceAccount:${CB_LEGACY}" \
    --role="roles/artifactregistry.writer" --quiet
  ```
  ```bash
  gcloud artifacts repositories add-iam-policy-binding "$AR_REPO" \
    --location "$AR_LOCATION" \
    --member="serviceAccount:${CB_COMPUTE}" \
    --role="roles/artifactregistry.writer" --quiet
  ```
  {% include step_image.html %} 

- {% include step_label.html %} Ejecuta el **Build + Push** desde Cloud Build.

  ```bash
  gcloud builds submit --tag "$IMAGE_URI" app | tee outputs/cloudbuild_submit_retry.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Lista las imágenes en el repo

  ```bash
  gcloud artifacts docker images list "${AR_LOCATION}-docker.pkg.dev/${PROJECT_ID}/${AR_REPO}" \
    --include-tags \
    --format="table(IMAGE,TAGS,DIGEST,CREATE_TIME)" | tee outputs/ar_images_after.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verifica que la imagen este guardada en **Artifact Registry** UI

  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[2] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 4. Crear clúster GKE (Autopilot) y conectar kubectl (Tiempo estimado: 20 min)

En esta tarea crearás un clúster GKE Autopilot y conectarás `kubectl` desde Cloud Shell.

#### Tarea 4.1 (UI) — Crear clúster Autopilot

- {% include step_label.html %} En consola: **Kubernetes Engine** luego **Clusters**.

  {% include step_image.html %}

- {% include step_label.html %} Clic en **Create**

  {% include step_image.html %}

- {% include step_label.html %} Configura los siguientes datos:

  > **NOTA:** Autopilot reduce administración de nodos y acelera el despliegue.
  {: .lab-note .info .compact}

  - Cluster name: `lab08-gke`
  - Region: **us-central1**

  {% include step_image.html %}

- {% include step_label.html %} Clic en **Create**.

  > **NOTA:** La creación puede tardar varios minutos; no avances a despliegues hasta que el estado sea **RUNNING**.
  {: .lab-note .warning .compact}

  {% include step_image.html %}

#### Tarea 4.2 (Cloud Shell) — Obtener credenciales y validar acceso

- {% include step_label.html %} Primero **Dentro de Cloud Shell** crea un Cloud NAT temporal para la descarga de dependencias de la practica, ejecuta todo el codigo sisguiente

  ```bash
  NET_FULL="$(gcloud container clusters describe "$CLUSTER_NAME" --region "$REGION" --format="value(network)")"
  NETWORK_NAME="${NET_FULL##*/}"

  ROUTER_NAME="lab08-router"
  NAT_NAME="lab08-nat"

  gcloud compute routers create "$ROUTER_NAME" \
    --region "$REGION" \
    --network "$NETWORK_NAME" \
    2>/dev/null || true

  gcloud compute routers nats create "$NAT_NAME" \
    --router "$ROUTER_NAME" \
    --region "$REGION" \
    --nat-all-subnet-ip-ranges \
    --auto-allocate-nat-external-ips \
    --enable-logging \
    2>/dev/null || \
  gcloud compute routers nats update "$NAT_NAME" \
    --router "$ROUTER_NAME" \
    --region "$REGION" \
    --nat-all-subnet-ip-ranges \
    --auto-allocate-nat-external-ips \
    --enable-logging
  ```

- {% include step_label.html %} Cuando el clúster esté **RUNNING**, obtén credenciales en Cloud Shell.

  > **NOTA:** Esto configura `kubectl` para apuntar al clúster.
  {: .lab-note .important .compact}

  ```bash
  source scripts/env.sh
  gcloud container clusters get-credentials "$CLUSTER_NAME" --region "$REGION"
  ```
  {% include step_image.html %}

- {% include step_label.html %} Valida contexto actual y acceso al API.

  > **NOTA:** Si `cluster-info` responde sin errores, `kubectl` está listo.
  {: .lab-note .info .compact}

  ```bash
  kubectl config current-context | tee outputs/kubectl_context.txt
  kubectl cluster-info | tee outputs/cluster_info.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Valida nodos/pods del sistema.

  > **NOTA:** En Autopilot los nodos se gestionan automáticamente; aun así puedes observar estado general.
  {: .lab-note .warning .compact}

  ```bash
  kubectl get nodes -o wide | tee outputs/nodes.txt
  kubectl -n kube-system get pods | head -n 30 | tee outputs/kube_system_pods_head.txt
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[3] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 5. Desplegar la app en GKE (Deployment + Service) (Tiempo estimado: 10 min)

En esta tarea crearás un namespace, desplegarás la app y la publicarás internamente con un Service (NodePort) requerido por Ingress.

#### Tarea 5.1 — Crear manifiesto Kubernetes

- {% include step_label.html %} Crea `k8s/app.yaml` (Namespace + Deployment + Service).

  > **NOTA:** `NodePort` permite que el controlador de Ingress enrute hacia tu servicio.
  {: .lab-note .info .compact}

  ```bash
  source scripts/env.sh
  export PROJECT_ID="$(gcloud config get-value project)"
  export IMAGE_URI="${AR_LOCATION}-docker.pkg.dev/${PROJECT_ID}/${AR_REPO}/${IMAGE_NAME}:${IMAGE_TAG}"

  cat > k8s/app.yaml <<EOF
  apiVersion: v1
  kind: Namespace
  metadata:
    name: ${NAMESPACE}
  ---
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: ${DEPLOY_NAME}
    namespace: ${NAMESPACE}
    labels:
      app: ${DEPLOY_NAME}
  spec:
    replicas: 2
    selector:
      matchLabels:
        app: ${DEPLOY_NAME}
    template:
      metadata:
        labels:
          app: ${DEPLOY_NAME}
      spec:
        containers:
        - name: web
          image: ${IMAGE_URI}
          ports:
          - containerPort: 80
          readinessProbe:
            httpGet:
              path: /healthz
              port: 80
            initialDelaySeconds: 2
            periodSeconds: 5
          livenessProbe:
            httpGet:
              path: /healthz
              port: 80
            initialDelaySeconds: 10
            periodSeconds: 10
          resources:
            requests:
              cpu: "50m"
              memory: "64Mi"
            limits:
              cpu: "250m"
              memory: "128Mi"
  ---
  apiVersion: v1
  kind: Service
  metadata:
    name: ${SVC_NAME}
    namespace: ${NAMESPACE}
  spec:
    type: NodePort
    selector:
      app: ${DEPLOY_NAME}
    ports:
    - name: http
      port: 80
      targetPort: 80
  EOF
  ```

- {% include step_label.html %} Valida el YAML en seco (evidencia).

  > **NOTA:** Evita errores de sintaxis antes de aplicar.
  {: .lab-note .important .compact}

  ```bash
  kubectl apply --dry-run=client -f k8s/app.yaml | tee outputs/app_dryrun.txt
  ```
  {% include step_image.html %}


#### Tarea 5.2 — Aplicar manifiestos y validar

- {% include step_label.html %} Aplica el despliegue.

  ```bash
  kubectl apply -f k8s/app.yaml | tee outputs/app_apply.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Espera el rollout y revisa pods.

  > **NOTA:** Debes ver 2 pods READY antes de crear el Ingress.
  {: .lab-note .important .compact}

  ```bash
  source scripts/env.sh
  kubectl -n "$NAMESPACE" rollout status deploy/"$DEPLOY_NAME" | tee outputs/rollout_status.txt
  ```
  {% include step_image.html %}
  ```bash
  kubectl -n "$NAMESPACE" get pods -o wide | tee outputs/pods.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Revisa el Service y su NodePort (evidencia).

  > **NOTA:** El NodePort será usado internamente por el LB del Ingress.
  {: .lab-note .info .compact}

  ```bash
  kubectl -n "$NAMESPACE" get svc "$SVC_NAME" -o wide | tee outputs/service.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} (UI) En consola: **Kubernetes Engine** luego en **Workloads** y confirma que `lab08-web` está **OK**.

  > **NOTA:** UI sirve como evidencia visual para revisión del instructor.
  {: .lab-note .info .compact}

  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[4] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 6. Exponer la app con Ingress (GCE) y probar acceso público (Tiempo estimado: 12 min)

En esta tarea crearás una IP global estática (recomendado) y un Ingress que aprovisiona un HTTP(S) Load Balancer.

#### Tarea 6.1 (UI) — Reservar IP global estática (recomendado)

- {% include step_label.html %} En consola: **VPC network** luego **IP addresses** → Reserve static address**.

  > **NOTA:** Evita que tu endpoint cambie si el Ingress se recrea.
  {: .lab-note .info .compact}

  {% include step_image.html %}

- {% include step_label.html %} Da clic en **Reserve external**.

  {% include step_image.html %}

- {% include step_label.html %} Configura los siguientes datos:

  - Name: `lab08-ingress-ip`
  - Type: **Global**

  {% include step_image.html %}

- {% include step_label.html %} Clic en **Reserve**

- {% include step_label.html %} Obtén la IP por CLI.

  ```bash
  source scripts/env.sh
  export INGRESS_IP="$(gcloud compute addresses describe "$INGRESS_IP_NAME" --global --format='value(address)')"
  echo "INGRESS_IP=$INGRESS_IP" | tee outputs/ingress_ip.txt
  ```
  {% include step_image.html %}

#### Tarea 6.2 — Crear y aplicar Ingress

- {% include step_label.html %} Crea el archivo `k8s/ingress.yaml` (GCE external Ingress + IP estática).

  > **NOTA:** `ingressClassName: gce` indica Ingress externo administrado por GKE (GCE Ingress).
  {: .lab-note .info .compact}

  ```bash
  source scripts/env.sh
  cat > k8s/ingress.yaml <<EOF
  apiVersion: networking.k8s.io/v1
  kind: Ingress
  metadata:
    name: ${INGRESS_NAME}
    namespace: ${NAMESPACE}
    annotations:
      kubernetes.io/ingress.class: "gce"
      kubernetes.io/ingress.global-static-ip-name: ${INGRESS_IP_NAME}
  spec:
    ingressClassName: gce
    defaultBackend:
      service:
        name: ${SVC_NAME}
        port:
          number: 80
  EOF
  ```

- {% include step_label.html %} Valida el archivo YAML.

  ```bash
  kubectl apply --dry-run=client -f k8s/ingress.yaml | tee outputs/ingress_dryrun.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Anota el Service para NEG

  ```bash
  source scripts/env.sh
  kubectl -n "$NAMESPACE" annotate svc "$SVC_NAME" \
    cloud.google.com/neg='{"ingress": true}' --overwrite \
    | tee outputs/svc_neg_annotate.txt
  ```

- {% include step_label.html %} Habilita la funcion de load balancing para GKE

  ```bash
  source scripts/env.sh
  gcloud container clusters update "$CLUSTER_NAME" --region "$REGION" \
    --update-addons=HttpLoadBalancing=ENABLED
  ```

- {% include step_label.html %} Aplica el Ingress.

  ```bash
  kubectl apply -f k8s/ingress.yaml | tee outputs/ingress_apply.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Observa el estado del Ingress hasta que tenga `ADDRESS`.

  > **NOTA:** La creación del Load Balancer y asignación de IP puede tardar minutos. Para salir ejecuta **CTRL + C**
  {: .lab-note .warning .compact}

  ```bash
  source scripts/env.sh
  kubectl -n "$NAMESPACE" get ingress "$INGRESS_NAME" -w
  ```
  {% include step_image.html %}
  {% include step_image.html %}

#### Tarea 6.3 — Probar acceso público y guardar evidencia

- {% include step_label.html %} Cuando el Ingress tenga `ADDRESS`, prueba la app por IP.

  > **NOTA:** `HTTP 200` en `/healthz` confirma que el LB enruta hacia pods saludables.
  {: .lab-note .important .compact}

  ```bash
  source scripts/env.sh
  export INGRESS_IP="$(cat outputs/ingress_ip.txt | cut -d= -f2)"
  curl -sS "http://${INGRESS_IP}/" | grep -E "Lab08|GKE|Ingress|Nginx" | head | tee outputs/curl_home.txt
  ```
  {% include step_image.html %}
  ```bash
  curl -sS -o /dev/null -w "HTTP %{http_code}\n" "http://${INGRESS_IP}/healthz" | tee outputs/curl_healthz.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Describe el Ingress.

  > **NOTA:** Verás eventos de aprovisionamiento y el backend asociado.
  {: .lab-note .info .compact}

  ```bash
  source scripts/env.sh
  kubectl -n "$NAMESPACE" describe ingress "$INGRESS_NAME" | tee outputs/ingress_describe.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} (UI) En consola: **Kubernetes Engine** luego en **Gateways, Services & Ingress**.

  {% include step_image.html %}

- {% include step_label.html %} Ahora da clic en la pestaña de **Ingress** y luego en la IP que muestra la columna de **Frontend**

  {% include step_image.html %}

- {% include step_label.html %} Abrira el sitio desplegado por el contenedor mediante GKE.

  > **NOTA:** Evidencia visual del endpoint público.
  {: .lab-note .info .compact}

  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[5] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}

---

### Tarea 7. Limpieza (Tiempo estimado: 3 min)

Elimina recursos para evitar costos (Ingress/LB, clúster, repositorio, IP).

#### Tarea 7.1

- {% include step_label.html %} Elimina Ingress y recursos Kubernetes.

  > **NOTA:** Eliminar el Ingress dispara la eliminación del Load Balancer (puede tardar).
  {: .lab-note .warning .compact}

  ```bash
  source scripts/env.sh
  kubectl -n "$NAMESPACE" delete ingress "$INGRESS_NAME" --ignore-not-found
  ```
  ```bash
  kubectl -n "$NAMESPACE" delete svc "$SVC_NAME" --ignore-not-found
  ```
  ```bash
  kubectl -n "$NAMESPACE" delete deploy "$DEPLOY_NAME" --ignore-not-found
  ```
  ```bash
  kubectl delete ns "$NAMESPACE" --ignore-not-found
  ```
  {% include step_image.html %}

- {% include step_label.html %} Elimina la IP global estática (si la reservaste).

  ```bash
  source scripts/env.sh
  gcloud compute addresses delete "$INGRESS_IP_NAME" --global --quiet
  ```
  {% include step_image.html %}

- {% include step_label.html %} Elimina el servicio del CLoud NAT

  ```bash
  # NAT
  gcloud compute routers nats delete "lab08-nat" \
    --router "lab08-router" \
    --region "$REGION" \
    --quiet | tee outputs/delete_nat.txt

  # Router
  gcloud compute routers delete "lab08-router" \
    --region "$REGION" \
    --quiet | tee outputs/delete_router.txt
  ```
  {% include step_image.html %}

- {% include step_label.html %} Elimina el clúster GKE (UI recomendado).

  > **NOTA:** GKE es el principal generador de costo si se deja corriendo.
  {: .lab-note .warning .compact}

  **UI (recomendado):**

  - Kubernetes Engine → Clusters → selecciona `lab08-gke` → Delete.

  {% include step_image.html %}

- {% include step_label.html %} Elimina el repositorio Artifact Registry.

  ```bash
  source scripts/env.sh
  gcloud artifacts repositories delete "$AR_REPO" --location "$AR_LOCATION" --quiet
  ```
  {% include step_image.html %}

- {% include step_label.html %} Verificación final.

  ```bash
  cd ~/labs-gcp-engineer/lab08
  mkdir -p scripts outputs

  cat > scripts/verify_cleanup_lab08.sh <<'EOF'
  #!/usr/bin/env bash
  set -euo pipefail

  # =========================
  # Lab08 - Verificación final de limpieza
  # =========================
  # Este script NO borra nada.
  # Solo verifica si aún existen recursos de Lab08 (GKE, Artifact Registry, IP global, NAT, etc.)
  # y muestra ✅/❌ por cada servicio.
  #
  # Recomendación: ejecútalo después de tu script/comandos de eliminación.
  # =========================

  # --- Helpers ---
  ts() { date "+%Y-%m-%d %H:%M:%S"; }
  ok() { echo "✅ $*"; }
  bad(){ echo "❌ $*"; }
  info(){ echo "ℹ️  $*"; }
  sep(){ echo "------------------------------------------------------------"; }

  # Carpeta de evidencias
  BASE_DIR="${BASE_DIR:-$HOME/labs-gcp-engineer/lab08}"
  cd "$BASE_DIR" 2>/dev/null || true
  mkdir -p outputs

  OUT="outputs/verify_cleanup_$(date +%Y%m%d_%H%M%S).log"
  exec > >(tee -a "$OUT") 2>&1

  echo "=== Lab08 | Verificación final de limpieza ==="
  echo "Timestamp: $(ts)"
  sep

  # Cargar variables si existe env.sh (NO falla si no existe)
  if [[ -f "scripts/env.sh" ]]; then
    # shellcheck disable=SC1091
    source scripts/env.sh
    info "Cargadas variables desde scripts/env.sh"
  else
    info "No encontré scripts/env.sh (sigo con variables por defecto / gcloud config)."
  fi

  # Variables con defaults razonables (por si env.sh no existe)
  REGION="${REGION:-us-central1}"
  AR_LOCATION="${AR_LOCATION:-us-central1}"
  AR_REPO="${AR_REPO:-lab08-repo}"
  CLUSTER_NAME="${CLUSTER_NAME:-lab08-gke}"
  NAMESPACE="${NAMESPACE:-lab08}"
  INGRESS_IP_NAME="${INGRESS_IP_NAME:-lab08-ingress-ip}"
  ROUTER_NAME="${ROUTER_NAME:-lab08-router}"
  NAT_NAME="${NAT_NAME:-lab08-nat}"

  PROJECT_ID="$(gcloud config get-value project 2>/dev/null || true)"

  echo "Proyecto activo (gcloud): ${PROJECT_ID:-'(vacío)'}"
  echo "REGION=$REGION | AR_LOCATION=$AR_LOCATION | AR_REPO=$AR_REPO"
  echo "CLUSTER_NAME=$CLUSTER_NAME | NAMESPACE=$NAMESPACE"
  echo "INGRESS_IP_NAME=$INGRESS_IP_NAME | ROUTER_NAME=$ROUTER_NAME | NAT_NAME=$NAT_NAME"
  sep

  if [[ -z "${PROJECT_ID}" ]]; then
    bad "No hay PROJECT_ID activo en gcloud. Ejecuta: gcloud config set project TU_PROJECT_ID"
    echo "Saliendo porque no puedo verificar recursos sin proyecto."
    exit 1
  fi

  # =========================
  # 1) GKE (Cluster)
  # =========================
  echo "=== 1) GKE: Cluster ($CLUSTER_NAME) ==="
  echo "Busco si el clúster aún existe en la región $REGION..."
  if gcloud container clusters describe "$CLUSTER_NAME" --region "$REGION" >/dev/null 2>&1; then
    bad "AÚN EXISTE: Cluster $CLUSTER_NAME (region: $REGION)"
    info "Si esperabas que ya no exista, elimínalo con: gcloud container clusters delete $CLUSTER_NAME --region $REGION --quiet"
  else
    ok "OK: Cluster eliminado (o nunca existió)."
  fi
  sep

  # =========================
  # 2) Artifact Registry (Repo)
  # =========================
  echo "=== 2) Artifact Registry: Repo ($AR_REPO) ==="
  echo "Verifico si el repositorio Docker aún existe en $AR_LOCATION..."
  if gcloud artifacts repositories describe "$AR_REPO" --location "$AR_LOCATION" >/dev/null 2>&1; then
    bad "AÚN EXISTE: Repo $AR_REPO (location: $AR_LOCATION)"
    info "Si esperabas que ya no exista, elimínalo con: gcloud artifacts repositories delete $AR_REPO --location $AR_LOCATION --quiet"
  else
    ok "OK: Repo eliminado (o nunca existió)."
  fi
  sep

  # =========================
  # 3) IP global estática del Ingress
  # =========================
  echo "=== 3) Compute: IP global ($INGRESS_IP_NAME) ==="
  echo "Verifico si la IP global reservada aún existe..."
  if gcloud compute addresses describe "$INGRESS_IP_NAME" --global >/dev/null 2>&1; then
    bad "AÚN EXISTE: IP global $INGRESS_IP_NAME"
    info "Si esperabas que ya no exista, elimínala con: gcloud compute addresses delete $INGRESS_IP_NAME --global --quiet"
  else
    ok "OK: IP global eliminada (o nunca existió)."
  fi
  sep

  # =========================
  # 4) Cloud NAT (Router/NAT)
  # =========================
  echo "=== 4) Networking: Cloud NAT ($NAT_NAME) + Router ($ROUTER_NAME) ==="
  echo "Verifico si existe la configuración de NAT y el router en $REGION..."
  if gcloud compute routers nats describe "$NAT_NAME" --router "$ROUTER_NAME" --region "$REGION" >/dev/null 2>&1; then
    bad "AÚN EXISTE: NAT $NAT_NAME (router: $ROUTER_NAME, region: $REGION)"
    info "Para borrar NAT: gcloud compute routers nats delete $NAT_NAME --router $ROUTER_NAME --region $REGION --quiet"
  else
    ok "OK: NAT eliminado (o nunca existió)."
  fi

  if gcloud compute routers describe "$ROUTER_NAME" --region "$REGION" >/dev/null 2>&1; then
    bad "AÚN EXISTE: Router $ROUTER_NAME (region: $REGION)"
    info "Para borrar router: gcloud compute routers delete $ROUTER_NAME --region $REGION --quiet"
  else
    ok "OK: Router eliminado (o nunca existió)."
  fi
  sep

  # =========================
  # 5) Kubernetes: Namespace (si kubectl aún tiene contexto)
  # =========================
  echo "=== 5) Kubernetes: Namespace ($NAMESPACE) ==="
  echo "Intento validar si el namespace sigue existiendo (esto requiere que kubectl apunte a un cluster vivo)..."

  if kubectl version --client >/dev/null 2>&1; then
    # kubectl existe, pero el contexto puede estar muerto si ya borraste el cluster
    if kubectl get ns "$NAMESPACE" >/dev/null 2>&1; then
      bad "AÚN EXISTE: Namespace $NAMESPACE"
      info "Si el cluster sigue vivo y quieres borrar el namespace: kubectl delete ns $NAMESPACE"
    else
      ok "OK: Namespace eliminado (o kubectl ya no tiene acceso al cluster)."
    fi
  else
    info "kubectl no está disponible; omito verificación de namespace."
  fi
  sep

  # =========================
  # 6) Residuos típicos del LB (Compute) creados por Ingress (opcional)
  # =========================
  echo "=== 6) Compute: Residuos típicos de Ingress/LB (opcional) ==="
  echo "Busco recursos globales que contengan 'k8s' o 'lab08' en el nombre."
  echo "OJO: Esto es informativo; algunos labs crean varios recursos 'k8s-*'."

  echo "-- Forwarding rules (global) --"
  FR="$(gcloud compute forwarding-rules list --global --format="value(name)" | egrep -i '(k8s|lab08)' || true)"
  if [[ -n "$FR" ]]; then
    bad "AÚN HAY forwarding-rules relacionadas:"
    echo "$FR"
  else
    ok "OK: no veo forwarding-rules k8s/lab08."
  fi

  echo "-- Target HTTP proxies (global) --"
  THP="$(gcloud compute target-http-proxies list --format="value(name)" | egrep -i '(k8s|lab08)' || true)"
  if [[ -n "$THP" ]]; then
    bad "AÚN HAY target-http-proxies relacionados:"
    echo "$THP"
  else
    ok "OK: no veo target-http-proxies k8s/lab08."
  fi

  echo "-- Target HTTPS proxies (global) --"
  TSP="$(gcloud compute target-https-proxies list --format="value(name)" | egrep -i '(k8s|lab08)' || true)"
  if [[ -n "$TSP" ]]; then
    bad "AÚN HAY target-https-proxies relacionados:"
    echo "$TSP"
  else
    ok "OK: no veo target-https-proxies k8s/lab08."
  fi

  echo "-- URL maps (global) --"
  UM="$(gcloud compute url-maps list --format="value(name)" | egrep -i '(k8s|lab08)' || true)"
  if [[ -n "$UM" ]]; then
    bad "AÚN HAY url-maps relacionados:"
    echo "$UM"
  else
    ok "OK: no veo url-maps k8s/lab08."
  fi

  echo "-- Backend services (global) --"
  BS="$(gcloud compute backend-services list --global --format="value(name)" | egrep -i '(k8s|lab08)' || true)"
  if [[ -n "$BS" ]]; then
    bad "AÚN HAY backend-services relacionados:"
    echo "$BS"
  else
    ok "OK: no veo backend-services k8s/lab08."
  fi
  sep

  echo "✅ Verificación final terminada."
  echo "Evidencia guardada en: $OUT"
  EOF

  chmod +x scripts/verify_cleanup_lab08.sh
  echo "OK: creado scripts/verify_cleanup_lab08.sh"
  ```
  ```bash
  cd ~/labs-gcp-engineer/lab08
  ./scripts/verify_cleanup_lab08.sh
  ```
  {% include step_image.html %}

{% assign results = site.data.task-results[page.slug].results %}
{% capture r1 %}{{ results[6] }}{% endcapture %}
{% include task-result.html title="Tarea finalizada" content=r1 %}
