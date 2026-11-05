```/text
cat > apache_lab.sh <<'LABEOF'
#!/bin/bash
# Monitor an Apache Web Server using Ops Agent (GSP1108) - universal solution

PROJECT_ID=$(gcloud config get-value project 2>/dev/null)
if [ -z "$PROJECT_ID" ] || [ "$PROJECT_ID" = "(unset)" ]; then
  echo "ERROR: no project set. Run: gcloud config set project <Project ID from the lab panel>"
  exit 1
fi
export GOOGLE_CLOUD_PROJECT="$PROJECT_ID"
ACCOUNT=$(gcloud config get-value account 2>/dev/null)
echo "Project: $PROJECT_ID"

ask() {   # ask "prompt" "default" VARNAME
  local ans
  read -r -p "$1 [$2]: " ans
  printf -v "$3" '%s' "${ans:-$2}"
}

# ---------- detect the zone ----------
meta() { gcloud compute project-info describe --quiet --format="value(commonInstanceMetadata.items[$1])" 2>/dev/null </dev/null; }
DET_ZONE=$(gcloud config get-value compute/zone 2>/dev/null)
if [ -z "$DET_ZONE" ] || [ "$DET_ZONE" = "(unset)" ]; then DET_ZONE=$(meta google-compute-default-zone); fi
if [ -z "$DET_ZONE" ]; then
  echo "Could not detect the zone - type the one from the lab text."
  DET_ZONE="us-central1-a"
fi

echo
echo "=== Press ENTER to accept each default (check them against the lab text) ==="
ask "Zone (Task 1, 'Zone' row)                        " "$DET_ZONE" ZONE
ask "VM name                                          " "quickstart-vm" VM
ask "Machine type                                     " "e2-small" MACHINE
ask "Boot disk OS as written in the lab (Task 1 step 4)" "Debian GNU/Linux 12 (bookworm)" OS_TEXT
ask "Email address for alerts (one you can read)      " "$ACCOUNT" ALERT_EMAIL
ask "Notification channel display name                " "Apache Alerts" CH_NAME
ask "Alert policy name                                " "Apache traffic above threshold" POLICY_NAME
ask "Alert threshold in bytes/s (4 KiB/s)             " "4000" THRESHOLD
ask "Alert autoclose duration                         " "1800s" AUTOCLOSE
ask "Pause so you can click 'Check my progress'? (y/n)" "y" PAUSE
export CH_NAME ALERT_EMAIL POLICY_NAME THRESHOLD AUTOCLOSE

# ---------- map the OS text to an image family ----------
os_lc=$(echo "$OS_TEXT" | tr 'A-Z' 'a-z')
case "$os_lc" in
  *debian*13*)  IMAGE_FAMILY="debian-13";           IMAGE_PROJECT="debian-cloud" ;;
  *debian*12*)  IMAGE_FAMILY="debian-12";           IMAGE_PROJECT="debian-cloud" ;;
  *debian*11*)  IMAGE_FAMILY="debian-11";           IMAGE_PROJECT="debian-cloud" ;;
  *debian*10*)  IMAGE_FAMILY="debian-10";           IMAGE_PROJECT="debian-cloud" ;;
  *ubuntu*24*)  IMAGE_FAMILY="ubuntu-2404-lts-amd64"; IMAGE_PROJECT="ubuntu-os-cloud" ;;
  *ubuntu*22*)  IMAGE_FAMILY="ubuntu-2204-lts";     IMAGE_PROJECT="ubuntu-os-cloud" ;;
  *ubuntu*20*)  IMAGE_FAMILY="ubuntu-2004-lts";     IMAGE_PROJECT="ubuntu-os-cloud" ;;
  *)            IMAGE_FAMILY="debian-12";           IMAGE_PROJECT="debian-cloud" ;;
esac

pause() {
  if [ "$PAUSE" = "y" ]; then
    read -r -p ">>> Lab page: click 'Check my progress' for: $1 -- then press ENTER here to continue " _
  fi
}

ssh_retry() {   # ssh_retry <command>
  local i
  for i in 1 2 3 4 5 6 7 8 9 10; do
    if gcloud compute ssh "$VM" --zone="$ZONE" --quiet --command="$1"; then return 0; fi
    echo "   ssh not ready yet (try $i/10) - waiting 20s"; sleep 20
  done
  return 1
}

get_ip() {
  gcloud compute instances describe "$VM" --zone="$ZONE" --format="get(networkInterfaces[0].accessConfigs[0].natIP)" 2>/dev/null
}

mon_get()  { curl -s -H "Authorization: Bearer $(gcloud auth print-access-token)" "https://monitoring.googleapis.com/v3/$1"; }
mon_post() { curl -s -X POST -H "Authorization: Bearer $(gcloud auth print-access-token)" -H "Content-Type: application/json" -d "$2" "https://monitoring.googleapis.com/v3/$1"; }

gcloud config set compute/zone "$ZONE" >/dev/null 2>&1
echo
echo "=== Enabling APIs ==="
gcloud services enable compute.googleapis.com monitoring.googleapis.com logging.googleapis.com --quiet

# ---------- TASK 1: VM instance ----------
echo
echo "=== TASK 1: VM instance $VM ($MACHINE) in $ZONE, image $IMAGE_FAMILY ==="
fw() {   # fw <name> <port> <target tag or empty>
  gcloud compute firewall-rules describe "$1" >/dev/null 2>&1 && return 0
  if [ -n "$3" ]; then
    gcloud compute firewall-rules create "$1" --network=default --direction=INGRESS --action=ALLOW \
      --rules="tcp:$2" --source-ranges=0.0.0.0/0 --target-tags="$3" --quiet
  else
    gcloud compute firewall-rules create "$1" --network=default --direction=INGRESS --action=ALLOW \
      --rules="tcp:$2" --source-ranges=0.0.0.0/0 --quiet
  fi
}
fw default-allow-http 80 http-server
fw default-allow-https 443 https-server
fw default-allow-ssh 22 ""

if ! gcloud compute images describe-from-family "$IMAGE_FAMILY" --project="$IMAGE_PROJECT" >/dev/null 2>&1; then
  echo "Image family $IMAGE_FAMILY not found - using debian-12 instead"
  IMAGE_FAMILY="debian-12"; IMAGE_PROJECT="debian-cloud"
fi

if gcloud compute instances describe "$VM" --zone="$ZONE" >/dev/null 2>&1; then
  echo "VM $VM already exists"
else
  gcloud compute instances create "$VM" --zone="$ZONE" --machine-type="$MACHINE" \
    --image-family="$IMAGE_FAMILY" --image-project="$IMAGE_PROJECT" \
    --tags=http-server,https-server || { echo "FAIL: could not create the VM"; exit 1; }
fi
pause "Create a Compute Engine VM instance (20)"

# ---------- remote scripts (they run on the VM) ----------
cat > remote_apache.sh <<'R1'
set -e
sudo apt-get update
sudo DEBIAN_FRONTEND=noninteractive apt-get install -y apache2 php7.0 \
  || sudo DEBIAN_FRONTEND=noninteractive apt-get install -y apache2 php
sudo systemctl enable --now apache2 || true
curl -s localhost | head -n 12
R1

cat > remote_ops.sh <<'R2'
set -e
curl -sSO https://dl.google.com/cloudagents/add-google-cloud-ops-agent-repo.sh
sudo bash add-google-cloud-ops-agent-repo.sh --also-install
# Create a back up of the existing file so existing configurations are not lost.
[ -f /etc/google-cloud-ops-agent/config.yaml ] && sudo cp /etc/google-cloud-ops-agent/config.yaml /etc/google-cloud-ops-agent/config.yaml.bak
sudo tee /etc/google-cloud-ops-agent/config.yaml > /dev/null << 'OPSCFG'
metrics:
  receivers:
    apache:
      type: apache
  service:
    pipelines:
      apache:
        receivers:
          - apache
logging:
  receivers:
    apache_access:
      type: apache_access
    apache_error:
      type: apache_error
  service:
    pipelines:
      apache:
        receivers:
          - apache_access
          - apache_error
OPSCFG
sudo service google-cloud-ops-agent restart
sleep 60
sudo systemctl is-active google-cloud-ops-agent || true
R2

cat > remote_traffic.sh <<'R3'
timeout 120 bash -c -- 'while true; do curl -s localhost > /dev/null; sleep $((RANDOM % 4)); done'
echo "traffic finished"
true
R3

echo
echo "--- copying the helper scripts to $VM (waits for the VM to accept SSH) ---"
copied=0
for i in 1 2 3 4 5 6 7 8 9 10; do
  if gcloud compute scp remote_apache.sh remote_ops.sh remote_traffic.sh "${VM}:~" --zone="$ZONE" --quiet; then copied=1; break; fi
  echo "   not ready yet (try $i/10) - waiting 20s"; sleep 20
done
[ "$copied" = 1 ] || { echo "FAIL: could not reach the VM over SSH (check that the firewall allows tcp:22)"; exit 1; }

# ---------- TASK 2: Apache ----------
echo
echo "=== TASK 2: install the Apache web server ==="
ssh_retry "bash remote_apache.sh" || { echo "FAIL: Apache install did not finish"; exit 1; }
IP=$(get_ip)
echo "External IP: $IP"
for i in 1 2 3 4 5 6; do
  PAGE=$(curl -s -m 10 "http://${IP}")
  echo "$PAGE" | grep -qi "apache" && break
  echo "web page not answering yet (try $i/6) - waiting 10s"; sleep 10
done
echo "$PAGE" | grep -qi "apache" && echo "PASS  http://${IP} shows the Apache page" || echo "CHECK http://${IP} did not show the Apache page"
pause "Install an Apache Web Server (20)"

# ---------- TASK 3: Ops Agent ----------
echo
echo "=== TASK 3: install and configure the Ops Agent (takes about 3 minutes) ==="
ssh_retry "bash remote_ops.sh" || { echo "FAIL: Ops Agent install did not finish"; exit 1; }
pause "Install the Ops Agent (30)"

# ---------- TASK 4: generate traffic ----------
echo
echo "=== TASK 4: generate traffic for 2 minutes ==="
ssh_retry "bash remote_traffic.sh" || echo "(traffic command did not finish - continuing)"

# ---------- TASK 5: alerting policy ----------
echo
echo "=== TASK 5: notification channel and alerting policy ==="
echo "Waiting for the apache.traffic metric to appear in Cloud Monitoring (can take a few minutes)..."
METRIC_OK=0
for i in $(seq 1 40); do
  code=$(curl -s -o /dev/null -w "%{http_code}" -H "Authorization: Bearer $(gcloud auth print-access-token)" \
    "https://monitoring.googleapis.com/v3/projects/${PROJECT_ID}/metricDescriptors/workload.googleapis.com/apache.traffic")
  if [ "$code" = "200" ]; then METRIC_OK=1; break; fi
  echo "   not there yet ($i/40) - waiting 15s"; sleep 15
done
[ "$METRIC_OK" = 1 ] && echo "metric workload.googleapis.com/apache.traffic is available" || echo "WARNING: metric not visible yet - trying to create the policy anyway"

find_by_name() {   # find_by_name <list path> <json key>   (uses $WANT_NAME)
  export WANT_NAME
  mon_get "$1" | python3 -c '
import sys, json, os
want = os.environ["WANT_NAME"]
try:
    d = json.load(sys.stdin)
except Exception:
    sys.exit(0)
for c in d.get("'"$2"'", []):
    if c.get("displayName") == want:
        print(c["name"]); break
'
}

WANT_NAME="$CH_NAME"
CH_ID=$(find_by_name "projects/${PROJECT_ID}/notificationChannels" notificationChannels)
if [ -z "$CH_ID" ]; then
  CH_BODY=$(python3 -c 'import json,os; print(json.dumps({"type":"email","displayName":os.environ["CH_NAME"],"labels":{"email_address":os.environ["ALERT_EMAIL"]}}))')
  CH_RESP=$(mon_post "projects/${PROJECT_ID}/notificationChannels" "$CH_BODY")
  CH_ID=$(echo "$CH_RESP" | python3 -c 'import sys,json; print(json.load(sys.stdin).get("name",""))' 2>/dev/null)
  [ -n "$CH_ID" ] || { echo "FAIL: could not create the notification channel:"; echo "$CH_RESP"; }
fi
echo "Notification channel: $CH_ID"
export CH_ID

WANT_NAME="$POLICY_NAME"
POLICY_ID=$(find_by_name "projects/${PROJECT_ID}/alertPolicies" alertPolicies)
if [ -n "$POLICY_ID" ]; then
  echo "Alert policy already exists: $POLICY_ID"
else
  POLICY_BODY=$(python3 - <<'PYEOF'
import json, os
body = {
  "displayName": os.environ["POLICY_NAME"],
  "combiner": "OR",
  "conditions": [{
    "displayName": "VM Instance - workload/apache.traffic",
    "conditionThreshold": {
      "filter": 'resource.type = "gce_instance" AND metric.type = "workload.googleapis.com/apache.traffic"',
      "aggregations": [{"alignmentPeriod": "60s", "perSeriesAligner": "ALIGN_RATE"}],
      "comparison": "COMPARISON_GT",
      "thresholdValue": float(os.environ["THRESHOLD"]),
      "duration": "0s",
      "trigger": {"count": 1}
    }
  }],
  "alertStrategy": {"autoClose": os.environ["AUTOCLOSE"]},
  "notificationChannels": [os.environ["CH_ID"]] if os.environ.get("CH_ID") else [],
  "enabled": True
}
print(json.dumps(body))
PYEOF
)
  for i in 1 2 3 4 5 6; do
    POLICY_RESP=$(mon_post "projects/${PROJECT_ID}/alertPolicies" "$POLICY_BODY")
    POLICY_ID=$(echo "$POLICY_RESP" | python3 -c 'import sys,json; print(json.load(sys.stdin).get("name",""))' 2>/dev/null)
    [ -n "$POLICY_ID" ] && break
    echo "Policy not created yet (try $i/6): $(echo "$POLICY_RESP" | head -c 300)"
    sleep 30
  done
  [ -n "$POLICY_ID" ] && echo "Alert policy created: $POLICY_ID" || echo "FAIL: could not create the alert policy - use the manual console steps"
fi
pause "Create an alerting policy (30)"

# ---------- SELF-CHECK ----------
echo
echo "=================== SELF-CHECK ==================="
ok()  { echo "  PASS  $1"; }
bad() { echo "  FAIL  $1"; }
DESC=$(gcloud compute instances describe "$VM" --zone="$ZONE" --format="value(status,machineType.basename(),tags.items)" 2>/dev/null)
echo "$DESC" | grep -q "RUNNING" && echo "$DESC" | grep -q "$MACHINE" \
  && ok "Task 1: $VM is RUNNING as $MACHINE" || bad "Task 1: VM missing or wrong type ($DESC)"
echo "$DESC" | grep -q "http-server" && echo "$DESC" | grep -q "https-server" \
  && ok "Task 1: HTTP and HTTPS traffic allowed (tags set)" || bad "Task 1: http/https tags missing"
curl -s -m 10 "http://$(get_ip)" | grep -qi "apache" \
  && ok "Task 2: Apache page answers on http://$(get_ip)" || bad "Task 2: Apache page not answering"
OPS=$(gcloud compute ssh "$VM" --zone="$ZONE" --quiet --command="sudo systemctl is-active google-cloud-ops-agent; sudo grep -c 'type: apache' /etc/google-cloud-ops-agent/config.yaml" 2>/dev/null)
echo "$OPS" | head -n1 | grep -q "^active" && ok "Task 3: Ops Agent service is active" || bad "Task 3: Ops Agent is not active"
[ "$(echo "$OPS" | tail -n1)" -ge 3 ] 2>/dev/null && ok "Task 3: Ops Agent config has the Apache receivers" || bad "Task 3: Apache config missing"
curl -s -o /dev/null -w "%{http_code}" -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  "https://monitoring.googleapis.com/v3/projects/${PROJECT_ID}/metricDescriptors/workload.googleapis.com/apache.traffic" | grep -q 200 \
  && ok "Task 4: apache.traffic metric is flowing into Cloud Monitoring" || echo "  CHECK apache.traffic metric not visible yet (wait a few minutes)"
WANT_NAME="$POLICY_NAME"
[ -n "$(find_by_name "projects/${PROJECT_ID}/alertPolicies" alertPolicies)" ] \
  && ok "Task 5: alert policy '$POLICY_NAME' exists" || bad "Task 5: alert policy missing"
echo
echo "Optional (not scored): Monitoring > Dashboards > Apache Overview to see the charts."
echo ">>> Done. Click 'Check my progress' for any of the 4 checkpoints that is not green yet."
LABEOF
```

bash apache_lab.sh
