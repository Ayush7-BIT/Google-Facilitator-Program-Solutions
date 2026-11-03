```/text
cat > rest_api_lab.sh <<'LABEOF'
#!/bin/bash
# Developing a REST API with Go and Cloud Run (GSP761) - universal solution

PROJECT_ID=$(gcloud config get-value project 2>/dev/null)
if [ -z "$PROJECT_ID" ] || [ "$PROJECT_ID" = "(unset)" ]; then
  echo "ERROR: no project set. Run: gcloud config set project <Project ID from the lab panel>"
  exit 1
fi
export GOOGLE_CLOUD_PROJECT="$PROJECT_ID"
echo "Project: $PROJECT_ID"

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
  echo "Could not detect the region. Read it from the Lab setup panel and type it at the first prompt."
  DET_REGION="us-central1"
fi

echo
echo "=== Press ENTER to accept each default (defaults are detected / this lab's values) ==="
ask "Region (Lab setup panel)                   " "$DET_REGION" REGION
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
  echo "Granting build/deploy roles to the default service accounts (only done after a failure)..."
  local pn sa role
  pn=$(gcloud projects describe "$PROJECT_ID" --format="value(projectNumber)")
  for sa in "${pn}-compute@developer.gserviceaccount.com" "${pn}@cloudbuild.gserviceaccount.com"; do
    for role in roles/artifactregistry.writer roles/storage.objectAdmin roles/logging.logWriter; do
      gcloud projects add-iam-policy-binding "$PROJECT_ID" \
        --member="serviceAccount:${sa}" --role="$role" --quiet >/dev/null 2>&1
    done
  done
  sleep 20
}

submit_build() {   # tag
  local tag="$1" i
  for i in 1 2 3; do
    if gcloud builds submit --tag "${REG_PATH}:${tag}" --quiet; then return 0; fi
    echo "Build failed (try $i/3)."
    [ "$i" -eq 1 ] && fix_build_permissions
    sleep 15
  done
  return 1
}

build_server() {
  CGO_ENABLED=0 GOOS=linux go build -o server && return 0
  echo "go build failed - running 'go mod tidy' and retrying"
  go mod tidy && CGO_ENABLED=0 GOOS=linux go build -o server
}

deploy_service() {   # tag
  local tag="$1" i
  for i in 1 2 3; do
    if gcloud run deploy "$SERVICE" --image "${REG_PATH}:${tag}" \
        --region "$REGION" --allow-unauthenticated --max-instances=2 --quiet; then
      return 0
    fi
    echo "Deploy failed (try $i/3) - waiting 20s"; sleep 20
  done
  return 1
}

get_url() {
  gcloud run services describe "$SERVICE" --region "$REGION" --format="value(status.url)" 2>/dev/null
}

# ---------- TASK 1: APIs ----------
echo
echo "=== TASK 1: enable APIs ==="
gcloud services enable cloudbuild.googleapis.com run.googleapis.com \
  artifactregistry.googleapis.com firestore.googleapis.com --quiet

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

deploy_service "0.1" || { echo "FAIL: deploy 0.1"; exit 1; }
URL=$(get_url)
echo "Service URL: $URL"
echo "Test /v1/ -> $(curl -s "${URL}/v1/")"
pause "REST API Deployed"

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

deploy_service "0.2" || { echo "FAIL: deploy 0.2"; exit 1; }
URL=$(get_url)
echo "Service URL: $URL"

CUST_OUT=""
for i in 1 2 3 4 5 6; do
  CUST_OUT=$(curl -s "${URL}/v1/customer/${TEST_ID}")
  echo "attempt $i: $CUST_OUT"
  echo "$CUST_OUT" | grep -q '"status": "success"' && break
  if echo "$CUST_OUT" | grep -qiE 'permission|denied'; then
    echo "Firestore permission problem - granting roles/datastore.user to the default service account"
    pn=$(gcloud projects describe "$PROJECT_ID" --format="value(projectNumber)")
    gcloud projects add-iam-policy-binding "$PROJECT_ID" \
      --member="serviceAccount:${pn}-compute@developer.gserviceaccount.com" \
      --role="roles/datastore.user" --quiet >/dev/null 2>&1
  fi
  sleep 15
done

# ---------- SELF-CHECK ----------
echo
echo "=================== SELF-CHECK ==================="
tag_exists "0.1" && echo "PASS  image ${IMAGE}:0.1 is in Artifact Registry" || echo "FAIL  image 0.1 missing"
tag_exists "0.2" && echo "PASS  image ${IMAGE}:0.2 is in Artifact Registry" || echo "FAIL  image 0.2 missing"
gcloud run services describe "$SERVICE" --region "$REGION" >/dev/null 2>&1 \
  && echo "PASS  Cloud Run service ${SERVICE} is deployed" || echo "FAIL  service not deployed"
gcloud firestore databases describe --database="(default)" >/dev/null 2>&1 \
  && echo "PASS  Firestore database exists" || echo "FAIL  no Firestore database"
curl -s "${URL}/v1/" | grep -q "running" \
  && echo "PASS  ${URL}/v1/ returns running" || echo "FAIL  /v1/ did not return running"
echo "$CUST_OUT" | grep -q '"status": "success"' \
  && echo "PASS  /v1/customer/${TEST_ID} returned success" || echo "FAIL  customer endpoint did not return success"
echo
echo "Extra tests from the lab:"
for t in 34216 70156 12345; do echo "  /v1/customer/$t -> $(curl -s "${URL}/v1/customer/$t")"; done
echo
echo "Open this in your browser: ${URL}/v1/customer/${TEST_ID}"
echo "Then click 'Check my progress' for any checkpoint that is not green yet."
LABEOF
```



bash rest_api_lab.sh
