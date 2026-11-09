```/text
cat > scc_lab.sh <<'LABEOF'
#!/bin/bash
# Universal solver: Mitigate Threats and Vulnerabilities with Security Command Center: Challenge Lab (GSP382)
# Run in Cloud Shell with:  bash scc_lab.sh      (safe to re-run)

ask() { local r; read -e -r -p "$2 [$3]: " r; printf -v "$1" '%s' "${r:-$3}"; }
die() { echo; echo "STOP: $1"; exit 1; }

PROJECT_ID=$(gcloud config get-value project 2>/dev/null)
if [ -z "$PROJECT_ID" ] || [ "$PROJECT_ID" = "(unset)" ]; then
  die "no project set. Run: gcloud config set project <Project ID from the lab panel>"
fi
echo "Project: $PROJECT_ID"

# ---------- auto-detect defaults ----------
DET_VM=$(gcloud compute instances list --format="value(name)" 2>/dev/null | grep -m1 -i cls)
DET_VM=${DET_VM:-cls-vm}
DET_ZONE=$(gcloud compute instances list --filter="name=$DET_VM" --format="value(zone.basename())" 2>/dev/null | head -1)
DET_REGION=$(gcloud config get-value compute/region 2>/dev/null)
if [ -z "$DET_REGION" ] || [ "$DET_REGION" = "(unset)" ]; then
  DET_REGION=$(gcloud compute project-info describe --quiet \
    --format="value(commonInstanceMetadata.items[google-compute-default-region])" 2>/dev/null </dev/null)
fi
[ -z "$DET_REGION" ] && [ -n "$DET_ZONE" ] && DET_REGION="${DET_ZONE%-*}"

echo
echo "=== Enter the YELLOW values from the lab text (press Enter to keep the [default]) ==="
ask VM      "VM name (Task 4: 'Compute Engine > VM instances > ...')"            "$DET_VM"
ask ZONE    "VM zone (auto-detected from the VM)"                                "$DET_ZONE"
ask BUCKET  "Export bucket name (Task 5: 'scc-export-bucket-...')"               "scc-export-bucket-${PROJECT_ID}"
ask REGION  "Bucket location (Task 5: Regional, location ...)"                   "${DET_REGION:-us-central1}"
ask PAUSE   "Pause so you can click 'Check my progress'? (y/n)"                  "y"
[ -n "$ZONE" ] || die "could not find the VM's zone - type it (VM instances page shows it)"
VM_REGION="${ZONE%-*}"

pause() {
  if [ "$PAUSE" = "y" ]; then
    read -r -p ">>> Lab page: click 'Check my progress' for: $1 -- then press ENTER here to continue " _
  fi
}

echo
echo "=== Enabling APIs ==="
gcloud services enable securitycenter.googleapis.com websecurityscanner.googleapis.com 2>/dev/null \
  || echo "   (could not enable some APIs - usually they are already on, continuing)"

echo
echo "NOTE (Task 1): in the console, Security Command Center > Findings, set the time range to"
echo "      'Last 180 days' whenever you look at SCC. The script itself does not need it."

# ================= Task 2: static mute rules =================
echo
echo "=== Task 2: static mute rules ==="
mute() {   # mute <name> <category> <description>
  if gcloud scc muteconfigs describe "$1" --project="$PROJECT_ID" --location=global >/dev/null 2>&1 \
     || gcloud scc muteconfigs describe "$1" --project="$PROJECT_ID" >/dev/null 2>&1; then
    echo "   $1 already exists"; return 0
  fi
  gcloud scc muteconfigs create "$1" --project="$PROJECT_ID" --location=global \
      --description="$3" --filter="category=\"$2\"" --type=STATIC >/dev/null 2>&1 \
  || gcloud scc muteconfigs create "$1" --project="$PROJECT_ID" \
      --description="$3" --filter="category=\"$2\"" --type=STATIC >/dev/null 2>&1 \
  || gcloud scc muteconfigs create "$1" --project="$PROJECT_ID" \
      --description="$3" --filter="category=\"$2\"" \
  || { echo "   (could not create $1)"; return 1; }
  echo "   created $1  (category=$2)"
}
mute muting-flow-log-findings      FLOW_LOGS_DISABLED     "Rule for muting VPC Flow Logs"
mute muting-audit-logging-findings AUDIT_LOGGING_DISABLED "Rule for muting audit logs"
mute muting-admin-sa-findings      ADMIN_SERVICE_ACCOUNT  "Rule for muting admin service account findings"
pause "Create static mute rules for Cymbal Bank (30 points)"

# ================= Task 3: Open SSH / Open RDP =================
echo
echo "=== Task 3: restrict open SSH (22) and RDP (3389) firewall rules to 35.235.240.0/20 ==="
IAP_RANGE="35.235.240.0/20"
fixed=0
for rule in $(gcloud compute firewall-rules list --filter="direction=INGRESS AND sourceRanges:0.0.0.0/0" --format="value(name)" 2>/dev/null); do
  ports=$(gcloud compute firewall-rules describe "$rule" --format="json(allowed)" 2>/dev/null)
  if echo "$ports" | grep -qE '"22"|"3389"|"(tcp|all)"' && ! echo "$ports" | grep -q '"ports"' \
     || echo "$ports" | grep -qE '"22"|"3389"'; then
    echo "   updating $rule"
    gcloud compute firewall-rules update "$rule" --source-ranges="$IAP_RANGE" --quiet && fixed=$((fixed+1))
  fi
done
# the usual default names, in case the filter above missed them
for rule in default-allow-ssh default-allow-rdp; do
  if gcloud compute firewall-rules describe "$rule" >/dev/null 2>&1; then
    gcloud compute firewall-rules update "$rule" --source-ranges="$IAP_RANGE" --quiet >/dev/null 2>&1 && echo "   $rule -> $IAP_RANGE"
  fi
done
echo "Firewall rules allowing 22/3389 now:"
gcloud compute firewall-rules list --format="table(name,sourceRanges.list(),allowed[].map().firewall_rule().list())" 2>/dev/null
pause "Fix Cymbal Bank's high vulnerability findings (20 points)"

# ================= Task 4: static IP + Web Security Scan =================
echo
echo "=== Task 4: static IP 'static-ip' for $VM and a Web Security Scan ==="
CUR_IP=$(gcloud compute instances describe "$VM" --zone="$ZONE" --format="value(networkInterfaces[0].accessConfigs[0].natIP)" 2>/dev/null)
if gcloud compute addresses describe static-ip --region="$VM_REGION" >/dev/null 2>&1; then
  STATIC_IP=$(gcloud compute addresses describe static-ip --region="$VM_REGION" --format="value(address)")
  echo "   static-ip already reserved: $STATIC_IP"
elif [ -n "$CUR_IP" ]; then
  # promote the VM's current external IP to a static address named static-ip (no downtime)
  gcloud compute addresses create static-ip --addresses="$CUR_IP" --region="$VM_REGION" || die "could not reserve static-ip"
  STATIC_IP="$CUR_IP"
else
  gcloud compute addresses create static-ip --region="$VM_REGION" || die "could not reserve static-ip"
  STATIC_IP=$(gcloud compute addresses describe static-ip --region="$VM_REGION" --format="value(address)")
fi
# make sure the VM actually uses static-ip
if [ "$CUR_IP" != "$STATIC_IP" ]; then
  AC=$(gcloud compute instances describe "$VM" --zone="$ZONE" --format="value(networkInterfaces[0].accessConfigs[0].name)" 2>/dev/null)
  [ -n "$AC" ] && gcloud compute instances delete-access-config "$VM" --zone="$ZONE" --access-config-name="$AC" --quiet
  gcloud compute instances add-access-config "$VM" --zone="$ZONE" --access-config-name="external-nat" --address="$STATIC_IP" \
    || die "could not attach static-ip to $VM"
fi
URL="http://${STATIC_IP}:8080"
echo "   VM external IP: $STATIC_IP   ->   $URL"

echo "   waiting for the Cymbal Bank web app on $URL ..."
for i in $(seq 1 24); do
  curl -s -m 5 -o /dev/null -w '%{http_code}' "$URL" 2>/dev/null | grep -q '^[23]' && { echo "   web app is up"; break; }
  sleep 5
done

TOKEN=$(gcloud auth print-access-token)
WSS="https://websecurityscanner.googleapis.com/v1/projects/$PROJECT_ID/scanConfigs"
SCAN=$(curl -s -H "Authorization: Bearer $TOKEN" "$WSS" | python3 -c "
import sys, json
for c in json.load(sys.stdin).get('scanConfigs', []):
    if '$STATIC_IP' in ' '.join(c.get('startingUrls', [])): print(c['name']); break")
echo "   address status: $(gcloud compute addresses describe static-ip --region="$VM_REGION" --format='value(status)' 2>/dev/null)"
# A freshly reserved static IP takes a few minutes to be recognised by the Web Security Scanner
# ("SEED_URL_HAS_UNRESERVED_IP_ADDRESS"), so keep retrying for up to ~5 minutes.
if [ -z "$SCAN" ]; then
  for i in $(seq 1 15); do
    TOKEN=$(gcloud auth print-access-token)
    RESP=$(curl -s -X POST -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" "$WSS" \
      -d "{\"displayName\":\"cymbal-bank-scan\",\"startingUrls\":[\"$URL\"]}")
    SCAN=$(echo "$RESP" | python3 -c "import sys, json; print(json.load(sys.stdin).get('name',''))" 2>/dev/null)
    [ -n "$SCAN" ] && break
    if echo "$RESP" | grep -q "UNRESERVED_IP"; then
      echo "   scanner does not see static-ip yet (normal for a new IP) - retry $i/15 in 20s..."
    else
      echo "   scan config error: $(echo "$RESP" | tr -d '\n' | cut -c1-200) - retry $i/15 in 20s..."
    fi
    sleep 20
  done
fi
if [ -n "$SCAN" ]; then
  echo "   scan config: $SCAN"
  TOKEN=$(gcloud auth print-access-token)
  RUN=$(curl -s -X POST -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
    "https://websecurityscanner.googleapis.com/v1/$SCAN:start" -d '{}')
  echo "$RUN" | grep -q '"name"' && echo "   scan started (it runs for a few minutes)" || echo "   start response: $RUN"
  echo "   waiting 60s for the scan to get going..."
  sleep 60
else
  echo
  echo "   !!! Could not create the scan from the script. Do it in the console (1 minute):"
  echo "       Security > Web Security Scanner > + NEW SCAN > Starting URL: $URL"
  echo "       > SAVE > RUN.  The script now continues with Task 5."
fi
pause "Run a Web Security Scan (20 points)"

# ================= Task 5: export findings =================
echo
echo "=== Task 5: export all findings to gs://$BUCKET/findings.jsonl ==="
if gcloud storage buckets describe "gs://$BUCKET" >/dev/null 2>&1; then
  echo "   bucket already exists"
else
  gcloud storage buckets create "gs://$BUCKET" --location="$REGION" --uniform-bucket-level-access \
    || die "could not create the bucket. Check the name matches the lab exactly."
fi
gcloud scc findings list "projects/$PROJECT_ID" --location=global --format=json > ~/findings_all.json 2>/dev/null \
  || gcloud scc findings list "projects/$PROJECT_ID" --format=json > ~/findings_all.json 2>/dev/null
python3 - "$HOME/findings_all.json" "$HOME/findings.jsonl" <<'PY'
import json, sys
try:
    data = json.load(open(sys.argv[1]))
except Exception:
    data = []
with open(sys.argv[2], "w") as f:
    for item in data:
        f.write(json.dumps(item) + "\n")
print(f"   {len(data)} findings written to findings.jsonl")
PY
gcloud storage cp ~/findings.jsonl "gs://$BUCKET/findings.jsonl" || die "could not upload findings.jsonl"
gcloud storage ls -l "gs://$BUCKET/"
pause "Export Cymbal Bank Findings to Google Cloud Storage (30 points)"
echo "   If this checkpoint is NOT green: in the console go to Security Command Center > Findings,"
echo "   set time range 'All time', click EXPORT > Cloud Storage, pick bucket $BUCKET,"
echo "   filename findings.jsonl, format JSONL, then click Export and check again."

# ================= Self-check =================
echo
echo "=== Self-check ==="
check() { if eval "$1"; then echo "  PASS  $2"; else echo "  FAIL  $2"; fi; }
for m in muting-flow-log-findings muting-audit-logging-findings muting-admin-sa-findings; do
  check "gcloud scc muteconfigs describe $m --project=$PROJECT_ID --location=global >/dev/null 2>&1 || gcloud scc muteconfigs describe $m --project=$PROJECT_ID >/dev/null 2>&1" "Task 2: mute rule $m"
done
check '! gcloud compute firewall-rules list --filter="sourceRanges:0.0.0.0/0" --format=json 2>/dev/null | grep -qE "\"(22|3389)\""' "Task 3: no 22/3389 rule open to 0.0.0.0/0"
check 'gcloud compute addresses describe static-ip --region=$VM_REGION >/dev/null 2>&1' "Task 4: static-ip reserved ($STATIC_IP)"
check '[ -n "$SCAN" ]' "Task 4: web security scan config exists"
check 'gcloud storage ls gs://$BUCKET/findings.jsonl >/dev/null 2>&1' "Task 5: gs://$BUCKET/findings.jsonl exists"
echo
echo ">>> Done. 4 checkpoints: mute rules (30), firewall fix (20), web scan (20), findings export (30)."
LABEOF
```

bash scc_lab.sh
