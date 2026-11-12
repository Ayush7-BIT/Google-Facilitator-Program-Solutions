```/text
cat > observability_lab.sh <<'LABEOF'
#!/bin/bash
# Monitor and Log with Google Cloud Observability: Challenge Lab (GSP338)
# Run in Cloud Shell:  bash observability_lab.sh      (safe to re-run)
# Everything runs from Cloud Shell - nothing to open by hand.

ask() { local r; read -e -r -p "$2 [$3]: " r; printf -v "$1" '%s' "${r:-$3}"; }
ask_req() { local r=""; while [ -z "$r" ]; do read -e -r -p "$2${3:+ [$3]}: " r; r="${r:-$3}"; \
  [ -z "$r" ] && echo "   (required - copy it from the lab text)"; done; printf -v "$1" '%s' "$r"; }
die() { echo; echo "STOP: $1"; exit 1; }
pause() { read -r -p ">>> Lab page: click 'Check my progress' for: $1 -- then press ENTER here " _; }

PROJECT_ID=$(gcloud config get-value project 2>/dev/null)
[ -z "$PROJECT_ID" ] || [ "$PROJECT_ID" = "(unset)" ] && die "run: gcloud config set project <Project ID from the lab panel>"
echo "Project: $PROJECT_ID"
VM=video-queue-monitor
ZONE=$(gcloud compute instances list --filter="name=$VM" --format="value(zone.basename())" 2>/dev/null | head -1)
[ -n "$ZONE" ] || die "instance $VM not found - wait a minute for the lab to finish provisioning and re-run"
INSTANCE_ID=$(gcloud compute instances describe "$VM" --zone="$ZONE" --format="value(id)")
echo "Instance: $VM  zone: $ZONE  id: $INSTANCE_ID"

echo
echo "=== Enter the values from the lab text (Enter keeps the [default]) ==="
ask_req METRIC    "Task 3: log-based metric name (yellow, 'custom log based metric called ...')" ""
ask_req THRESHOLD "Task 5: alert threshold ('... exceeds a rate of ... per second')"        ""
case "$THRESHOLD" in ""|*[!0-9.]*) die "threshold must be a number, e.g. 3";; esac
ask     ALERT     "Task 5: alert policy name (any name is fine unless the lab gives one)"   "Large Video Upload Rate Alert"

# ===================== TASK 1 =====================
echo; echo "=== Task 1: enable Cloud Monitoring ==="
gcloud services enable monitoring.googleapis.com logging.googleapis.com
gcloud monitoring dashboards list --format="value(displayName)" 2>/dev/null | grep -q "Media_Dashboard" \
  && echo "   Media_Dashboard is available" || echo "   (Media_Dashboard not listed yet - it appears shortly)"
pause "Configure Cloud Monitoring (20 pts)"

# ===================== TASK 2 =====================
echo; echo "=== Task 2: fix the startup script of $VM so it writes input_queue_size ==="
KEY=$(gcloud compute instances describe "$VM" --zone="$ZONE" --format="value(metadata.items[].key)" | tr ';' '\n' | grep -m1 -E '^startup[-_]script$')
KEY=${KEY:-startup-script}
gcloud compute instances describe "$VM" --zone="$ZONE" --format=json | KEY="$KEY" python3 -c "
import sys,json,os
for i in json.load(sys.stdin)['metadata'].get('items',[]):
    if i['key']==os.environ['KEY']: sys.stdout.write(i['value'])" > ~/startup_orig.sh
grep -q "MY_PROJECT_ID" ~/startup_orig.sh || die "could not read the startup script (key $KEY) - send Claude a screenshot"
cp ~/startup_orig.sh ~/startup_new.sh
# 1) fill in the three environment variables the Go app needs
sed -i -E "s|^(\s*(export )?MY_PROJECT_ID=).*|\1$PROJECT_ID|; s|^(\s*(export )?MY_GCE_INSTANCE_ID=).*|\1$INSTANCE_ID|; s|^(\s*(export )?MY_GCE_INSTANCE_ZONE=).*|\1$ZONE|" ~/startup_new.sh
# 2) install the Ops (monitoring) agent if the script does not do it already
if ! grep -qiE "ops-agent|stackdriver-agent|monitoring agent" ~/startup_new.sh; then
  python3 - <<'PY'
import os
p = os.path.expanduser("~/startup_new.sh"); lines = open(p).read().split("\n")
agent = ["# Install the Ops Agent (Cloud Monitoring agent)",
         "curl -sSO https://dl.google.com/cloudagents/add-google-cloud-ops-agent-repo.sh",
         "sudo bash add-google-cloud-ops-agent-repo.sh --also-install",
         "sudo service google-cloud-ops-agent start", ""]
idx = 1 if lines and lines[0].startswith("#!") else 0
lines[idx:idx] = agent
open(p, "w").write("\n".join(lines))
PY
fi
echo "   --- changes to the startup script ---"
diff -u ~/startup_orig.sh ~/startup_new.sh
# the VM only runs the key 'startup-script', so write that one (and the original key if it differs)
EXTRA=""; [ "$KEY" != "startup-script" ] && EXTRA=",$KEY=$HOME/startup_new.sh"
gcloud compute instances add-metadata "$VM" --zone="$ZONE" --metadata-from-file="startup-script=$HOME/startup_new.sh$EXTRA" \
  || die "could not update the startup script"
echo "   restarting $VM so the new startup script runs..."
gcloud compute instances stop "$VM" --zone="$ZONE" --quiet
gcloud compute instances start "$VM" --zone="$ZONE" --quiet
echo "   waiting for the custom metric input_queue_size (the VM installs Go + agent, about 3-6 min)..."
TOKEN=$(gcloud auth print-access-token)
for i in $(seq 1 40); do
  if curl -s -H "Authorization: Bearer $(gcloud auth print-access-token)" \
     "https://monitoring.googleapis.com/v3/projects/$PROJECT_ID/metricDescriptors?filter=metric.type%3Dstarts_with(%22custom.googleapis.com/opencensus/my.videoservice.org%22)" \
     | grep -q "input_queue_size"; then echo "   OK: custom metric input_queue_size exists"; break; fi
  echo "   not there yet ($i/40) - waiting 20s"; sleep 20
done
pause "Configure a Compute Instance to generate Custom Cloud Monitoring metrics (20 pts)"

# ===================== TASK 3 =====================
echo; echo "=== Task 3: log-based metric $METRIC (4K / 8K uploads, label file_format) ==="
cat > ~/log_metric.yaml <<'YAML'
description: Rate of high resolution (4K and 8K) video uploads
filter: 'textPayload=~"file_format\: ([4,8]K).*"'
labelExtractors:
  file_format: 'REGEXP_EXTRACT(textPayload, "file_format\\: ([4,8]K).*")'
metricDescriptor:
  metricKind: DELTA
  valueType: INT64
  unit: '1'
  labels:
  - key: file_format
    valueType: STRING
    description: video file format
YAML
if gcloud logging metrics describe "$METRIC" >/dev/null 2>&1; then
  gcloud logging metrics update "$METRIC" --config-from-file="$HOME/log_metric.yaml" >/dev/null && echo "   updated $METRIC"
else
  gcloud logging metrics create "$METRIC" --config-from-file="$HOME/log_metric.yaml" || die "could not create $METRIC"
fi
gcloud logging metrics describe "$METRIC" --format="yaml(name,filter,labelExtractors)"
pause "Create a custom metric using Cloud Operations logging events (20 pts)"

# ===================== TASK 4 =====================
echo; echo "=== Task 4: add both metrics to Media_Dashboard ==="
DASH=$(gcloud monitoring dashboards list --filter="displayName=Media_Dashboard" --format="value(name)" | head -1)
[ -n "$DASH" ] || die "Media_Dashboard not found"
gcloud monitoring dashboards describe "$DASH" --format=json > ~/dash.json
METRIC="$METRIC" python3 - <<'PY'
import json, os
p = os.path.expanduser("~/dash.json"); d = json.load(open(p)); m = os.environ["METRIC"]
def chart(title, flt, aligner):
    return {"title": title, "xyChart": {"dataSets": [{"plotType": "LINE", "timeSeriesQuery": {
        "timeSeriesFilter": {"filter": flt, "aggregation": {"alignmentPeriod": "60s", "perSeriesAligner": aligner}}}}]}}
w1 = chart("Video input queue length",
           'metric.type="custom.googleapis.com/opencensus/my.videoservice.org/measure/input_queue_size" resource.type="gce_instance"', "ALIGN_MEAN")
w2 = chart("High resolution video upload rate", 'metric.type="logging.googleapis.com/user/%s"' % m, "ALIGN_RATE")
existing = json.dumps(d)
# a chart counts as present only if it already uses the right metric (not just the same title)
new = [w for w, key in ((w1, "input_queue_size"), (w2, "logging.googleapis.com/user/" + m)) if key not in existing]
if not new:
    print("   both charts are already on the dashboard")
elif "mosaicLayout" in d:
    tiles = d["mosaicLayout"].setdefault("tiles", [])
    y = max([t.get("yPos", 0) + t.get("height", 4) for t in tiles] or [0])
    for i, w in enumerate(new):
        tiles.append({"xPos": 0 if i == 0 else 24, "yPos": y, "width": 24, "height": 16, "widget": w})
elif "gridLayout" in d:
    d["gridLayout"].setdefault("widgets", []).extend(new)
elif "columnLayout" in d:
    d["columnLayout"]["columns"][0].setdefault("widgets", []).extend(new)
elif "rowLayout" in d:
    d["rowLayout"]["rows"][0].setdefault("widgets", []).extend(new)
else:
    d["gridLayout"] = {"columns": "2", "widgets": new}
json.dump(d, open(p, "w"))
print("   charts to add:", [w["title"] for w in new])
PY
gcloud monitoring dashboards update "$DASH" --config-from-file="$HOME/dash.json" >/dev/null \
  && echo "   Media_Dashboard updated" || die "could not update Media_Dashboard"
echo "   (log-based data can take 5-10 minutes to show in the charts - the checkpoint only needs the charts)"
pause "Add custom metrics to the Media Dashboard in Cloud Operations Monitoring (20 pts)"

# ===================== TASK 5 =====================
echo; echo "=== Task 5: alert '$ALERT' when $METRIC > $THRESHOLD per second ==="
RTYPE=$(gcloud logging read 'textPayload=~"file_format"' --limit=1 --format="value(resource.type)" 2>/dev/null)
RTYPE=${RTYPE:-cloud_function}
API="https://monitoring.googleapis.com/v3/projects/$PROJECT_ID/alertPolicies"
# an alert counts as present only if it already watches this metric
EXISTS=$(curl -s -H "Authorization: Bearer $(gcloud auth print-access-token)" "$API" | grep -cE "user/${METRIC}([^a-zA-Z0-9_]|$)")
if [ "$EXISTS" -gt 0 ]; then
  echo "   alert policy already exists"
else
  cat > ~/alert.json <<EOF
{
  "displayName": "$ALERT",
  "combiner": "OR",
  "conditions": [{
    "displayName": "High resolution video upload rate above $THRESHOLD/s",
    "conditionThreshold": {
      "filter": "metric.type=\"logging.googleapis.com/user/$METRIC\" AND resource.type=\"$RTYPE\"",
      "aggregations": [{"alignmentPeriod": "60s", "perSeriesAligner": "ALIGN_RATE"}],
      "comparison": "COMPARISON_GT",
      "thresholdValue": $THRESHOLD,
      "duration": "0s",
      "trigger": {"count": 1}
    }
  }],
  "enabled": true
}
EOF
  RES=$(curl -s -X POST -H "Authorization: Bearer $(gcloud auth print-access-token)" -H "Content-Type: application/json" \
        "$API" -d @"$HOME/alert.json")
  echo "$RES" | grep -q '"name"' && echo "   created alert policy '$ALERT'" || { echo "$RES"; die "could not create the alert (error above)"; }
fi
pause "Create custom alert for large video uploads (20 pts)"

echo; echo ">>> Done. 5 checkpoints x 20 points."
LABEOF
```


bash observability_lab.sh
