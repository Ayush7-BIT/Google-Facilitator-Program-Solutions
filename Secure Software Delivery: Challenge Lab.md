```/text
cat > securedelivery_lab.sh <<'LABEOF'
#!/bin/bash
# Secure Software Delivery: Challenge Lab (GSP521) - universal solution
# Run in Cloud Shell:  bash securedelivery_lab.sh
# Safe to re-run: it remembers your answers and resumes from the last finished task.

SCRIPT_PATH=$(readlink -f "$0")

PROJECT_ID=$(gcloud config get-value project 2>/dev/null)
if [ -z "$PROJECT_ID" ] || [ "$PROJECT_ID" = "(unset)" ]; then
  echo "ERROR: no project set. Run: gcloud config set project <Project ID from the lab panel>"
  exit 1
fi
export GOOGLE_CLOUD_PROJECT="$PROJECT_ID"
echo "Project: $PROJECT_ID"

WORKDIR="$HOME/sample-app"
STATE_DIR="$HOME/.gsp521"
mkdir -p "$STATE_DIR"
VARS_FILE="$STATE_DIR/vars"
PROGRESS_FILE="$STATE_DIR/done"
[ -f "$VARS_FILE" ] && . "$VARS_FILE"
DONE=$(cat "$PROGRESS_FILE" 2>/dev/null)
DONE=${DONE:-0}
START_DONE=$DONE

ask() {   # ask "prompt" "default" VARNAME
  local ans
  read -r -p "$1 [$2]: " ans
  ans=$(echo "${ans:-$2}" | xargs)
  printf -v "$3" '%s' "$ans"
}
fail() { echo; echo "FAIL: $1"; echo "Fix the error above, then re-run: bash $SCRIPT_PATH  (it resumes where it stopped)"; exit 1; }
mark() { echo "$1" > "$PROGRESS_FILE"; DONE=$1; }
pause() { read -r -p ">>> Lab page: click 'Check my progress' for: $1 -- wait for green, then press ENTER here " _; }

# ---------- detect the region ----------
meta() { gcloud compute project-info describe --quiet --format="value(commonInstanceMetadata.items[$1])" 2>/dev/null </dev/null; }
DET_REGION=$(gcloud config get-value compute/region 2>/dev/null)
if [ -z "$DET_REGION" ] || [ "$DET_REGION" = "(unset)" ]; then DET_REGION=$(meta google-compute-default-region); fi
if [ -z "$DET_REGION" ]; then DET_REGION="us-central1"; echo "Could not detect the region - type the one from the lab text."; fi

echo
echo "=== Copy each value from the lab text (yellow), or press ENTER to accept the default ==="
ask "Region -> Task 2, step 3 ('...use the region <yellow>') - used for repos, Cloud Build and Cloud Run" "${REGION:-$DET_REGION}" REGION
printf 'REGION=%q\n' "$REGION" > "$VARS_FILE"
gcloud config set compute/region "$REGION" >/dev/null 2>&1
gcloud config set run/region "$REGION" >/dev/null 2>&1

PROJECT_NUMBER=$(gcloud projects describe "$PROJECT_ID" --format="value(projectNumber)" </dev/null)
SCAN_IMAGE="${REGION}-docker.pkg.dev/${PROJECT_ID}/artifact-scanning-repo/sample-image"
PROD_IMAGE="${REGION}-docker.pkg.dev/${PROJECT_ID}/artifact-prod-repo/sample-image:latest"
NOTE_ID="vulnerability_note"
ATTESTOR_ID="vulnerability-attestor"
KEY_VERSION_PATH="projects/${PROJECT_ID}/locations/global/keyRings/binauthz-keys/cryptoKeys/lab-key/cryptoKeyVersions/1"
TOKEN() { gcloud auth print-access-token </dev/null; }

# ---------- TASK 1: APIs, starter files, Artifact Registry repos ----------
if [ "$DONE" -lt 1 ]; then
  echo; echo "=== TASK 1: enable APIs (takes 1-2 minutes) ==="
  gcloud services enable cloudkms.googleapis.com run.googleapis.com cloudbuild.googleapis.com \
    container.googleapis.com containerregistry.googleapis.com artifactregistry.googleapis.com \
    containerscanning.googleapis.com ondemandscanning.googleapis.com binaryauthorization.googleapis.com </dev/null \
    || fail "could not enable the APIs"

  echo; echo "=== TASK 1: download the starter files into ~/sample-app ==="
  mkdir -p "$WORKDIR" && cd "$WORKDIR" || exit 1
  gcloud storage cp "gs://spls/gsp521/*" . </dev/null || fail "could not copy gs://spls/gsp521/*"
  ls -la

  echo; echo "=== TASK 1: Artifact Registry repos artifact-scanning-repo and artifact-prod-repo ($REGION) ==="
  for repo in artifact-scanning-repo artifact-prod-repo; do
    if gcloud artifacts repositories describe "$repo" --location="$REGION" >/dev/null 2>&1 </dev/null; then
      echo "   $repo already exists - skipping"
    else
      gcloud artifacts repositories create "$repo" --repository-format=docker --location="$REGION" \
        --description="$repo" </dev/null || fail "could not create $repo"
    fi
  done
  mark 1
fi
cd "$WORKDIR" || fail "$WORKDIR is missing - delete $STATE_DIR and re-run"
[ -f cloudbuild.yaml.orig ] || cp cloudbuild.yaml cloudbuild.yaml.orig 2>/dev/null
[ -f Dockerfile.orig ] || cp Dockerfile Dockerfile.orig 2>/dev/null

# ---------- permissions for the Cloud Build service accounts ----------
# (the lab pre-grants most of these; adding them again is harmless and makes the pipeline steps work)
grant_build_roles() {
  for sa in "${PROJECT_NUMBER}@cloudbuild.gserviceaccount.com" "${PROJECT_NUMBER}-compute@developer.gserviceaccount.com"; do
    for role in roles/iam.serviceAccountUser roles/ondemandscanning.admin roles/artifactregistry.writer \
                roles/cloudkms.signerVerifier roles/cloudkms.viewer roles/binaryauthorization.attestorsViewer \
                roles/containeranalysis.notes.attacher roles/containeranalysis.notes.occurrences.viewer \
                roles/containeranalysis.occurrences.editor roles/run.admin roles/storage.admin roles/logging.logWriter; do
      gcloud projects add-iam-policy-binding "$PROJECT_ID" --member="serviceAccount:$sa" --role="$role" \
        --condition=None --quiet >/dev/null 2>&1 </dev/null
    done
  done
  echo "   build service account roles checked"
}

# ---------- TASK 2: first Cloud Build (build + push to artifact-scanning-repo) ----------
if [ "$DONE" -lt 2 ]; then
  echo; echo "=== TASK 2: build sample-image and push it to artifact-scanning-repo ==="
  grant_build_roles
  cat > cloudbuild-task2.yaml <<EOF
steps:
- id: "build"
  name: 'gcr.io/cloud-builders/docker'
  args: ['build', '-t', '${SCAN_IMAGE}', '.']
  waitFor: ['-']

- id: "push"
  name: 'gcr.io/cloud-builders/docker'
  args: ['push', '${SCAN_IMAGE}']

images:
  - ${SCAN_IMAGE}
EOF
  gcloud builds submit . --region="$REGION" --config=cloudbuild-task2.yaml </dev/null || fail "the Task 2 build failed"
  mark 2
fi

# ---------- TASK 3: note, attestor, KMS key, policy ----------
if [ "$DONE" -lt 3 ]; then
  echo; echo "=== TASK 3a: Container Analysis note $NOTE_ID ==="
  cat > vulnz_note.json <<'EOF'
{
  "attestation": {
    "hint": {
      "human_readable_name": "Container Vulnerabilities attestation authority"
    }
  }
}
EOF
  curl -s -X POST -H "Content-Type: application/json" -H "Authorization: Bearer $(TOKEN)" \
    --data-binary @./vulnz_note.json \
    "https://containeranalysis.googleapis.com/v1/projects/${PROJECT_ID}/notes/?noteId=${NOTE_ID}" | head -n 20
  curl -s -H "Authorization: Bearer $(TOKEN)" \
    "https://containeranalysis.googleapis.com/v1/projects/${PROJECT_ID}/notes/${NOTE_ID}" | grep -q "\"name\"" \
    || fail "note $NOTE_ID was not created"
  echo "   note $NOTE_ID exists"

  echo; echo "=== TASK 3b: attestor $ATTESTOR_ID ==="
  if gcloud container binauthz attestors describe "$ATTESTOR_ID" >/dev/null 2>&1 </dev/null; then
    echo "   $ATTESTOR_ID already exists - skipping"
  else
    gcloud container binauthz attestors create "$ATTESTOR_ID" \
      --attestation-authority-note="$NOTE_ID" --attestation-authority-note-project="$PROJECT_ID" </dev/null \
      || fail "could not create attestor $ATTESTOR_ID"
  fi
  gcloud container binauthz attestors list </dev/null

  echo; echo "=== TASK 3c: let the Binary Authorization service account view the note ==="
  gcloud beta services identity create --service=binaryauthorization.googleapis.com --project="$PROJECT_ID" >/dev/null 2>&1 </dev/null
  BINAUTHZ_SA_EMAIL="service-${PROJECT_NUMBER}@gcp-sa-binaryauthorization.iam.gserviceaccount.com"
  cat > iam_request.json <<EOF
{
  "resource": "projects/${PROJECT_ID}/notes/${NOTE_ID}",
  "policy": {
    "bindings": [
      {
        "role": "roles/containeranalysis.notes.occurrences.viewer",
        "members": [
          "serviceAccount:${BINAUTHZ_SA_EMAIL}"
        ]
      }
    ]
  }
}
EOF
  curl -s -X POST -H "Content-Type: application/json" -H "Authorization: Bearer $(TOKEN)" \
    --data-binary @./iam_request.json \

```

bash securedelivery_lab.sh
