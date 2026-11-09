```/text
cat > monitor_lab.sh <<'LABEOF'
#!/bin/bash
# Universal solver: Monitor and Manage Google Cloud Resources - Challenge Lab (catalog_lab/6219)
# Run in Cloud Shell with:  bash monitor_lab.sh      (safe to re-run)

ask() { local r; read -e -r -p "$2 [$3]: " r; printf -v "$1" '%s' "${r:-$3}"; }
ask_req() {   # required value, no stale default
  local r=""
  while [ -z "$r" ]; do
    read -e -r -p "$2${3:+ [$3]}: " r; r="${r:-$3}"
    [ -z "$r" ] && echo "   (required - copy it from the yellow text)"
  done
  printf -v "$1" '%s' "$r"
}
die() { echo; echo "STOP: $1"; exit 1; }

IMG_URL="https://storage.googleapis.com/cloud-training/arc101/travel.jpg"
POLICY_NAME="Active Cloud Run Function Instances"

PROJECT_ID=$(gcloud config get-value project 2>/dev/null)
if [ -z "$PROJECT_ID" ] || [ "$PROJECT_ID" = "(unset)" ]; then
  die "no project set. Run: gcloud config set project <Project ID from the lab panel>"
fi
PROJECT_NUMBER=$(gcloud projects describe "$PROJECT_ID" --format="value(projectNumber)")
ME=$(gcloud config get-value account 2>/dev/null)
echo "Project: $PROJECT_ID ($PROJECT_NUMBER)   You: $ME"

# ---------- auto-detect defaults ----------
DET_REGION=$(gcloud config get-value compute/region 2>/dev/null)
if [ -z "$DET_REGION" ] || [ "$DET_REGION" = "(unset)" ]; then
  DET_REGION=$(gcloud compute project-info describe --quiet \
    --format="value(commonInstanceMetadata.items[google-compute-default-region])" 2>/dev/null </dev/null)
fi
[ "$DET_REGION" = "(unset)" ] && DET_REGION=""

# the "second user" = any other user:... on the project IAM policy
DET_USER2=$(gcloud projects get-iam-policy "$PROJECT_ID" --flatten="bindings[].members" \
  --format="value(bindings.members)" 2>/dev/null | grep '^user:' | sed 's/^user://' \
  | grep -vi "^${ME}$" | sort -u | head -1)

echo
echo "=== Enter the YELLOW values from the lab text (press Enter to keep the [default]) ==="
ask_req REGION   "Region ('Create all resources in the ... region')"                   "$DET_REGION"
ask_req BUCKET   "Bucket name (Task 1: 'Create a bucket called ...')"                   ""
ask_req USER2    "Second user (Task 1: Storage Object Viewer for 'Username 2' in lab panel)" "$DET_USER2"
ask_req TOPIC    "Pub/Sub topic name (Task 2)"                                          ""
ask_req FUNCTION "Cloud Run function name (Task 3: 'Create a Cloud Run function ...')"  ""
ask     ENTRY    "Entry point (Task 3: 'Entry point ... to ...')"                       "thumbnail"
ask     RUNTIME  "Runtime (Task 3 says Node.js 22 -> nodejs22)"                         "nodejs22"
ask     EMAIL    "Email for the alert (Task 4: 'your personal email', any email works)" "$ME"
ask     PAUSE    "Pause so you can click 'Check my progress'? (y/n)"                    "y"
gcloud config set compute/region "$REGION" >/dev/null 2>&1

pause() {
  if [ "$PAUSE" = "y" ]; then
    read -r -p ">>> Lab page: click 'Check my progress' for: $1 -- then press ENTER here to continue " _
  fi
}

echo
echo "=== Enabling APIs ==="
gcloud services enable cloudfunctions.googleapis.com run.googleapis.com eventarc.googleapis.com \
  pubsub.googleapis.com cloudbuild.googleapis.com artifactregistry.googleapis.com \
  storage.googleapis.com logging.googleapis.com monitoring.googleapis.com 2>/dev/null \
  || echo "   (could not enable some APIs - usually they are already on, continuing)"

# ================= Task 1 =================
echo
echo "=== Task 1: bucket gs://$BUCKET in $REGION + Storage Object Viewer for $USER2 ==="
if gcloud storage buckets describe "gs://$BUCKET" >/dev/null 2>&1; then
  echo "bucket already exists"
else
  gcloud storage buckets create "gs://$BUCKET" --location="$REGION" || die "could not create the bucket. Check the name matches the lab exactly."
fi
ok=0
for i in 1 2 3; do
  gcloud projects add-iam-policy-binding "$PROJECT_ID" --member="user:$USER2" \
    --role="roles/storage.objectViewer" --condition=None --quiet >/dev/null && { ok=1; break; }
  sleep 5
done
[ "$ok" = 1 ] && echo "   granted Storage Object Viewer to $USER2" || die "could not grant Storage Object Viewer to $USER2 (check the username)"
pause "Create a Cloud Storage bucket (25 points)"

# ================= Task 2 =================
echo
echo "=== Task 2: Pub/Sub topic $TOPIC ==="
gcloud pubsub topics describe "$TOPIC" >/dev/null 2>&1 || gcloud pubsub topics create "$TOPIC" || die "could not create the topic"
pause "Create a Pub/Sub topic (25 points)"

# ================= Task 3 =================
echo
echo "=== Permissions for the Cloud Run function ==="
gcloud beta services identity create --service=pubsub.googleapis.com --project="$PROJECT_ID" >/dev/null 2>&1 || true
gcloud beta services identity create --service=eventarc.googleapis.com --project="$PROJECT_ID" >/dev/null 2>&1 || true
GCS_SA=$(gcloud storage service-agent --project="$PROJECT_ID" 2>/dev/null)
COMPUTE_SA="${PROJECT_NUMBER}-compute@developer.gserviceaccount.com"
PUBSUB_SA="service-${PROJECT_NUMBER}@gcp-sa-pubsub.iam.gserviceaccount.com"
EVENTARC_SA="service-${PROJECT_NUMBER}@gcp-sa-eventarc.iam.gserviceaccount.com"

grant() {   # grant <member> <role>   (fast; the deploy step grants what it needs itself anyway)
  local i err
  for i in 1 2; do
    err=$(gcloud projects add-iam-policy-binding "$PROJECT_ID" --member="$1" --role="$2" \
            --condition=None --quiet 2>&1 >/dev/null) && { echo "   granted $2"; return 0; }
    echo "$err" | grep -qiE "concurrent|etag|aborted|409" || break
    sleep 5
  done
  echo "   (skipped $2 for ${1#serviceAccount:} - that's OK)"
  return 0
}
[ -n "$GCS_SA" ] && grant "serviceAccount:$GCS_SA" roles/pubsub.publisher
grant "serviceAccount:$COMPUTE_SA" roles/eventarc.eventReceiver
grant "serviceAccount:$COMPUTE_SA" roles/run.invoker
grant "serviceAccount:$COMPUTE_SA" roles/pubsub.publisher
grant "serviceAccount:$COMPUTE_SA" roles/storage.objectAdmin
grant "serviceAccount:$COMPUTE_SA" roles/artifactregistry.writer
grant "serviceAccount:$COMPUTE_SA" roles/logging.logWriter
grant "serviceAccount:$PUBSUB_SA" roles/iam.serviceAccountTokenCreator
grant "serviceAccount:$EVENTARC_SA" roles/eventarc.serviceAgent
echo "waiting 90s for the permissions to spread (the lab says to wait a few minutes)..."
sleep 90

echo
echo "=== Task 3: Cloud Run function (2nd gen) $FUNCTION, entry point '$ENTRY' ==="
mkdir -p ~/thumbnail-fn && cd ~/thumbnail-fn || die "could not create the source folder"
rm -rf node_modules package-lock.json
# Thumbnail code: same behaviour as the lab code (64x64 thumbnail + message to the topic).
# Uses 'sharp' from npm, which builds reliably on Node.js 22.
cat > index.js <<'JS1'
const functions = require('@google-cloud/functions-framework');
const { Storage } = require('@google-cloud/storage');
const { PubSub } = require('@google-cloud/pubsub');
const sharp = require('sharp');

functions.cloudEvent('ENTRY_PLACEHOLDER', async cloudEvent => {
  const event = cloudEvent.data;
  console.log(`Event: ${JSON.stringify(event)}`);

  const fileName = event.name;
  const bucketName = event.bucket;
  const size = "64x64";
  const bucket = new Storage().bucket(bucketName);
  const topicName = "TOPIC_PLACEHOLDER";
  const pubsub = new PubSub();

  if (fileName.search("64x64_thumbnail") === -1) {
    const filename_split = fileName.split('.');
    const filename_ext = filename_split[filename_split.length - 1].toLowerCase();
    const filename_without_ext = fileName.substring(0, fileName.length - filename_ext.length - 1);

    if (filename_ext === 'png' || filename_ext === 'jpg' || filename_ext === 'jpeg') {
      console.log(`Processing Original: gs://${bucketName}/${fileName}`);
      const gcsObject = bucket.file(fileName);
      const newFilename = `${filename_without_ext}_${size}_thumbnail.${filename_ext}`;
      const gcsNewObject = bucket.file(newFilename);
      try {
        const [buffer] = await gcsObject.download();
        const resizedBuffer = await sharp(buffer)
          .resize(64, 64, { fit: 'inside', withoutEnlargement: true })
          .toFormat(filename_ext === 'jpg' ? 'jpeg' : filename_ext)
          .toBuffer();
        await gcsNewObject.save(resizedBuffer, {
          metadata: { contentType: `image/${filename_ext === 'jpg' ? 'jpeg' : filename_ext}` },
        });
        console.log(`Success: ${fileName} → ${newFilename}`);
        await pubsub.topic(topicName).publishMessage({ data: Buffer.from(newFilename) });
        console.log(`Message published to ${topicName}`);
      } catch (err) {
        console.error(`Error: ${err}`);
      }
    } else {
      console.log(`gs://${bucketName}/${fileName} is not an image I can handle`);
    }
  } else {
    console.log(`gs://${bucketName}/${fileName} already has a thumbnail`);
  }
});
JS1
cat > package.json <<'JSON1'
{
  "name": "thumbnails",
  "version": "1.0.0",
  "description": "Create Thumbnail of uploaded image",
  "scripts": {
    "start": "node index.js"
  },
  "dependencies": {
    "@google-cloud/functions-framework": "^3.0.0",
    "@google-cloud/pubsub": "^2.0.0",
    "@google-cloud/storage": "^6.11.0",
    "sharp": "^0.33.5"
  }
}
JSON1
sed -i "s/TOPIC_PLACEHOLDER/${TOPIC}/; s/ENTRY_PLACEHOLDER/${ENTRY}/" index.js
grep -n "topicName =\|cloudEvent('" index.js

deployed=0
for i in 1 2 3 4 5 6; do
  echo "--- deploy attempt $i (the build takes a few minutes) ---"
  gcloud functions deploy "$FUNCTION" --gen2 --runtime="$RUNTIME" --region="$REGION" \
       --source=. --entry-point="$ENTRY" --memory=512MiB \
       --trigger-event-filters="type=google.cloud.storage.object.v1.finalized" \
       --trigger-event-filters="bucket=$BUCKET" \
       --trigger-location="$REGION" --quiet 2>&1 | tee ~/deploy_attempt.log
  if [ "${PIPESTATUS[0]}" = 0 ]; then deployed=1; break; fi
  echo
  echo "   ^^^ Attempt $i failed. 'Permission denied ... Eventarc Service Agent' on the first try is"
  echo "       expected (Google is still applying permissions). Waiting 60s and trying again..."
  sleep 60
done
[ "$deployed" = 1 ] || die "the function did not deploy. Send me the error above."
echo "function deployed; waiting 60s for the Eventarc trigger to become ready..."
sleep 60

TRIG=$(gcloud functions describe "$FUNCTION" --gen2 --region="$REGION" --format="value(eventTrigger.trigger)" 2>/dev/null)
if [ -z "$TRIG" ] && [ -z "$(gcloud eventarc triggers list --location="$REGION" --format='value(name)' 2>/dev/null)" ]; then
  echo "WARNING: no Cloud Storage trigger found - creating one (as the lab note says)"
  gcloud eventarc triggers create "${FUNCTION}-trigger" --location="$REGION" \
    --destination-run-service="$FUNCTION" --destination-run-region="$REGION" \
    --event-filters="type=google.cloud.storage.object.v1.finalized" \
    --event-filters="bucket=$BUCKET" --service-account="$COMPUTE_SA"
else
  echo "Cloud Storage trigger is in place"
fi

echo
echo "=== Task 3 test: upload a test image and wait for the thumbnail ==="
cd ~ || exit 1
curl -sSLf -o travel.jpg "$IMG_URL" || die "could not download the test image"
found=0
for n in 1 2 3; do
  gcloud storage cp travel.jpg "gs://$BUCKET/travel-test-$n.jpg"
  for w in $(seq 1 12); do
    if gcloud storage ls "gs://$BUCKET/" 2>/dev/null | grep -q "64x64_thumbnail"; then found=1; break; fi
    sleep 10
  done
  [ "$found" = 1 ] && break
  echo "   no thumbnail yet - checking why..."
  LOGS=$(gcloud functions logs read "$FUNCTION" --gen2 --region="$REGION" --limit=50 2>/dev/null)
  if ! echo "$LOGS" | grep -q "Event:"; then
    # the function was never called -> the trigger cannot deliver events; fix its permissions
    echo "   the function was not called by the trigger -> fixing trigger permissions"
    gcloud run services add-iam-policy-binding "$FUNCTION" --region="$REGION" \
      --member="serviceAccount:$COMPUTE_SA" --role=roles/run.invoker --quiet >/dev/null 2>&1 \
      && echo "   granted Cloud Run Invoker on $FUNCTION to the trigger account"
    [ -n "$GCS_SA" ] && gcloud projects add-iam-policy-binding "$PROJECT_ID" \
      --member="serviceAccount:$GCS_SA" --role=roles/pubsub.publisher --condition=None --quiet >/dev/null 2>&1 \
      && echo "   granted Pub/Sub Publisher to the Cloud Storage service agent"
    echo "   waiting 60s for that to apply..."; sleep 60
  else
    echo "   the function ran but did not make a thumbnail. Its last errors:"
    echo "$LOGS" | grep -iE "error|denied|cannot|fail" | head -n 10
  fi
  echo "   uploading the test image again..."
done
gcloud storage ls "gs://$BUCKET/"
if [ "$found" = 1 ]; then
  echo "   thumbnail created"
else
  echo "   (no thumbnail - this checkpoint still passed in your last run; function logs:)"
  gcloud functions logs read "$FUNCTION" --gen2 --region="$REGION" --limit=20 2>/dev/null
fi
pause "Create a Cloud Run Function (25 points)"

# ================= Task 4 =================
echo
echo "=== Task 4: alerting policy '$POLICY_NAME' -> $EMAIL ==="
TOKEN=$(gcloud auth print-access-token)
API="https://monitoring.googleapis.com/v3/projects/$PROJECT_ID"

CHANNEL=$(curl -s -H "Authorization: Bearer $TOKEN" "$API/notificationChannels" \
  | python3 -c "
import sys, json
for c in json.load(sys.stdin).get('notificationChannels', []):
    if c.get('type') == 'email' and c.get('labels', {}).get('email_address') == '$EMAIL':
        print(c['name']); break")
if [ -z "$CHANNEL" ]; then
  CHANNEL=$(curl -s -X POST -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
    "$API/notificationChannels" \
    -d "{\"type\":\"email\",\"displayName\":\"Alert email\",\"labels\":{\"email_address\":\"$EMAIL\"}}" \
    | python3 -c "import sys, json; print(json.load(sys.stdin).get('name',''))")
fi
[ -n "$CHANNEL" ] && echo "   email channel: $CHANNEL" || die "could not create the email notification channel"

EXISTING=$(curl -s -H "Authorization: Bearer $TOKEN" "$API/alertPolicies" \
  | python3 -c "
import sys, json
for p in json.load(sys.stdin).get('alertPolicies', []):
    if p.get('displayName') == '$POLICY_NAME':
        print(p['name']); break")
if [ -n "$EXISTING" ]; then
  echo "   alerting policy already exists: $EXISTING"
else
  cat > ~/alert_policy.json <<POL
{
  "displayName": "$POLICY_NAME",
  "combiner": "OR",
  "conditions": [{
    "displayName": "Cloud Function - Active instances",
    "conditionThreshold": {
      "filter": "resource.type = \"cloud_function\" AND metric.type = \"cloudfunctions.googleapis.com/function/active_instances\"",
      "aggregations": [{"alignmentPeriod": "60s", "perSeriesAligner": "ALIGN_MEAN"}],
      "comparison": "COMPARISON_GT",
      "thresholdValue": 0,
      "duration": "0s",
      "trigger": {"count": 1}
    }
  }],
  "notificationChannels": ["$CHANNEL"],
  "enabled": true
}
POL
  RES=$(curl -s -X POST -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
    "$API/alertPolicies" -d @"$HOME/alert_policy.json")
  echo "$RES" | grep -q '"name"' && echo "   created alerting policy '$POLICY_NAME'" \
    || { echo "$RES"; die "could not create the alerting policy (error above)"; }
fi
pause "Create an alerting policy (25 points)"

# ================= Self-check =================
echo
echo "=== Self-check ==="
check() { if eval "$1"; then echo "  PASS  $2"; else echo "  FAIL  $2"; fi; }
check '[ "$(gcloud storage buckets describe gs://$BUCKET --format="value(location)" 2>/dev/null | tr a-z A-Z)" = "$(echo $REGION | tr a-z A-Z)" ]' "Task 1: bucket gs://$BUCKET is in $REGION"
check 'gcloud projects get-iam-policy "$PROJECT_ID" --flatten="bindings[].members" --filter="bindings.role=roles/storage.objectViewer" --format="value(bindings.members)" | grep -qi "user:$USER2"' "Task 1: $USER2 has Storage Object Viewer"
check 'gcloud pubsub topics describe "$TOPIC" >/dev/null 2>&1' "Task 2: topic $TOPIC exists"
check '[ "$(gcloud functions describe "$FUNCTION" --gen2 --region="$REGION" --format="value(state)" 2>/dev/null)" = "ACTIVE" ]' "Task 3: function $FUNCTION is ACTIVE"
check 'gcloud storage ls "gs://$BUCKET/" 2>/dev/null | grep -q "64x64_thumbnail"' "Task 3: a thumbnail exists in the bucket"
check 'curl -s -H "Authorization: Bearer $(gcloud auth print-access-token)" "$API/alertPolicies" | grep -q "\"$POLICY_NAME\""' "Task 4: alerting policy '$POLICY_NAME' exists"
echo
echo ">>> Done. 4 checkpoints on the lab page: bucket, topic, Cloud Run function, alerting policy (25 each)."
LABEOF
```



bash monitor_lab.sh
