```/text
cat > pdf_lab.sh <<'LABEOF'
#!/bin/bash
# Create PDFs with Go and Cloud Run (GSP762) - universal solution

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
ask "Region (Lab setup panel)                 " "$DET_REGION" REGION
ask "Repository + image name                  " "pdf-converter" REPO
ask "Cloud Run service name                   " "pdf-converter" SERVICE
ask "Dockerfile base image                    " "amd64/debian" BASE_IMAGE
ask "Upload bucket                            " "${PROJECT_ID}-upload" UPLOAD_BUCKET
ask "Processed bucket                         " "${PROJECT_ID}-processed" PROCESSED_BUCKET
ask "Pub/Sub topic                            " "new-doc" TOPIC
ask "Service account name                     " "pubsub-cloud-run-invoker" SA_NAME
ask "Pub/Sub subscription                     " "pdf-conv-sub" SUB
ask "Test files to upload (Task 7)            " "gs://spls/gsp762/*" TEST_FILES
ask "Cloud Run memory                         " "2Gi" MEMORY
ask "Pause so you can click 'Check my progress'? (y/n)" "y" PAUSE

IMAGE="${REGION}-docker.pkg.dev/${PROJECT_ID}/${REPO}/${REPO}"
SA_EMAIL="${SA_NAME}@${PROJECT_ID}.iam.gserviceaccount.com"
PN=$(gcloud projects describe "$PROJECT_ID" --format="value(projectNumber)" 2>/dev/null)

# ---------- helpers ----------
pause() {
  if [ "$PAUSE" = "y" ]; then
    read -r -p ">>> Lab page: click 'Check my progress' for: $1 -- then press ENTER here to continue " _
  fi
}

image_exists() {
  gcloud artifacts docker images list "$IMAGE" --format="value(package)" 2>/dev/null | grep -q .
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

submit_build() {
  local i
  for i in 1 2 3; do
    if gcloud builds submit --tag "$IMAGE" --timeout=20m --quiet; then return 0; fi
    echo "Build failed (try $i/3)."
    [ "$i" -eq 1 ] && fix_build_permissions
    sleep 15
  done
  return 1
}

deploy_service() {
  local i
  for i in 1 2 3; do
    if gcloud run deploy "$SERVICE" --image "$IMAGE" --region "$REGION" \
        --memory="$MEMORY" --no-allow-unauthenticated \
        --set-env-vars "PDF_BUCKET=${PROCESSED_BUCKET}" --max-instances=3 --quiet; then
      return 0
    fi
    echo "Deploy failed (try $i/3) - waiting 20s"; sleep 20
  done
  return 1
}

get_url() {
  gcloud run services describe "$SERVICE" --region "$REGION" --format="value(status.url)" 2>/dev/null
}

# ---------- TASK 1: source code ----------
echo
echo "=== TASK 1: get the source code ==="
gcloud auth list --filter=status:ACTIVE --format="value(account)"
cd "$HOME" || exit 1
[ -d pet-theory ] || git clone https://github.com/Deleplace/pet-theory.git
cd "$HOME/pet-theory/lab03" || { echo "FAIL: pet-theory/lab03 not found"; exit 1; }

# ---------- TASK 2: server.go ----------
echo
echo "=== TASK 2: invoice microservice (server.go) ==="
cat > server.go <<'GOEOF'
package main

import (
    "fmt"
    "log"
    "net/http"
    "os"
    "os/exec"
    "regexp"
    "strings"
)

func main() {
    http.HandleFunc("/", process)
    port := os.Getenv("PORT")
    if port == "" {
        port = "8080"
        log.Printf("Defaulting to port %s", port)
    }

    log.Printf("Listening on port %s", port)
    err := http.ListenAndServe(fmt.Sprintf(":%s", port), nil)
    log.Fatal(err)
}

func process(w http.ResponseWriter, r *http.Request) {
    log.Println("Serving request")

    if r.Method == "GET" {
        fmt.Fprintln(w, "Ready to process POST requests from Cloud Storage trigger")
        return
    }

    //
    // Read request body containing Cloud Storage object metadata
    //
    gcsInputFile, err1 := readBody(r)
    if err1 != nil {
        log.Printf("Error reading POST data: %v", err1)
        w.WriteHeader(http.StatusBadRequest)
        fmt.Fprintf(w, "Problem with POST data: %v \n", err1)
        return
    }

    //
    // Working directory (concurrency-safe)
    localDir, err := os.MkdirTemp("", "")
    if err != nil {
        log.Printf("Error creating local temp dir: %v", err)
        w.WriteHeader(http.StatusInternalServerError)
        fmt.Fprintf(w, "Could not create a temp directory on server. \n")
        return
    }
    defer os.RemoveAll(localDir)

    //
    // Download input file from Cloud Storage
    //
    localInputFile, err2 := download(gcsInputFile, localDir)
    if err2 != nil {
        log.Printf("Error downloading Cloud Storage file [%s] from bucket [%s]: %v", gcsInputFile.Name, gcsInputFile.Bucket, err2)
        w.WriteHeader(http.StatusInternalServerError)
        fmt.Fprintf(w, "Error downloading Cloud Storage file [%s] from bucket [%s]", gcsInputFile.Name, gcsInputFile.Bucket)
        return
    }

    //
    // Use LibreOffice to convert local input file to local PDF file.
    //
    localPDFFilePath, err3 := convertToPDF(localInputFile.Name(), localDir)
    if err3 != nil {
        log.Printf("Error converting to PDF: %v", err3)
        w.WriteHeader(http.StatusInternalServerError)
        fmt.Fprintf(w, "Error converting to PDF.")
        return
    }

    //
    // Upload the freshly generated PDF to Cloud Storage
    //
    targetBucket := os.Getenv("PDF_BUCKET")
    err4 := upload(localPDFFilePath, targetBucket)
    if err4 != nil {
        log.Printf("Error uploading PDF file to bucket [%s]: %v", targetBucket, err4)
        w.WriteHeader(http.StatusInternalServerError)
        fmt.Fprintf(w, "Error downloading Cloud Storage file [%s] from bucket [%s]", gcsInputFile.Name, gcsInputFile.Bucket)
        return
    }

    //
    // Delete the original input file from Cloud Storage.
    //
    err5 := deleteGCSFile(gcsInputFile.Bucket, gcsInputFile.Name)
    if err5 != nil {
        log.Printf("Error deleting file [%s] from bucket [%s]: %v", gcsInputFile.Name, gcsInputFile.Bucket, err5)
        // This is not a blocking error.
        // The PDF was successfully generated and uploaded.
    }

    log.Println("Successfully produced PDF")
    fmt.Fprintln(w, "Successfully produced PDF")
}

func convertToPDF(localFilePath string, localDir string) (resultFilePath string, err error) {
    log.Printf("Converting [%s] to PDF", localFilePath)
    cmd := exec.Command("libreoffice", "--headless", "--convert-to", "pdf", "--outdir", localDir, localFilePath)
    cmd.Stdout, cmd.Stderr = os.Stdout, os.Stderr
    log.Println(cmd)
    err = cmd.Run()
    if err != nil {
        return "", err
    }
    pdfFilePath := regexp.MustCompile(`\.\w+$`).ReplaceAllString(localFilePath, ".pdf")
    if !strings.HasSuffix(pdfFilePath, ".pdf") {
        pdfFilePath += ".pdf"
    }
    log.Printf("Converted %s to %s", localFilePath, pdfFilePath)
    return pdfFilePath, nil
}
GOEOF

for f in notification.go gcs.go; do
  [ -f "$f" ] || echo "WARNING: $f is missing in $(pwd) - the build may fail"
done
if ! go build -o server; then
  echo "go build failed - running 'go mod tidy' and retrying"
  go mod tidy && go build -o server || { echo "FAIL: go build"; exit 1; }
fi
echo "PASS  server binary built"

# ---------- TASK 3: container, build, deploy ----------
echo
echo "=== TASK 3: PDF conversion service ==="
cat > Dockerfile <<EOF
FROM ${BASE_IMAGE}
RUN apt-get update -y \\
    && apt-get install -y libreoffice \\
    && apt-get clean
WORKDIR /usr/src/app
COPY server .
CMD [ "./server" ]
EOF

if ! gcloud artifacts repositories describe "$REPO" --location="$REGION" >/dev/null 2>&1; then
  gcloud artifacts repositories create "$REPO" --repository-format=docker --location="$REGION" --quiet
fi

if image_exists; then
  echo "Image already in Artifact Registry - skipping build"
else
  echo "Building the image (installing LibreOffice) - this takes several minutes..."
  submit_build || { echo "FAIL: could not build the image"; exit 1; }
fi
pause "Build an image with Cloud Build"

deploy_service || { echo "FAIL: deploy"; exit 1; }
pause "Deploy the PDF converter service"

# ---------- TASK 4: notification, service account, permissions ----------
echo
echo "=== TASK 4: service account ==="
for b in "$UPLOAD_BUCKET" "$PROCESSED_BUCKET"; do
  if ! gcloud storage buckets describe "gs://${b}" >/dev/null 2>&1; then
    echo "Bucket gs://${b} not found - creating it"
    gcloud storage buckets create "gs://${b}" --location="$REGION" --quiet
  fi
done

if gcloud storage buckets notifications list "gs://${UPLOAD_BUCKET}" 2>/dev/null | grep -q "topics/${TOPIC}"; then
  echo "Notification for topic ${TOPIC} already exists"
else
  gcloud storage buckets notifications create --topic="$TOPIC" --payload-format=json \
    --event-types=OBJECT_FINALIZE "gs://${UPLOAD_BUCKET}"
fi

if gcloud iam service-accounts describe "$SA_EMAIL" >/dev/null 2>&1; then
  echo "Service account ${SA_NAME} already exists"
else
  gcloud iam service-accounts create "$SA_NAME" --display-name "PubSub Cloud Run Invoker"
fi

gcloud run services add-iam-policy-binding "$SERVICE" \
  --member="serviceAccount:${SA_EMAIL}" \
  --role=roles/run.invoker \
  --region "$REGION" --quiet
pause "Create a service account"

# ---------- TASK 5: test the service ----------
echo
echo "=== TASK 5: test the Cloud Run service ==="
SERVICE_URL=$(get_url)
echo "Service URL: $SERVICE_URL"
ANON_CODE=$(curl -s -o /dev/null -w "%{http_code}" "$SERVICE_URL")
echo "Anonymous GET -> HTTP $ANON_CODE (403 is what you want)"
AUTH_OUT=""
for i in 1 2 3 4 5 6; do
  AUTH_OUT=$(curl -s -X GET -H "Authorization: Bearer $(gcloud auth print-identity-token)" "$SERVICE_URL")
  echo "$AUTH_OUT" | grep -q "Ready to process" && break
  echo "authenticated GET not ready yet (try $i/6) - waiting 10s"; sleep 10
done
echo "Authenticated GET -> $AUTH_OUT"

# ---------- TASK 6: Pub/Sub subscription ----------
echo
echo "=== TASK 6: Cloud Storage trigger ==="
if gcloud pubsub subscriptions describe "$SUB" >/dev/null 2>&1; then
  echo "Subscription ${SUB} already exists"
else
  for i in 1 2 3; do
    if gcloud pubsub subscriptions create "$SUB" --topic "$TOPIC" \
        --push-endpoint="$SERVICE_URL" --push-auth-service-account="$SA_EMAIL"; then
      break
    fi
    echo "Subscription create failed (try $i/3)."
    if [ "$i" -eq 1 ]; then
      echo "Granting token-creator to the Pub/Sub service agent (only done after a failure)..."
      gcloud projects add-iam-policy-binding "$PROJECT_ID" \
        --member="serviceAccount:service-${PN}@gcp-sa-pubsub.iam.gserviceaccount.com" \
        --role=roles/iam.serviceAccountTokenCreator --quiet >/dev/null 2>&1
    fi
    sleep 15
  done
fi
pause "Confirm the Pub/Sub subscription"

# ---------- TASK 7: test the notification ----------
echo
echo "=== TASK 7: upload test files and wait for the PDFs ==="
gcloud storage cp -r "$TEST_FILES" "gs://${UPLOAD_BUCKET}" --quiet
for i in $(seq 1 30); do
  LEFT=$(gcloud storage ls "gs://${UPLOAD_BUCKET}" 2>/dev/null | grep -c .)
  DONE=$(gcloud storage ls "gs://${PROCESSED_BUCKET}" 2>/dev/null | grep -c .)
  echo "waiting... files still in upload bucket: $LEFT | PDFs in processed bucket: $DONE ($i/30)"
  [ "$LEFT" -eq 0 ] && [ "$DONE" -gt 0 ] && break
  sleep 20
done

# ---------- SELF-CHECK ----------
echo
echo "=================== SELF-CHECK ==================="
image_exists && echo "PASS  image is in Artifact Registry (${REPO})" || echo "FAIL  image missing"
gcloud run services describe "$SERVICE" --region "$REGION" --format="value(spec.template.spec.containers[0].env)" 2>/dev/null | grep -q "$PROCESSED_BUCKET" \
  && echo "PASS  Cloud Run service deployed with PDF_BUCKET=${PROCESSED_BUCKET}" || echo "FAIL  service missing or PDF_BUCKET not set"
[ "$ANON_CODE" = "403" ] && echo "PASS  service is private (anonymous GET = 403)" || echo "CHECK anonymous GET returned $ANON_CODE (expected 403)"
echo "$AUTH_OUT" | grep -q "Ready to process" && echo "PASS  authenticated GET works" || echo "FAIL  authenticated GET did not work"
gcloud iam service-accounts describe "$SA_EMAIL" >/dev/null 2>&1 && echo "PASS  service account exists" || echo "FAIL  service account missing"
gcloud run services get-iam-policy "$SERVICE" --region "$REGION" 2>/dev/null | grep -q "$SA_EMAIL" \
  && echo "PASS  service account can invoke the service (run.invoker)" || echo "FAIL  run.invoker binding missing"
gcloud storage buckets notifications list "gs://${UPLOAD_BUCKET}" 2>/dev/null | grep -q "topics/${TOPIC}" \
  && echo "PASS  upload bucket notifies topic ${TOPIC}" || echo "FAIL  bucket notification missing"
[ "$(gcloud pubsub subscriptions describe "$SUB" --format="value(pushConfig.pushEndpoint)" 2>/dev/null)" = "$SERVICE_URL" ] \
  && echo "PASS  subscription ${SUB} pushes to the service" || echo "FAIL  subscription missing or wrong endpoint"
PDFS=$(gcloud storage ls "gs://${PROCESSED_BUCKET}" 2>/dev/null | grep -c '\.pdf')
[ "$PDFS" -gt 0 ] && echo "PASS  ${PDFS} PDF(s) in the processed bucket" || echo "CHECK no PDFs yet in the processed bucket (not scored; can take a few more minutes)"
echo
echo "Now click 'Check my progress' for any checkpoint that is not green yet."
LABEOF
```

bash pdf_lab.sh
