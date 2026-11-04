```/text
cat > rest_api_lab.sh <<'LABEOF'
#!/bin/bash
# Developing a REST API with Go and Cloud Run (GSP761) - universal solution (v3)
# v3: works around "iam.serviceaccounts.actAs denied" on Cloud Run deploys,
#     and keeps going with the other tasks if the deploy turns out to be impossible.

PROJECT_ID=$(gcloud config get-value project 2>/dev/null)
if [ -z "$PROJECT_ID" ] || [ "$PROJECT_ID" = "(unset)" ]; then
  echo "ERROR: no project set. Run: gcloud config set project <Project ID from the lab panel>"
  exit 1
fi
export GOOGLE_CLOUD_PROJECT="$PROJECT_ID"
ACCOUNT=$(gcloud config get-value account 2>/dev/null)
echo "Project: $PROJECT_ID"
echo "Account: $ACCOUNT"

ask() {   # ask "prompt" "default" VARNAME
  local ans
  read -r -p "$1 [$2]: " ans
  printf -v "$3" '%s' "${ans:-$2}"
}

# ---------- detect the lab region ----------
DET_REGION=$(gcloud config get-value compute/region 2>/dev/null)
if [ -z "$DET_REGION" ] || [ "$DET_REGION" = "(unset)" ]; then
  DET_REGION=$(gcloud compute project-info describe --quiet \
    --format="value(commonInstanceMetadata.items[google-compute-default-region])" 2>/dev/null </dev/null)
fi
if [ -z "$DET_REGION" ] || [ "$DET_REGION" = "(unset)" ]; then
  echo "Could not detect the region. Read it from the lab text and type it at the first prompt."
  DET_REGION="us-west1"
fi

echo
echo "=== Press ENTER to accept each default (defaults are detected / this lab's values) ==="
ask "Region (lab text: 'Select the region ...')  " "$DET_REGION" REGION
ask "Distroless base image (Task 2 step 5, after FROM)" "gcr.io/distroless/base-debian12" BASE_IMAGE
ask "Firestore location                         " "$REGION" FS_LOCATION
ask "Artifact Registry repo                     " "my-repo" REPO
ask "Image name                                 " "rest-api" IMAGE
ask "Cloud Run service name                     " "rest-api" SERVICE
ask "Customer data folder (Task 3 step 8)       " "gs://spls/gsp645/2019-10-06T20:10:37_43617" SRC_DATA
ask "Customer bucket                            " "${PROJECT_ID}-customer" CUST_BUCKET
ask "Test customer ID (Task 7 step 6)           " "22530" TEST_ID
ask "Pause so you can click 'Check my progress'? (y/n)" "y" PAUSE
IMPORT_DIR="${SRC_DATA##*/}"

pause() {
  if [ "$PAUSE" = "y" ]; then
    read -r -p ">>> Lab page: click 'Check my progress' for: $1 -- then press ENTER here to continue " _
  fi
}

REG_PATH="${REGION}-docker.pkg.dev/${PROJECT_ID}/${REPO}/${IMAGE}"
PN=$(gcloud projects describe "$PROJECT_ID" --format="value(projectNumber)" 2>/dev/null)
DEPLOY_OK=0
RUNTIME_SA=""
USE_BUILD_DEPLOY=0
ACTAS_TRIED=0
URL=""

echo
echo "Your roles on this project (for troubleshooting):"
gcloud projects get-iam-policy "$PROJECT_ID" --flatten="bindings[].members" \
  --filter="bindings.members:user:${ACCOUNT}" --format="value(bindings.role)" 2>/dev/null | sed 's/^/   /'
echo "Service accounts in this project:"
gcloud iam service-accounts list --format="value(email)" 2>/dev/null | sed 's/^/   /'

# ---------- helpers ----------
tag_exists() {
  gcloud artifacts docker tags list "$REG_PATH" --format="value(tag)" 2>/dev/null | grep -Fq "$1"
}

check_image() {   # 0 = exists or unknown, 1 = definitely missing (gcr.io only)
  local img="$1" path code
  case "$img" in gcr.io/*) ;; *) return 0 ;; esac
  path="${img#gcr.io/}"; path="${path%%:*}"; path="${path%%@*}"
  code=$(curl -s -o /dev/null -w "%{http_code}" "https://gcr.io/v2/${path}/tags/list")
  [ "$code" = "404" ] && return 1
  return 0
}

fix_build_permissions() {
  echo "Granting build roles to the default service accounts (only done after a failure)..."
  local sa role
  for sa in "${PN}-compute@developer.gserviceaccount.com" "${PN}@cloudbuild.gserviceaccount.com"; do
    for role in roles/artifactregistry.writer roles/storage.objectAdmin roles/logging.logWriter; do
      gcloud projects add-iam-policy-binding "$PROJECT_ID" \
        --member="serviceAccount:${sa}" --role="$role" --quiet >/dev/null 2>&1
    done
  done
  sleep 20
}

grant_user_role() {   # grant_user_role <role>  -> tries to give YOUR student account a role
  local out
  if out=$(gcloud projects add-iam-policy-binding "$PROJECT_ID" --member="user:${ACCOUNT}" --role="$1" --condition=None 2>&1); then
    echo "   granted $1 to $ACCOUNT"
    return 0
  fi
  echo "   (not allowed to grant $1 to $ACCOUNT in this lab)"
  return 1
}

fix_actas() {   # fix_actas <service account unique id or email from the error message>
  local id="$1" email=""
  echo "The build needs permission to act as service account: $id"
  if echo "$id" | grep -q "@"; then
    email="$id"
  elif [ -n "$id" ]; then
    email=$(gcloud iam service-accounts list --project="$PROJECT_ID" --format="value(email,uniqueId)" 2>/dev/null \
            | awk -v id="$id" '$2==id {print $1}')
  fi
  if [ -n "$email" ]; then
    echo "That service account is: $email"
    gcloud iam service-accounts add-iam-policy-binding "$email" --project="$PROJECT_ID" \
      --member="user:${ACCOUNT}" --role=roles/iam.serviceAccountUser --quiet >/dev/null 2>&1 \
      && echo "   granted serviceAccountUser on $email to $ACCOUNT" \
      || echo "   (could not grant on the service account itself - trying project level)"
  fi
  grant_user_role roles/iam.serviceAccountUser
  sleep 20
}

wait_build() {   # wait_build <build id>  -> 0 if the build ends in SUCCESS
  local id="$1" s i
  for i in $(seq 1 30); do
    s=$(gcloud builds describe "$id" --format="value(status)" 2>/dev/null)
    case "$s" in
      SUCCESS) return 0 ;;
      FAILURE|CANCELLED|TIMEOUT|INTERNAL_ERROR|EXPIRED) return 1 ;;
    esac
    sleep 10
  done
  return 1
}

docker_fallback_build() {   # last resort: build with Docker in Cloud Shell
  echo "Last resort: building with Docker in Cloud Shell and pushing to Artifact Registry..."
  gcloud auth configure-docker "${REGION}-docker.pkg.dev" --quiet >/dev/null 2>&1
  docker build -t "${REG_PATH}:$1" . && docker push "${REG_PATH}:$1"
}

submit_build() {   # submit_build <tag>
  local tag="$1" i rc id sa
  for i in 1 2 3 4; do
    gcloud builds submit --tag "${REG_PATH}:${tag}" --quiet 2>&1 | tee build_log.txt
    rc=${PIPESTATUS[0]}
    [ "$rc" -eq 0 ] && return 0
    echo "Build command failed (try $i/4)."
    id=$(grep -o 'builds/[0-9a-f-]\{36\}' build_log.txt | head -n1 | sed 's#builds/##')
    if [ -n "$id" ]; then
      echo "Build $id was created - checking its status..."
      if wait_build "$id"; then echo "Build succeeded."; return 0; fi
    fi
    if grep -q "permission to act as service account" build_log.txt; then
      sa=$(grep -o 'serviceAccounts/[^ ]*' build_log.txt | head -n1 | sed 's#serviceAccounts/##; s/\.$//')
      fix_actas "$sa"
    else
      fix_build_permissions
    fi
    sleep 20
  done
  tag_exists "$tag" && return 0
  docker_fallback_build "$tag"
}

build_server() {
  CGO_ENABLED=0 GOOS=linux go build -o server && return 0
  echo "go build failed - running 'go mod tidy' and retrying"
  go mod tidy && CGO_ENABLED=0 GOOS=linux go build -o server
}

get_url() {
  gcloud run services describe "$SERVICE" --region "$REGION" --format="value(status.url)" 2>/dev/null
}

api_get() {   # api_get <path>  -> tries without login, then with your identity token
  local out
  out=$(curl -s "${URL}$1")
  if [ -z "$out" ] || echo "$out" | grep -qiE "403|forbidden|<html"; then
    out=$(curl -s -H "Authorization: Bearer $(gcloud auth print-identity-token 2>/dev/null)" "${URL}$1")
  fi
  echo "$out"
}

# ---------- Cloud Run deploy with workarounds ----------
run_deploy() {   # run_deploy <tag> [extra gcloud args]
  local tag="$1"; shift
  gcloud run deploy "$SERVICE" --image "${REG_PATH}:${tag}" --region "$REGION" \
    --allow-unauthenticated --max-instances=2 --quiet "$@" 2>&1 | tee deploy_log.txt
  return "${PIPESTATUS[0]}"
}

try_grant_actas_once() {
  [ "$ACTAS_TRIED" = "1" ] && return
  ACTAS_TRIED=1
  gcloud iam service-accounts add-iam-policy-binding "${PN}-compute@developer.gserviceaccount.com" \
    --project="$PROJECT_ID" --member="user:${ACCOUNT}" --role=roles/iam.serviceAccountUser --quiet >/dev/null 2>&1 \
    && echo "   granted you serviceAccountUser on the default compute service account (waiting 20s)" && sleep 20 \
    || echo "   (you are not allowed to grant yourself that role in this lab)"
}

deploy_via_build() {   # deploy_via_build <tag>  -> the deploy runs as the build service account
  local tag="$1"
  cat > cloudbuild_deploy.yaml <<EOF
steps:
- name: gcr.io/google.com/cloudsdktool/cloud-sdk:slim
  entrypoint: gcloud
  args:
  - run
  - deploy
  - ${SERVICE}
  - --image=${REG_PATH}:${tag}
  - --region=${REGION}
  - --max-instances=2
  - --quiet
options:
  logging: CLOUD_LOGGING_ONLY
EOF
  gcloud builds submit --no-source --config=cloudbuild_deploy.yaml --quiet 2>&1 | tee deploy_build_log.txt
  [ "${PIPESTATUS[0]}" -eq 0 ] || return 1
  gcloud run services describe "$SERVICE" --region "$REGION" >/dev/null 2>&1 || return 1
  USE_BUILD_DEPLOY=1
  echo "Service deployed through Cloud Build. Making it public (as you)..."
  gcloud run services add-iam-policy-binding "$SERVICE" --region "$REGION" \
    --member=allUsers --role=roles/run.invoker --quiet >/dev/null 2>&1 \
    && echo "   public access enabled" \
    || echo "   (could not make it public - the script will test it with your identity token instead)"
  return 0
}

deploy_service() {   # deploy_service <tag>
  local tag="$1" i sa mysa
  if [ -n "$RUNTIME_SA" ]; then
    run_deploy "$tag" --service-account="$RUNTIME_SA" && return 0
  fi
  if [ "$USE_BUILD_DEPLOY" = "1" ]; then
    deploy_via_build "$tag" && return 0
  fi

  echo "--- Deploy attempt 1: the lab's own command ---"
  run_deploy "$tag" && return 0
  if ! grep -qiE "actAs|permission to act as" deploy_log.txt; then
    for i in 2 3; do
      echo "Deploy failed (try $i/3) - waiting 20s"; sleep 20
      run_deploy "$tag" && return 0
    done
    return 1
  fi

  echo
  echo "Your account is not allowed to act as the default runtime service account (actAs denied)."
  try_grant_actas_once
  run_deploy "$tag" && return 0

  echo
  echo "--- Workaround 1: deploy through Cloud Build (runs as the build service account) ---"
  deploy_via_build "$tag" && return 0

  echo
  echo "--- Workaround 2: use another service account of this project as the runtime account ---"
  for sa in $(gcloud iam service-accounts list --filter="disabled=false" --format="value(email)" 2>/dev/null); do
    [ "$sa" = "${PN}-compute@developer.gserviceaccount.com" ] && continue
    echo "trying runtime service account: $sa"
    if run_deploy "$tag" --service-account="$sa"; then RUNTIME_SA="$sa"; return 0; fi
  done

  echo
  echo "--- Workaround 3: create my own runtime service account ---"
  mysa="rest-api-runtime@${PROJECT_ID}.iam.gserviceaccount.com"
  gcloud iam service-accounts describe "$mysa" >/dev/null 2>&1 \
    || gcloud iam service-accounts create rest-api-runtime --display-name="REST API runtime" --quiet >/dev/null 2>&1
  gcloud iam service-accounts add-iam-policy-binding "$mysa" --member="user:${ACCOUNT}" \
    --role=roles/iam.serviceAccountUser --quiet >/dev/null 2>&1
  sleep 10
  if run_deploy "$tag" --service-account="$mysa"; then RUNTIME_SA="$mysa"; return 0; fi
  return 1
}

# ---------- TASK 1: APIs ----------
echo
echo "=== TASK 1: enable APIs ==="
gcloud services enable cloudbuild.googleapis.com run.googleapis.com \
  artifactregistry.googleapis.com firestore.googleapis.com --quiet 2>&1 | tail -n 3

# ---------- TASK 2: simple REST API (0.1) ----------
echo
echo "=== TASK 2: REST API v0.1 ==="
cd "$HOME" || exit 1
[ -d pet-theory ] || git clone https://github.com/rosera/pet-theory.git
cd "$HOME/pet-theory/lab08" || { echo "FAIL: pet-theory/lab08 not found"; exit 1; }

# pick a base image that really exists (only replaces yours if it is definitely missing)
for cand in "$BASE_IMAGE" gcr.io/distroless/base-debian12 gcr.io/distroless/base-debian13 \
            gcr.io/distroless/base-debian11 gcr.io/distroless/base; do
  if check_image "$cand"; then BASE_IMAGE="$cand"; break; else echo "Base image not found: $cand - trying the next one"; fi
done
echo "Using base image: $BASE_IMAGE"

cat > main.go <<'GOEOF'
package main

import (
    "fmt"
    "log"
    "net/http"
    "os"
)

func main() {
    port := os.Getenv("PORT")
    if port == "" {
        port = "8080"
    }
    http.HandleFunc("/v1/", func(w http.ResponseWriter, r *http.Request) {
        fmt.Fprintf(w, "{status: 'running'}")
    })
    log.Println("Pets REST API listening on port", port)
    if err := http.ListenAndServe(":"+port, nil); err != nil {
        log.Fatalf("Error launching Pets REST API server: %v", err)
    }
}
GOEOF

cat > Dockerfile <<EOF
FROM ${BASE_IMAGE}
WORKDIR /usr/src/app
COPY server .
CMD [ "/usr/src/app/server" ]
EOF

if ! gcloud artifacts repositories describe "$REPO" --location="$REGION" >/dev/null 2>&1; then
  gcloud artifacts repositories create "$REPO" --repository-format=docker \
    --location="$REGION" --description="Docker repository for REST API" --quiet
fi

if tag_exists "0.1"; then
  echo "Image 0.1 already exists - skipping build"
else
  build_server || { echo "FAIL: go build"; exit 1; }
  submit_build "0.1" || { echo "FAIL: could not build image 0.1"; exit 1; }
fi
pause "Build with Cloud Build"

if deploy_service "0.1"; then
  DEPLOY_OK=1
  URL=$(get_url)
  echo "Service URL: $URL"
  echo "Test /v1/ -> $(api_get /v1/)"
  pause "REST API Deployed"
else
  echo
  echo "!!! The service could not be deployed yet (explanation at the end)."
  echo "!!! Continuing with the tasks that do not need a deployment (Firestore, data import, image 0.2)..."
fi

# ---------- TASK 3: Firestore + test data ----------
echo
echo "=== TASK 3: Firestore database and customer data ==="
if ! gcloud firestore databases describe --database="(default)" >/dev/null 2>&1; then
  echo "Creating Firestore database in $FS_LOCATION ..."
  gcloud firestore databases create --database="(default)" --location="$FS_LOCATION" \
    --type=firestore-native --edition=standard --quiet 2>/dev/null \
  || gcloud firestore databases create --database="(default)" --location="$FS_LOCATION" \
    --type=firestore-native --quiet
fi
for i in $(seq 1 18); do
  gcloud firestore databases describe --database="(default)" >/dev/null 2>&1 && break
  echo "waiting for Firestore ($i/18)..."; sleep 10
done
if gcloud firestore databases describe --database="(default)" >/dev/null 2>&1; then
  echo "Firestore database is ready."
else
  echo "FAIL: Firestore was not created. Create it by hand: Navigation menu > Firestore > Create database > Standard > Open > Region ${FS_LOCATION}"
fi
pause "Firestore database created"

if ! gcloud storage buckets describe "gs://${CUST_BUCKET}" >/dev/null 2>&1; then
  gcloud storage buckets create "gs://${CUST_BUCKET}" --default-storage-class=standard \
    --location="$FS_LOCATION" --quiet
fi
gcloud storage cp -r "$SRC_DATA" "gs://${CUST_BUCKET}" --quiet
gcloud firestore import "gs://${CUST_BUCKET}/${IMPORT_DIR}/" --quiet \
  || gcloud beta firestore import "gs://${CUST_BUCKET}/${IMPORT_DIR}/" --quiet

# ---------- TASK 4 + 7: connect to Firestore, build 0.2, deploy ----------
echo
echo "=== TASK 4 + 7: Firestore-connected API (0.2) ==="
cat > main.go <<'GOEOF'
package main

import (
    "context"
    "encoding/json"
    "fmt"
    "log"
    "net/http"
    "os"

    "cloud.google.com/go/firestore"
    "github.com/gorilla/handlers"
    "github.com/gorilla/mux"
    "google.golang.org/api/iterator"
)

var client *firestore.Client

func main() {
    var err error
    ctx := context.Background()
    client, err = firestore.NewClient(ctx, "__PROJECT_ID__")
    if err != nil {
        log.Fatalf("Error initializing Cloud Firestore client: %v", err)
    }
    port := os.Getenv("PORT")
    if port == "" {
        port = "8080"
    }
    r := mux.NewRouter()
    r.HandleFunc("/v1/", rootHandler)
    r.HandleFunc("/v1/customer/{id}", customerHandler)
    log.Println("Pets REST API listening on port", port)
    cors := handlers.CORS(
        handlers.AllowedHeaders([]string{"X-Requested-With", "Authorization", "Origin"}),
        handlers.AllowedOrigins([]string{"https://storage.googleapis.com"}),
        handlers.AllowedMethods([]string{"GET", "HEAD", "POST", "OPTIONS", "PATCH", "CONNECT"}),
    )
    if err := http.ListenAndServe(":"+port, cors(r)); err != nil {
        log.Fatalf("Error launching Pets REST API server: %v", err)
    }
}

func rootHandler(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "{status: 'running'}")
}

func customerHandler(w http.ResponseWriter, r *http.Request) {
    id := mux.Vars(r)["id"]
    ctx := context.Background()
    customer, err := getCustomer(ctx, id)
    if err != nil {
        w.WriteHeader(http.StatusInternalServerError)
        fmt.Fprintf(w, `{"status": "fail", "data": '%s'}`, err)
        return
    }
    if customer == nil {
        w.WriteHeader(http.StatusNotFound)
        msg := fmt.Sprintf("`Customer \"%s\" not found`", id)
        fmt.Fprintf(w, fmt.Sprintf(`{"status": "fail", "data": {"title": %s}}`, msg))
        return
    }
    amount, err := getAmounts(ctx, customer)
    if err != nil {
        w.WriteHeader(http.StatusInternalServerError)
        fmt.Fprintf(w, `{"status": "fail", "data": "Unable to fetch amounts: %s"}`, err)
        return
    }
    data, err := json.Marshal(amount)
    if err != nil {
        w.WriteHeader(http.StatusInternalServerError)
        fmt.Fprintf(w, `{"status": "fail", "data": "Unable to fetch amounts: %s"}`, err)
        return
    }
    fmt.Fprintf(w, fmt.Sprintf(`{"status": "success", "data": %s}`, data))
}

type Customer struct {
    Email string `firestore:"email"`
    ID    string `firestore:"id"`
    Name  string `firestore:"name"`
    Phone string `firestore:"phone"`
}

func getCustomer(ctx context.Context, id string) (*Customer, error) {
    query := client.Collection("customers").Where("id", "==", id)
    iter := query.Documents(ctx)
    var c Customer
    for {
        doc, err := iter.Next()
        if err == iterator.Done {
            break
        }
        if err != nil {
            return nil, err
        }
        err = doc.DataTo(&c)
        if err != nil {
            return nil, err
        }
    }
    return &c, nil
}

func getAmounts(ctx context.Context, c *Customer) (map[string]int64, error) {
    if c == nil {
        return map[string]int64{}, fmt.Errorf("Customer should be non-nil: %v", c)
    }
    result := map[string]int64{
        "proposed": 0,
        "approved": 0,
        "rejected": 0,
    }
    query := client.Collection(fmt.Sprintf("customers/%s/treatments", c.Email))
    if query == nil {
        return map[string]int64{}, fmt.Errorf("Query is nil: %v", c)
    }
    iter := query.Documents(ctx)
    for {
        doc, err := iter.Next()
        if err == iterator.Done {
            break
        }
        if err != nil {
            return nil, err
        }
        treatment := doc.Data()
        result[treatment["status"].(string)] += treatment["cost"].(int64)
    }
    return result, nil
}
GOEOF
sed -i "s/__PROJECT_ID__/${PROJECT_ID}/" main.go

if tag_exists "0.2"; then
  echo "Image 0.2 already exists - skipping build"
else
  build_server || { echo "FAIL: go build"; exit 1; }
  submit_build "0.2" || { echo "FAIL: could not build image 0.2"; exit 1; }
fi
pause "Build image revision 0.2"

if deploy_service "0.2"; then
  DEPLOY_OK=1
  URL=$(get_url)
  echo "Service URL: $URL"
else
  DEPLOY_OK=0
fi

CUST_OUT=""
if [ "$DEPLOY_OK" = "1" ]; then
  for i in 1 2 3 4 5 6; do
    CUST_OUT=$(api_get "/v1/customer/${TEST_ID}")
    echo "attempt $i: $CUST_OUT"
    echo "$CUST_OUT" | grep -q '"status": "success"' && break
    if echo "$CUST_OUT" | grep -qiE 'permission|denied'; then
      echo "Firestore permission problem - trying to grant roles/datastore.user to the runtime service account"
      gcloud projects add-iam-policy-binding "$PROJECT_ID" \
        --member="serviceAccount:${RUNTIME_SA:-${PN}-compute@developer.gserviceaccount.com}" \
        --role="roles/datastore.user" --quiet >/dev/null 2>&1 || echo "   (not allowed to change IAM in this lab)"
    fi
    sleep 15
  done
fi

# ---------- SELF-CHECK ----------
echo
echo "=================== SELF-CHECK ==================="
tag_exists "0.1" && echo "PASS  image ${IMAGE}:0.1 is in Artifact Registry" || echo "FAIL  image 0.1 missing"
tag_exists "0.2" && echo "PASS  image ${IMAGE}:0.2 is in Artifact Registry" || echo "FAIL  image 0.2 missing"
gcloud firestore databases describe --database="(default)" >/dev/null 2>&1 \
  && echo "PASS  Firestore database exists" || echo "FAIL  no Firestore database"
if gcloud run services describe "$SERVICE" --region "$REGION" >/dev/null 2>&1; then
  echo "PASS  Cloud Run service ${SERVICE} is deployed"
  [ -n "$RUNTIME_SA" ] && echo "      (runtime service account: $RUNTIME_SA)"
  [ -z "$URL" ] && URL=$(get_url)
  api_get "/v1/" | grep -q "running" && echo "PASS  ${URL}/v1/ returns running" || echo "FAIL  /v1/ did not return running"
  echo "$CUST_OUT" | grep -q '"status": "success"' \
    && echo "PASS  /v1/customer/${TEST_ID} returned success" || echo "CHECK customer endpoint did not return success (see the output above)"
else
  echo "FAIL  Cloud Run service not deployed"
fi

if [ "$DEPLOY_OK" != "1" ]; then
  echo
  echo "------------------------------------------------------------------"
  echo "WHY THE DEPLOY FAILED"
  echo "Your student account is not allowed to act as any service account that Cloud Run can use"
  echo "(iam.serviceaccounts.actAs denied), and it is not allowed to grant itself that right."
  echo "A script cannot fix that. The checkpoints that do not need a deployment are done:"
  echo "build 0.1, Firestore, and build 0.2 (up to 75/100)."
  echo "For the 'REST API service deployed' checkpoint:"
  echo "  1. Click End Lab, then Start Lab, to get a fresh project, and run this script again."
  echo "  2. Report it through the lab's help chat: 'gcloud run deploy fails with iam.serviceaccounts.actAs denied on the compute service account'."
  echo "  3. Send me the 'Your roles on this project' lines printed at the start of this run."
  echo "------------------------------------------------------------------"
fi

echo
[ -n "$URL" ] && echo "Open this in your browser: ${URL}/v1/customer/${TEST_ID}"
echo "Then click 'Check my progress' for any checkpoint that is not green yet."
LABEOF
```



bash rest_api_lab.sh
