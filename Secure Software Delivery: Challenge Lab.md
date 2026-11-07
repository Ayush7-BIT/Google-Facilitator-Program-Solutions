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
    "https://containeranalysis.googleapis.com/v1/projects/${PROJECT_ID}/notes/${NOTE_ID}:setIamPolicy" | head -n 20

  echo; echo "=== TASK 3d: KMS keyring binauthz-keys and key lab-key (global) ==="
  gcloud kms keyrings describe binauthz-keys --location=global >/dev/null 2>&1 </dev/null \
    || gcloud kms keyrings create binauthz-keys --location=global </dev/null || fail "could not create keyring binauthz-keys"
  gcloud kms keys describe lab-key --keyring=binauthz-keys --location=global >/dev/null 2>&1 </dev/null \
    || gcloud kms keys create lab-key --keyring=binauthz-keys --location=global \
         --purpose=asymmetric-signing --default-algorithm=ec-sign-p256-sha256 </dev/null || fail "could not create key lab-key"

  echo; echo "=== TASK 3e: add lab-key version 1 to the attestor ==="
  if gcloud container binauthz attestors describe "$ATTESTOR_ID" --format=json </dev/null 2>/dev/null | grep -q "lab-key"; then
    echo "   key already added - skipping"
  else
    gcloud beta container binauthz attestors public-keys add --attestor="$ATTESTOR_ID" \
      --keyversion-project="$PROJECT_ID" --keyversion-location=global --keyversion-keyring=binauthz-keys \
      --keyversion-key=lab-key --keyversion=1 </dev/null || fail "could not add the key to the attestor"
  fi

  echo; echo "=== TASK 3f: Binary Authorization policy - default rule requires $ATTESTOR_ID ==="
  cat > my_policy.yaml <<EOF
defaultAdmissionRule:
  enforcementMode: ENFORCED_BLOCK_AND_AUDIT_LOG
  evaluationMode: REQUIRE_ATTESTATION
  requireAttestationsBy:
  - projects/${PROJECT_ID}/attestors/${ATTESTOR_ID}
globalPolicyEvaluationMode: ENABLE
name: projects/${PROJECT_ID}/policy
EOF
  gcloud container binauthz policy import my_policy.yaml </dev/null || fail "could not import the policy"
  mark 3
  pause "Create an Attestor, KMS pair, and update the policy (20)  [also click Task 1 and Task 2 checkpoints now]"
fi

# ---------- full pipeline (Task 4 / Task 5) ----------
write_pipeline() {
  cat > cloudbuild.yaml <<'EOF'
steps:
# 1. Build
- id: "build"
  name: 'gcr.io/cloud-builders/docker'
  args: ['build', '-t', '__SCAN_IMAGE__', '.']
  waitFor: ['-']

# 2. Push to Artifact Registry
- id: "push"
  name: 'gcr.io/cloud-builders/docker'
  args: ['push', '__SCAN_IMAGE__']

# 3. Vulnerability scan
- id: scan
  name: 'gcr.io/cloud-builders/gcloud'
  entrypoint: 'bash'
  args:
  - '-c'
  - |
      (gcloud artifacts docker images scan \
      __SCAN_IMAGE__ \
      --location us \
      --format="value(response.scan)") > /workspace/scan_id.txt

# 4. Fail the build if CRITICAL vulnerabilities are found
- id: severity check
  name: 'gcr.io/cloud-builders/gcloud'
  entrypoint: 'bash'
  args:
  - '-c'
  - |
      gcloud artifacts docker images list-vulnerabilities $$(cat /workspace/scan_id.txt) \
      --format="value(vulnerability.effectiveSeverity)" | if grep -Fxq CRITICAL; \
      then echo "Failed vulnerability check for CRITICAL level" && exit 1; else echo \
      "No CRITICAL vulnerability found, congrats !" && exit 0; fi

# 5. Sign the image (only reached if the severity check passed)
- id: 'create-attestation'
  name: 'gcr.io/${PROJECT_ID}/binauthz-attestation:latest'
  args:
    - '--artifact-url'
    - '__SCAN_IMAGE__'
    - '--attestor'
    - 'projects/__PROJECT_ID__/attestors/vulnerability-attestor'
    - '--keyversion'
    - '__KEY_VERSION__'

# 6. Re-tag for production and push
- id: "push-to-prod"
  name: 'gcr.io/cloud-builders/docker'
  args:
    - 'tag'
    - '__SCAN_IMAGE__'
    - '__PROD_IMAGE__'
- id: "push-to-prod-final"
  name: 'gcr.io/cloud-builders/docker'
  args: ['push', '__PROD_IMAGE__']

# 7. Deploy to Cloud Run
- id: 'deploy-to-cloud-run'
  name: 'gcr.io/cloud-builders/gcloud'
  entrypoint: 'bash'
  args:
  - '-c'
  - |
    gcloud run deploy auth-service --image=__PROD_IMAGE__ \
    --binary-authorization=default --region=__REGION__ --allow-unauthenticated

# 8. Image produced by the build step
images:
  - __SCAN_IMAGE__
EOF
  sed -i -e "s|__SCAN_IMAGE__|${SCAN_IMAGE}|g" -e "s|__PROD_IMAGE__|${PROD_IMAGE}|g" \
         -e "s|__PROJECT_ID__|${PROJECT_ID}|g" -e "s|__KEY_VERSION__|${KEY_VERSION_PATH}|g" \
         -e "s|__REGION__|${REGION}|g" cloudbuild.yaml
}

# Runs the full pipeline on the ORIGINAL (vulnerable) Dockerfile. The lab expects this build to FAIL
# at step #3 "severity check" because CRITICAL vulnerabilities are found.
run_failing_build() {   # run_failing_build SOURCE_DIR
  local src="$1" tmp="$STATE_DIR/task4-src"
  rm -rf "$tmp"; mkdir -p "$tmp"
  cp -r "$src"/. "$tmp"/
  [ -f "$tmp/Dockerfile.orig" ] && cp "$tmp/Dockerfile.orig" "$tmp/Dockerfile"
  echo; echo "=== TASK 4b: run the full pipeline on the original Dockerfile - it SHOULD fail at 'severity check' ==="
  echo "   NOTE: this build is SUPPOSED to fail. The red 'ERROR' / 'BUILD FAILURE' at the end is expected -"
  echo "         the lab's Task 4 checkpoint needs exactly this failed build. Do not worry about it."
  (cd "$tmp" && write_pipeline && gcloud builds submit . --region="$REGION" --config=cloudbuild.yaml </dev/null 2>&1) \
    | tee "$STATE_DIR/task4_build.log"
  echo
  if grep -q "Failed vulnerability check for CRITICAL level" "$STATE_DIR/task4_build.log" \
     && grep -q 'build step 3 ' "$STATE_DIR/task4_build.log"; then
    echo "   =============================================================================="
    echo "   PASS (Task 4): the build failed at step #3 'severity check' with"
    echo "                  'Failed vulnerability check for CRITICAL level' - exactly as the lab wants."
    echo "                  The red ERROR above is expected. Click 'Check my progress' for Task 4."
    echo "   =============================================================================="
    return 0
  fi
  local failed
  failed=$(grep -o 'build step [0-9]* "[^"]*" failed' "$STATE_DIR/task4_build.log" | head -n 1)
  if [ -n "$failed" ]; then
    echo "   WARNING: the build failed, but NOT at the severity check ($failed)."
    echo "            Send me a screenshot of the first red lines above."
  else
    echo "   WARNING: the build did not fail - no CRITICAL vulnerability was reported. Send me a screenshot."
  fi
  return 1
}

# ---------- TASK 4: custom attestation builder + pipeline run (expected to FAIL on CRITICAL) ----------
if [ "$DONE" -lt 4 ]; then
  echo; echo "=== TASK 4a: build the binauthz-attestation custom build step ==="
  grant_build_roles
  if gcloud container images describe "gcr.io/${PROJECT_ID}/binauthz-attestation:latest" >/dev/null 2>&1 </dev/null; then
    echo "   gcr.io/${PROJECT_ID}/binauthz-attestation:latest already exists - skipping"
  else
    rm -rf /tmp/cloud-builders-community
    git clone https://github.com/GoogleCloudPlatform/cloud-builders-community.git /tmp/cloud-builders-community \
      || fail "git clone failed"
    (cd /tmp/cloud-builders-community/binauthz-attestation && gcloud builds submit . --config cloudbuild.yaml </dev/null) \
      || fail "could not build the binauthz-attestation step"
    rm -rf /tmp/cloud-builders-community
  fi

  run_failing_build "$WORKDIR"
  mark 4
  pause "Integrate vulnerability scanning into your CI/CD pipeline (20)"
fi

# ---------- TASK 5: fix the Dockerfile and redeploy ----------
if [ "$DONE" -lt 5 ]; then
  echo; echo "=== TASK 5a: update the Dockerfile (python:3.12-slim, Flask 3.0.3, gunicorn 23.0.0, Werkzeug 3.0.4) ==="
  echo "--- Dockerfile before:"; cat Dockerfile
  sed -i -E \
    -e 's|^FROM[[:space:]]+[^[:space:]]+|FROM python:3.12-slim|' \
    -e 's/([Ff]lask)==[0-9][0-9.]*/\1==3.0.3/g' \
    -e 's/([Gg]unicorn)==[0-9][0-9.]*/\1==23.0.0/g' \
    -e 's/([Ww]erkzeug)==[0-9][0-9.]*/\1==3.0.4/g' Dockerfile
  # The lab's Dockerfile carries a workaround for the old Debian 'bullseye' base image that edits
  # /etc/apt/sources.list. python:3.12-slim has no such file, so that RUN step must be removed.
  python3 - <<'PYEOF'
import re
lines = open("Dockerfile").read().split("\n")
out, i = [], 0
while i < len(lines):
    if re.match(r"\s*RUN\b", lines[i], re.I):
        j = i
        while lines[j].rstrip().endswith("\\") and j + 1 < len(lines):
            j += 1
        block = lines[i:j + 1]
        if "sources.list" in "\n".join(block):
            print("   removed the old bullseye sources.list workaround step")
        else:
            out.extend(block)
        i = j + 1
    else:
        out.append(lines[i]); i += 1
open("Dockerfile", "w").write("\n".join(out))
PYEOF
  echo "--- Dockerfile after:"; cat Dockerfile

  echo; echo "=== TASK 5b: run the pipeline again - it should now succeed and deploy auth-service ==="
  write_pipeline
  gcloud builds submit . --region="$REGION" --config=cloudbuild.yaml </dev/null \
    || fail "the Task 5 build failed - send a screenshot of the first red lines and of 'cat ~/sample-app/Dockerfile.orig'"

  echo; echo "=== TASK 5c: allow unauthenticated access to auth-service (testing only) ==="
  gcloud beta run services add-iam-policy-binding --region="$REGION" --member=allUsers \
    --role=roles/run.invoker auth-service </dev/null >/dev/null || fail "could not add the run.invoker binding"
  URL=$(gcloud run services describe auth-service --region="$REGION" --format="value(status.url)" </dev/null)
  echo "   auth-service URL: $URL"
  echo "   response: $(curl -s -m 10 "$URL" | head -c 300)"
  mark 5
fi

if [ "$START_DONE" -ge 5 ]; then
  echo
  echo "All 5 tasks were already done in this lab."
  read -r -p "Re-run Task 4's failing build so its checkpoint can get the full 20 points? (y/n) [y]: " again
  if [ "${again:-y}" = "y" ]; then
    run_failing_build "$WORKDIR"
    echo
    echo ">>> Lab page: click 'Check my progress' for 'Integrate vulnerability scanning into your CI/CD pipeline' again."
  fi
  exit 0
fi

echo
echo ">>> Done. On the lab page click 'Check my progress' for any checkpoint that is not green yet."
LABEOF

```

bash securedelivery_lab.sh
