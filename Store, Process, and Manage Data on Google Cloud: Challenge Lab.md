```text
cat > lab7.sh <<'LABEOF'
#!/bin/bash
# Universal solver (v2): Store, Process, and Manage Data on Google Cloud - Challenge Lab (ARC100)
# Run in Cloud Shell with:  bash lab7.sh

ask() { local r; read -e -r -p "$2 [$3]: " r; printf -v "$1" '%s' "${r:-$3}"; }
die() { echo; echo "STOP: $1"; exit 1; }

# Fixed by the lab text (not yellow)
IMG_URL="https://storage.googleapis.com/cloud-training/arc101/travel.jpg"

PROJECT_ID=$(gcloud config get-value project 2>/dev/null)
if [ -z "$PROJECT_ID" ] || [ "$PROJECT_ID" = "(unset)" ]; then
  die "no project set. Run: gcloud config set project <Project ID from the lab panel>"
fi
PROJECT_NUMBER=$(gcloud projects describe "$PROJECT_ID" --format="value(projectNumber)")
echo "Project: $PROJECT_ID ($PROJECT_NUMBER)"

# ---------- detect the lab region ----------
DET_REGION=$(gcloud config get-value compute/region 2>/dev/null)
if [ -z "$DET_REGION" ] || [ "$DET_REGION" = "(unset)" ]; then
  DET_REGION=$(gcloud compute project-info describe --quiet \
    --format="value(commonInstanceMetadata.items[google-compute-default-region])" 2>/dev/null </dev/null)
fi
if [ -z "$DET_REGION" ] || [ "$DET_REGION" = "(unset)" ]; then
  echo "Could not detect the region - type the one from the lab text."
  DET_REGION="us-east1"
fi

echo
echo "=== Enter the YELLOW values from the lab text (press Enter to keep the default) ==="
ask REGION   "Region ('Create all resources in the ... region')"      "$DET_REGION"
ask ZONE     "Zone (Task 3: '... region and ... zone')"               "${REGION}-c"
ask BUCKET   "Bucket name (Task 1)"                                   "memories-bucket-${PROJECT_ID}"
ask TOPIC    "Pub/Sub topic name (Task 2)"                            "memories-topic-403"
ask FUNCTION "Cloud Run function name (Task 3, also the entry point)" "memories-thumbnail-generator"
ask RUNTIME  "Runtime (lab says Node.js 22 -> nodejs22)"              "nodejs22"
ask PAUSE    "Pause so you can click 'Check my progress'? (y/n)"      "y"
gcloud config set compute/region "$REGION" >/dev/null 2>&1
gcloud config set compute/zone "$ZONE" >/dev/null 2>&1

pause() {
  if [ "$PAUSE" = "y" ]; then
    read -r -p ">>> Lab page: click 'Check my progress' for: $1 -- then press ENTER here to continue " _
  fi
}

echo
echo "=== Enabling APIs ==="
gcloud services enable cloudfunctions.googleapis.com run.googleapis.com eventarc.googleapis.com \
  pubsub.googleapis.com cloudbuild.googleapis.com artifactregistry.googleapis.com \
  storage.googleapis.com logging.googleapis.com

echo
echo "=== Task 1: bucket gs://$BUCKET in $REGION ==="
if gcloud storage buckets describe "gs://$BUCKET" >/dev/null 2>&1; then
  echo "bucket already exists"
else
  gcloud storage buckets create "gs://$BUCKET" --location="$REGION" || die "could not create the bucket. Check the name matches the lab exactly."
fi
pause "Create a bucket (30 points)"

echo
echo "=== Task 2: Pub/Sub topic $TOPIC ==="
gcloud pubsub topics describe "$TOPIC" >/dev/null 2>&1 || gcloud pubsub topics create "$TOPIC" || die "could not create the topic"
pause "Create a Pub/Sub topic (30 points)"

echo
echo "=== Permissions for the Cloud Run function ==="
gcloud services identity create --service=pubsub.googleapis.com --project="$PROJECT_ID" >/dev/null 2>&1 || \
  gcloud beta services identity create --service=pubsub.googleapis.com --project="$PROJECT_ID" >/dev/null 2>&1 || true
gcloud services identity create --service=eventarc.googleapis.com --project="$PROJECT_ID" >/dev/null 2>&1 || \
  gcloud beta services identity create --service=eventarc.googleapis.com --project="$PROJECT_ID" >/dev/null 2>&1 || true

GCS_SA=$(gcloud storage service-agent --project="$PROJECT_ID" 2>/dev/null)
COMPUTE_SA="${PROJECT_NUMBER}-compute@developer.gserviceaccount.com"
PUBSUB_SA="service-${PROJECT_NUMBER}@gcp-sa-pubsub.iam.gserviceaccount.com"
EVENTARC_SA="service-${PROJECT_NUMBER}@gcp-sa-eventarc.iam.gserviceaccount.com"

grant() {   # grant <member> <role>
  local i
  for i in 1 2 3 4 5; do
    gcloud projects add-iam-policy-binding "$PROJECT_ID" --member="$1" --role="$2" --condition=None >/dev/null 2>&1 && { echo "   granted $2"; return 0; }
    sleep 8
  done
  echo "   (warning: could not grant $2 to $1)"; return 1
}
[ -n "$GCS_SA" ] && grant "serviceAccount:$GCS_SA" roles/pubsub.publisher
grant "serviceAccount:$COMPUTE_SA" roles/eventarc.eventReceiver
grant "serviceAccount:$COMPUTE_SA" roles/run.invoker
grant "serviceAccount:$COMPUTE_SA" roles/pubsub.publisher
grant "serviceAccount:$COMPUTE_SA" roles/storage.objectAdmin
grant "serviceAccount:$COMPUTE_SA" roles/artifactregistry.writer
grant "serviceAccount:$COMPUTE_SA" roles/logging.logWriter
grant "serviceAccount:$COMPUTE_SA" roles/cloudbuild.builds.builder
grant "serviceAccount:$PUBSUB_SA" roles/iam.serviceAccountTokenCreator
grant "serviceAccount:$EVENTARC_SA" roles/eventarc.serviceAgent
echo "waiting 90s for the permissions to spread (the lab says to wait a few minutes)..."
sleep 90

echo
echo "=== Task 3: Cloud Run function (2nd gen) $FUNCTION ==="
mkdir -p ~/thumbnail-function && cd ~/thumbnail-function || die "could not create the source folder"
cat > index.js <<'JS1'
const functions = require('@google-cloud/functions-framework');
const { Storage } = require('@google-cloud/storage');
const { PubSub } = require('@google-cloud/pubsub');
const sharp = require('sharp');

functions.cloudEvent('FUNCTION_NAME_PLACEHOLDER', async cloudEvent => {
  const event = cloudEvent.data;

  console.log(`Event: ${JSON.stringify(event)}`);
  console.log(`Hello ${event.bucket}`);

  const fileName = event.name;
  const bucketName = event.bucket;
  const size = "64x64";
  const bucket = new Storage().bucket(bucketName);
  const topicName = "TOPIC_NAME_PLACEHOLDER";
  const pubsub = new PubSub();

  if (fileName.search("64x64_thumbnail") === -1) {
    // doesn't have a thumbnail, get the filename extension
    const filename_split = fileName.split('.');
    const filename_ext = filename_split[filename_split.length - 1].toLowerCase();
    const filename_without_ext = fileName.substring(0, fileName.length - filename_ext.length - 1); // fix sub string to remove the dot

    if (filename_ext === 'png' || filename_ext === 'jpg' || filename_ext === 'jpeg') {
      // only support png and jpg at this point
      console.log(`Processing Original: gs://${bucketName}/${fileName}`);
      const gcsObject = bucket.file(fileName);
      const newFilename = `${filename_without_ext}_64x64_thumbnail.${filename_ext}`;
      const gcsNewObject = bucket.file(newFilename);

      try {
        const [buffer] = await gcsObject.download();
        const resizedBuffer = await sharp(buffer)
          .resize(64, 64, {
            fit: 'inside',
            withoutEnlargement: true,
          })
          .toFormat(filename_ext)
          .toBuffer();

        await gcsNewObject.save(resizedBuffer, {
          metadata: {
            contentType: `image/${filename_ext}`,
          },
        });

        console.log(`Success: ${fileName} → ${newFilename}`);

        await pubsub
          .topic(topicName)
          .publishMessage({ data: Buffer.from(newFilename) });

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
   "sharp": "^0.32.1"
 },
 "devDependencies": {},
 "engines": {
   "node": ">=4.3.2"
 }
}
JSON1
sed -i "s/TOPIC_NAME_PLACEHOLDER/${TOPIC}/; s/FUNCTION_NAME_PLACEHOLDER/${FUNCTION}/" index.js
grep -n "topicName =\|cloudEvent('" index.js

deployed=0
for i in 1 2 3 4; do
  echo "--- deploy attempt $i (the build takes a few minutes) ---"
  if gcloud functions deploy "$FUNCTION" --gen2 --runtime="$RUNTIME" --region="$REGION" \
       --source=. --entry-point="$FUNCTION" --memory=512MiB \
       --trigger-event-filters="type=google.cloud.storage.object.v1.finalized" \
       --trigger-event-filters="bucket=$BUCKET" \
       --trigger-location="$REGION" --quiet; then
    deployed=1; break
  fi
  echo "   deploy failed, waiting 60s for permissions and trying again..."; sleep 60
done
[ "$deployed" = 1 ] || die "the function did not deploy. Send me the error above."
echo "function deployed; waiting 60s for the Eventarc trigger to become ready..."
sleep 60

# The lab says to check the Triggers tab and recreate the trigger if it is missing
TRIG=$(gcloud functions describe "$FUNCTION" --gen2 --region="$REGION" --format="value(eventTrigger.trigger)" 2>/dev/null)
TRIG_LIST=$(gcloud eventarc triggers list --location="$REGION" --format="value(name)" 2>/dev/null)
if [ -z "$TRIG" ] && [ -z "$TRIG_LIST" ]; then
  echo "WARNING: no Cloud Storage trigger found - creating one"
  gcloud eventarc triggers create "${FUNCTION}-trigger" --location="$REGION" \
    --destination-run-service="$FUNCTION" --destination-run-region="$REGION" \
    --event-filters="type=google.cloud.storage.object.v1.finalized" \
    --event-filters="bucket=$BUCKET" --service-account="$COMPUTE_SA"
else
  echo "Cloud Storage trigger is in place"
fi

echo
echo "=== Task 4: upload a test image and wait for the thumbnail ==="
cd ~ || exit 1
curl -sSLf -o travel.jpg "$IMG_URL" || die "could not download the test image"
found=0
for n in 1 2 3; do
  gcloud storage cp travel.jpg "gs://$BUCKET/memories-test-$n.jpg"
  for w in $(seq 1 12); do
    if gcloud storage ls "gs://$BUCKET/" 2>/dev/null | grep -q "_64x64_thumbnail"; then found=1; break; fi
    sleep 10
  done
  [ "$found" = 1 ] && break
  echo "   no thumbnail yet, uploading the test image again..."
done
gcloud storage ls "gs://$BUCKET/"
[ "$found" = 1 ] || echo "(no thumbnail yet - see the log hint at the end)"
pause "Create the Cloud Function / Verify the thumbnail (40 points)"

echo
echo "=== Self-check ==="
check() { if eval "$1"; then echo "  PASS  $2"; else echo "  FAIL  $2"; fi; }
check '[ "$(gcloud storage buckets describe gs://$BUCKET --format="value(location)" 2>/dev/null | tr a-z A-Z)" = "$(echo $REGION | tr a-z A-Z)" ]' "Task 1: bucket gs://$BUCKET is in $REGION"
check 'gcloud pubsub topics describe "$TOPIC" >/dev/null 2>&1' "Task 2: topic $TOPIC exists"
check '[ "$(gcloud functions describe "$FUNCTION" --gen2 --region="$REGION" --format="value(state)" 2>/dev/null)" = "ACTIVE" ]' "Task 3: function $FUNCTION is ACTIVE"
check 'gcloud functions describe "$FUNCTION" --gen2 --region="$REGION" --format="value(eventTrigger.eventFilters)" 2>/dev/null | grep -q "$BUCKET" || gcloud eventarc triggers list --location="$REGION" --format="value(name)" 2>/dev/null | grep -q .' "Task 3: a Cloud Storage trigger exists for $BUCKET"
check 'gcloud storage ls "gs://$BUCKET/" 2>/dev/null | grep -q "_64x64_thumbnail"' "Task 4: a thumbnail exists in the bucket"
echo
echo "If a check failed, see the function logs with:"
echo "  gcloud functions logs read $FUNCTION --gen2 --region=$REGION --limit=20"
echo
echo ">>> Done. On the lab page there are 3 checkpoints: bucket (30), topic (30), and the Cloud Function check (40, after the test upload)."
LABEOF
```

bash lab7.sh
