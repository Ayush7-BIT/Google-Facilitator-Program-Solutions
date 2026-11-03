```/text
cat > monitoring_lab.sh <<'LABEOF'
#!/bin/bash
# Monitoring and Logging for Cloud Run Functions (GSP092) - universal solution

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
ask "Function / service name                  " "helloworld" SERVICE
ask "Runtime                                  " "nodejs22" RUNTIME
ask "Maximum number of instances              " "5" MAX_INSTANCES
ask "Logs-based metric name                   " "CloudRunFunctionLatency-Logs" METRIC_NAME
ask "Log field to extract                     " "httpRequest.latency" METRIC_FIELD
ask "Test traffic duration                    " "300s" DURATION
ask "Test traffic rate (requests/second)      " "200" RATE
ask "Also build the Task 4 dashboard (not scored)? (y/n)" "y" DASH
ask "Dashboard name                           " "Cloud Run Function Custom Dashboard" DASH_NAME
ask "Pause so you can click 'Check my progress'? (y/n)" "y" PAUSE

SRC_DIR="$HOME/${SERVICE}-fn"
PN=$(gcloud projects describe "$PROJECT_ID" --format="value(projectNumber)" 2>/dev/null)

pause() {
  if [ "$PAUSE" = "y" ]; then
    read -r -p ">>> Lab page: click 'Check my progress' for: $1 -- then press ENTER here to continue " _
  fi
}

fix_build_permissions() {
  echo "Granting build roles to the default service accounts (only done after a failure)..."
  local sa role
  for sa in "${PN}-compute@developer.gserviceaccount.com" "${PN}@cloudbuild.gserviceaccount.com"; do
    for role in roles/cloudbuild.builds.builder roles/artifactregistry.writer roles/storage.objectAdmin roles/logging.logWriter; do
      gcloud projects add-iam-policy-binding "$PROJECT_ID" \
        --member="serviceAccount:${sa}" --role="$role" --quiet >/dev/null 2>&1
    done
  done
  sleep 20
}

deploy_function() {
  local i
  for i in 1 2 3; do
    if gcloud functions deploy "$SERVICE" --gen2 --runtime="$RUNTIME" --region="$REGION" \
        --source="$SRC_DIR" --entry-point=helloWorld --trigger-http --allow-unauthenticated \
        --max-instances="$MAX_INSTANCES" --quiet; then
      return 0
    fi
    echo "Deploy failed (try $i/3)."
    [ "$i" -eq 1 ] && fix_build_permissions
    sleep 20
  done
  return 1
}

# ---------- TASK 1: create the Cloud Run function ----------
echo
echo "=== TASK 1: create the Cloud Run function ==="
gcloud services enable run.googleapis.com cloudfunctions.googleapis.com cloudbuild.googleapis.com \
  artifactregistry.googleapis.com logging.googleapis.com monitoring.googleapis.com --quiet

mkdir -p "$SRC_DIR"
cat > "$SRC_DIR/index.js" <<'JSEOF'
const functions = require('@google-cloud/functions-framework');

functions.http('helloWorld', (req, res) => {
  res.status(200).send('Hello World!');
});
JSEOF
cat > "$SRC_DIR/package.json" <<'JSONEOF'
{
  "name": "helloworld",
  "version": "1.0.0",
  "main": "index.js",
  "dependencies": {
    "@google-cloud/functions-framework": "^3.4.0"
  }
}
JSONEOF

echo "Deploying (takes a few minutes)..."
deploy_function || { echo "FAIL: could not deploy the function"; exit 1; }

# Execution environment: second generation (as in the lab)
gcloud run services update "$SERVICE" --region "$REGION" --execution-environment=gen2 \
  --max-instances="$MAX_INSTANCES" --quiet || echo "(could not set gen2 explicitly - continuing)"

CLOUD_RUN_URL=$(gcloud run services describe "$SERVICE" --region="$REGION" --format='value(status.url)')
echo "Service URL: $CLOUD_RUN_URL"
for i in 1 2 3 4 5 6; do
  HELLO=$(curl -s "$CLOUD_RUN_URL")
  echo "$HELLO" | grep -q "Hello World" && break
  echo "not answering yet (try $i/6) - waiting 10s"; sleep 10
done
echo "GET $CLOUD_RUN_URL -> $HELLO"
pause "Creating a Cloud Function"

# ---------- send test traffic in the background ----------
echo
echo "=== Test traffic with vegeta (runs in the background for ${DURATION}) ==="
cd "$HOME" || exit 1
if [ ! -x ./vegeta ]; then
  curl -fLO 'https://github.com/tsenart/vegeta/releases/download/v12.12.0/vegeta_12.12.0_linux_386.tar.gz' \
    && tar -xzf vegeta_12.12.0_linux_386.tar.gz
fi
if [ -x ./vegeta ]; then
  nohup bash -c "echo 'GET ${CLOUD_RUN_URL}' | ./vegeta attack -duration=${DURATION} -rate=${RATE} > results.bin" >/dev/null 2>&1 &
  sleep 3
  pgrep -f "vegeta attack" >/dev/null && echo "PASS  vegeta is sending traffic (keep this Cloud Shell tab open)" || echo "CHECK vegeta did not start"
else
  echo "CHECK vegeta download failed - the checks do not need it, but Metrics Explorer will have no data"
fi
echo "Waiting 45s so request logs appear before the metric is created..."
sleep 45

# ---------- TASK 2: logs-based metric ----------
echo
echo "=== TASK 2: create the logs-based metric ==="
cat > metric.json <<EOF
{
  "name": "${METRIC_NAME}",
  "description": "Latency of Cloud Run function requests (from ${METRIC_FIELD})",
  "filter": "resource.type=\"cloud_run_revision\" AND resource.labels.service_name=\"${SERVICE}\"",
  "valueExtractor": "EXTRACT(${METRIC_FIELD})",
  "metricDescriptor": {
    "metricKind": "DELTA",
    "valueType": "DISTRIBUTION",
    "unit": "s",
    "displayName": "${METRIC_NAME}"
  },
  "bucketOptions": {
    "exponentialBuckets": {"numFiniteBuckets": 64, "growthFactor": 2, "scale": 0.01}
  }
}
EOF

if gcloud logging metrics describe "$METRIC_NAME" >/dev/null 2>&1; then
  echo "Metric ${METRIC_NAME} already exists"
else
  gcloud logging metrics create "$METRIC_NAME" --config-from-file=metric.json --quiet \
  || curl -s -X POST -H "Authorization: Bearer $(gcloud auth print-access-token)" \
       -H "Content-Type: application/json" -d @metric.json \
       "https://logging.googleapis.com/v2/projects/${PROJECT_ID}/metrics"
fi
echo "Metric type: $(gcloud logging metrics describe "$METRIC_NAME" --format='value(metricDescriptor.valueType)' 2>/dev/null)"
pause "Create logs-based metric"

# ---------- TASK 4 (optional, not scored): dashboard ----------
if [ "$DASH" = "y" ]; then
  echo
  echo "=== TASK 4 (not scored): custom dashboard ==="
  cat > dashboard.json <<EOF
{
  "displayName": "${DASH_NAME}",
  "mosaicLayout": {
    "columns": 12,
    "tiles": [
      {"width": 6, "height": 4, "widget": {"title": "Cloud Run Revision - Request Count",
        "xyChart": {"dataSets": [{"plotType": "STACKED_BAR",
          "timeSeriesQuery": {"timeSeriesFilter": {"filter": "metric.type=\"run.googleapis.com/request_count\" resource.type=\"cloud_run_revision\"",
            "aggregation": {"alignmentPeriod": "60s", "perSeriesAligner": "ALIGN_RATE"}}}}]}}},
      {"xPos": 6, "width": 6, "height": 4, "widget": {"title": "${METRIC_NAME}",
        "xyChart": {"dataSets": [{"plotType": "HEATMAP",
          "timeSeriesQuery": {"timeSeriesFilter": {"filter": "metric.type=\"logging.googleapis.com/user/${METRIC_NAME}\" resource.type=\"cloud_run_revision\"",
            "aggregation": {"alignmentPeriod": "60s", "perSeriesAligner": "ALIGN_DELTA", "crossSeriesReducer": "REDUCE_SUM"}}}}]}}},
      {"yPos": 4, "width": 6, "height": 4, "widget": {"title": "Cloud Run Revision - Request Latency (mean)",
        "xyChart": {"dataSets": [{"plotType": "LINE",
          "timeSeriesQuery": {"timeSeriesFilter": {"filter": "metric.type=\"run.googleapis.com/request_latencies\" resource.type=\"cloud_run_revision\"",
            "aggregation": {"alignmentPeriod": "60s", "perSeriesAligner": "ALIGN_DELTA", "crossSeriesReducer": "REDUCE_MEAN"}}}}]}}},
      {"xPos": 6, "yPos": 4, "width": 6, "height": 4, "widget": {"title": "Cloud Run Revision - Container CPU Allocation",
        "xyChart": {"dataSets": [{"plotType": "STACKED_BAR",
          "timeSeriesQuery": {"timeSeriesFilter": {"filter": "metric.type=\"run.googleapis.com/container/cpu/allocation_time\" resource.type=\"cloud_run_revision\"",
            "aggregation": {"alignmentPeriod": "60s", "perSeriesAligner": "ALIGN_RATE"}}}}]}}}
    ]
  }
}
EOF
  if gcloud monitoring dashboards list --filter="displayName=\"${DASH_NAME}\"" --format="value(name)" 2>/dev/null | grep -q .; then
    echo "Dashboard '${DASH_NAME}' already exists"
  else
    gcloud monitoring dashboards create --config-from-file=dashboard.json --quiet \
    || curl -s -X POST -H "Authorization: Bearer $(gcloud auth print-access-token)" \
         -H "Content-Type: application/json" -d @dashboard.json \
         "https://monitoring.googleapis.com/v1/projects/${PROJECT_ID}/dashboards"
  fi
fi

# ---------- SELF-CHECK ----------
echo
echo "=================== SELF-CHECK ==================="
gcloud run services describe "$SERVICE" --region "$REGION" >/dev/null 2>&1 \
  && echo "PASS  Cloud Run function/service '${SERVICE}' exists in ${REGION}" || echo "FAIL  service missing"
gcloud run services describe "$SERVICE" --region "$REGION" --format=json 2>/dev/null | grep -q "\"autoscaling.knative.dev/maxScale\": \"${MAX_INSTANCES}\"" \
  && echo "PASS  max instances = ${MAX_INSTANCES}" || echo "CHECK could not confirm max instances = ${MAX_INSTANCES}"
curl -s "$CLOUD_RUN_URL" | grep -q "Hello World" && echo "PASS  public URL returns Hello World!" || echo "FAIL  URL does not return Hello World!"
[ "$(gcloud logging metrics describe "$METRIC_NAME" --format='value(metricDescriptor.valueType)' 2>/dev/null)" = "DISTRIBUTION" ] \
  && echo "PASS  logs-based metric ${METRIC_NAME} exists (Distribution)" || echo "FAIL  metric missing or not Distribution"
gcloud logging metrics describe "$METRIC_NAME" --format='value(valueExtractor)' 2>/dev/null | grep -q "$METRIC_FIELD" \
  && echo "PASS  metric extracts ${METRIC_FIELD}" || echo "CHECK metric field extractor not confirmed"
echo
echo "Now click 'Check my progress' for any checkpoint that is not green yet."
echo "Optional (not scored): Monitoring > Metrics explorer, search ${METRIC_NAME}; Monitoring > Dashboards to see the dashboard."
LABEOF
```


bash monitoring_lab.sh
