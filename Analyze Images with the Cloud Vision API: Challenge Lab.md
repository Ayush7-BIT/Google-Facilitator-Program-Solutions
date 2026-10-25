## task 1

```text
vision_lab() {
  PROJECT=$(gcloud config get-value project 2>/dev/null)
  [ -z "$PROJECT" ] && { echo "STOP: no project set. Run: gcloud config set project <Project ID from the lab panel>"; return 1; }

  echo "=== Task 1: API key, bucket, public image ==="
  gcloud services enable apikeys.googleapis.com vision.googleapis.com translate.googleapis.com language.googleapis.com

  get_key() {
    local n
    n=$(gcloud services api-keys list --filter="displayName=lab-key" --format="value(name)" 2>/dev/null | head -1)
    [ -n "$n" ] && gcloud services api-keys get-key-string "$n" --format="value(keyString)" 2>/dev/null
  }
  API_KEY=$(get_key)
  if [ ${#API_KEY} -ne 39 ]; then
    gcloud services api-keys create --display-name="lab-key" \
      --api-target=service=vision.googleapis.com \
      --api-target=service=translate.googleapis.com \
      --api-target=service=language.googleapis.com
    sleep 10
    API_KEY=$(get_key)
  fi
  if [ ${#API_KEY} -ne 39 ]; then   # last resort: any key already in the project
    local n; n=$(gcloud services api-keys list --format="value(name)" 2>/dev/null | head -1)
    [ -n "$n" ] && API_KEY=$(gcloud services api-keys get-key-string "$n" --format="value(keyString)")
  fi
  [ ${#API_KEY} -ne 39 ] && { echo "STOP: no API key. Create one in Console > APIs & Services > Credentials, then run vision_lab again."; return 1; }
  export API_KEY
  grep -q "^export API_KEY=" ~/.bashrc 2>/dev/null || echo "export API_KEY=$API_KEY" >> ~/.bashrc
  echo "API_KEY ready (length ${#API_KEY})"

  BUCKET="${1:-$(gcloud storage buckets list --format='value(name)' 2>/dev/null | grep -m1 -- '-bucket$')}"
  BUCKET="${BUCKET:-${PROJECT}-bucket}"
  IMG_URI="${2:-$(gcloud storage ls "gs://$BUCKET" 2>/dev/null | grep -iE '\.(jpe?g|png|gif|bmp|webp|tiff?)$' | head -1)}"
  [ -z "$IMG_URI" ] && IMG_URI=$(gcloud storage ls "gs://$BUCKET" 2>/dev/null | grep -v '/$' | head -1)
  [ -z "$IMG_URI" ] && { echo "STOP: no image found in gs://$BUCKET"; return 1; }
  echo "Bucket: $BUCKET"
  echo "Image : $IMG_URI"

  gcloud storage objects update "$IMG_URI" --predefined-acl=publicRead 2>/dev/null \
    || gcloud storage buckets add-iam-policy-binding "gs://$BUCKET" --member=allUsers --role=roles/storage.objectViewer
  echo "Public check (want 200): $(curl -s -o /dev/null -w '%{http_code}' "https://storage.googleapis.com/${IMG_URI#gs://}")"

  annotate() {   # annotate <output file>
    for i in 1 2 3 4 5 6; do
      curl -s -X POST -H "Content-Type: application/json" --data-binary @request.json \
        "https://vision.googleapis.com/v1/images:annotate?key=${API_KEY}" -o "$1"
      grep -q '"error": *{' "$1" || return 0
      echo "  attempt $i failed, retrying in 10s (a new key or API can take a minute)..."
      sleep 10
    done
    return 1
  }

  echo "=== Task 2: request.json ==="
cat > request.json <<EOF
{
  "requests": [
    {
      "image": { "source": { "gcsImageUri": "${IMG_URI}" } },
      "features": [ { "type": "TEXT_DETECTION", "maxResults": 10 } ]
    }
  ]
}
EOF

  echo "=== Task 3a: TEXT_DETECTION ==="
  annotate text-response.json || { echo "STOP: Vision call failed:"; head -c 600 text-response.json; return 1; }
  gcloud storage cp text-response.json "gs://$BUCKET"

  echo "=== Task 3b: LANDMARK_DETECTION ==="
  sed -i 's/TEXT_DETECTION/LANDMARK_DETECTION/' request.json
  annotate landmark-response.json || { echo "STOP: Vision call failed:"; head -c 600 landmark-response.json; return 1; }
  gcloud storage cp landmark-response.json "gs://$BUCKET"

  echo; echo "=== Result ==="
  echo "Text found    : $(grep -m1 '"description"' text-response.json | cut -c1-120)"
  echo "Landmark found: $(grep -m1 '"description"' landmark-response.json | cut -c1-120)  (blank = none in this image, which is fine)"
  gcloud storage ls "gs://$BUCKET"
  echo ">>> Now click 'Check my progress' on every task in the lab page."
}
vision_lab
```
