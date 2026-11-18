```/text
set +H
cat > arc116_lab.sh <<'LABEOF'
#!/bin/bash
# Implement Sensitive Data Protection on Google Cloud: Challenge Lab (ARC116)
# Run: bash arc116_lab.sh   (3 bucket questions, then ~5 min, no ENTER)
ask() { local r; read -e -r -p "$2 [$3]: " r; printf -v "$1" '%s' "${r:-$3}"; }
die() { echo; echo "STOP: $1 - send Claude a screenshot"; exit 1; }
P=$(gcloud config get-value project 2>/dev/null)
[ -z "$P" ] && die "run: gcloud config set project <Project ID>"
export PROJECT_ID=$P
gcloud services enable dlp.googleapis.com -q
API=https://dlp.googleapis.com/v2/projects/$P
L=$API/locations/us
tok() { gcloud auth print-access-token; }
call() { local a=(-s -X "$1" -H "Authorization: Bearer $(tok)"
  -H "Content-Type: application/json" "$2"); [ -n "$3" ] && a+=(-d "$3")
  curl "${a[@]}"; }

# ---------- bucket names (blank on the lab page = yellow values) ----------
BK=$(gcloud storage buckets list --format="value(name)" 2>/dev/null)
echo "Buckets in this project:"; echo "$BK" | sed 's/^/   /'
pick() { echo "$BK" | grep -m1 -iE "$1"; }
D_IN=$(pick 'input|sample|data'); D_OUT=$(pick 'output|redact|deid')
D_UP=$(pick 'redact|output'); D_UP=${D_UP:-$(echo "$BK" | head -1)}
echo "=== YELLOW values: bucket names from the lab text ==="
ask UPB "Task 1: bucket to upload redact-response.txt to" "${D_UP}"
ask INB "Task 3: input bucket with the sample files" "${D_IN}"
ask OUTB "Task 3: output bucket for de-identified copies" "${D_OUT}"
for b in "$UPB" "$INB" "$OUTB"; do b=${b#gs://}
  gcloud storage buckets describe gs://$b >/dev/null 2>&1 \
    || die "bucket '$b' not found - check the name in the lab text"
done
UPB=${UPB#gs://}; INB=${INB#gs://}; OUTB=${OUTB#gs://}

# ================= Task 1 =================
echo; echo "=== Task 1: redact text with content:deidentify ==="
cd ~
python3 - <<'PY'
import json
v = ("Please update my records with the following information:\n"
     " Email address: foo@example.com,\nNational Provider Identifier: 1245319599")
r = {"item": {"value": v},
     "deidentifyConfig": {"infoTypeTransformations": {"transformations": [
         {"primitiveTransformation": {"replaceWithInfoTypeConfig": {}}}]}},
     "inspectConfig": {"infoTypes": [{"name": "EMAIL_ADDRESS"},
                                     {"name": "US_HEALTHCARE_NPI"}]}}
json.dump(r, open("redact-request.json", "w"), indent=2)
PY
curl -s -X POST -H "Content-Type: application/json" \
  -H "Authorization: Bearer $(tok)" \
  "https://dlp.googleapis.com/v2/projects/$P/content:deidentify" \
  -d @redact-request.json -o redact-response.txt
cat redact-response.txt | head -12
grep -q "EMAIL_ADDRESS" redact-response.txt || die "deidentify call failed"
gcloud storage cp redact-response.txt gs://$UPB/ || die "upload failed"
echo ">>> Task 1 DONE - click 'Check my progress' (Redact sensitive data)"

# ================= Task 2 =================
echo; echo "=== Task 2: de-identify templates in location us ==="
mk_tpl() { # id body
  if call GET "$L/deidentifyTemplates/$1" | grep -q '"name"'; then
    echo "   $1 already exists"; return; fi
  OUT=$(call POST "$L/deidentifyTemplates" "$2")
  echo "$OUT" | grep -q '"name"' && echo "   $1 created" \
    || { echo "$OUT" | head -8; die "template $1 failed"; }
}
mk_tpl structured_data_template '{
 "templateId": "structured_data_template",
 "deidentifyTemplate": {
  "displayName": "structured_data_template",
  "deidentifyConfig": { "recordTransformations": { "fieldTransformations": [
   { "fields": [ {"name": "bank name"}, {"name": "zip code"} ],
     "primitiveTransformation": { "characterMaskConfig": {
        "maskingCharacter": "#", "numberToMask": 0 } } },
   { "fields": [ {"name": "message"} ],
     "infoTypeTransformations": { "transformations": [
        { "primitiveTransformation": { "replaceWithInfoTypeConfig": {} } } ] } }
  ] } } } }'
mk_tpl unstructured_data_template '{
 "templateId": "unstructured_data_template",
 "deidentifyTemplate": {
  "displayName": "unstructured_data_template",
  "deidentifyConfig": { "infoTypeTransformations": { "transformations": [
   { "primitiveTransformation": { "replaceConfig": {
       "newValue": { "stringValue": "[redacted]" } } } } ] } } } }'
echo ">>> Task 2 DONE - click 'Check my progress' (Create DLP templates)"

# ================= Task 3 =================
echo; echo "=== Task 3: job trigger dlp_job (weekly) + run it ==="
PN=$(gcloud projects describe $P --format="value(projectNumber)")
DSA=service-$PN@dlp-api.iam.gserviceaccount.com
call POST "$API:inspectContent" '{"item":{"value":"x"}}' >/dev/null  # creates DLP agent
for b in $INB $OUTB; do
  gcloud storage buckets add-iam-policy-binding gs://$b -q \
    --member="serviceAccount:$DSA" --role=roles/storage.admin >/dev/null 2>&1
done
if call GET "$L/jobTriggers/dlp_job" | grep -q '"name"'; then
  echo "   dlp_job already exists"
else
  TB="projects/$P/locations/us/deidentifyTemplates"
  OUT=$(call POST "$L/jobTriggers" "{
   \"triggerId\": \"dlp_job\",
   \"jobTrigger\": {
    \"displayName\": \"dlp_job\", \"status\": \"HEALTHY\",
    \"triggers\": [ { \"schedule\": { \"recurrencePeriodDuration\": \"604800s\" } } ],
    \"inspectJob\": {
     \"storageConfig\": { \"cloudStorageOptions\": {
        \"fileSet\": { \"regexFileSet\": { \"bucketName\": \"$INB\" } },
        \"filesLimitPercent\": 100 } },
     \"inspectConfig\": {},
     \"actions\": [ { \"deidentify\": {
        \"transformationConfig\": {
          \"deidentifyTemplate\": \"$TB/unstructured_data_template\",
          \"structuredDeidentifyTemplate\": \"$TB/structured_data_template\" },
        \"cloudStorageOutput\": \"gs://$OUTB\" } } ]
    } } }")
  echo "$OUT" | grep -q '"name"' && echo "   dlp_job created (weekly)" \
    || { echo "$OUT" | head -10; die "job trigger failed"; }
fi
echo "   running the trigger now (Run now)..."
sleep 20   # let the new trigger and bucket permissions settle
JOB=""
for i in 1 2 3 4; do
  OUT=$(call POST "$L/jobTriggers/dlp_job:activate" '{}')
  JOB=$(echo "$OUT" | python3 -c 'import json,sys
print(json.load(sys.stdin).get("name",""))' 2>/dev/null)
  [ -n "$JOB" ] && break
  echo "   start failed:"; echo "$OUT" | grep -m2 '"message"'
  echo "   retry $i in 30s"; sleep 30
done
[ -z "$JOB" ] && die "job did not start - in the console use: Sensitive
  Data Protection > Inspection > Job triggers > dlp_job > RUN NOW"
echo "   job started: ${JOB##*/}"
ST=""
for i in $(seq 1 60); do
  sleep 15
  R=$(curl -s -H "Authorization: Bearer $(tok)" "https://dlp.googleapis.com/v2/$JOB")
  ST=$(echo "$R" | python3 -c 'import json,sys
print(json.load(sys.stdin).get("state",""))' 2>/dev/null)
  echo "   $(date +%H:%M:%S) job state: ${ST:-?}"
  case "$ST" in DONE) break;;
    FAILED|CANCELED) echo "$R" | grep -m3 '"message"'; die "DLP job $ST";; esac
done
[ "$ST" = "DONE" ] || echo "   job still running - wait, then check the output bucket"
echo "   files in gs://$OUTB:"
gcloud storage ls -r "gs://$OUTB/**" 2>/dev/null | head -20
echo ">>> Task 3 DONE - click 'Check my progress' (job trigger + DLP inspection)"
echo "    (output files above must exist; if grey, wait 1-2 min and click again)"
LABEOF
echo "SCRIPT SAVED OK - now run: bash arc116_lab.sh"
```

bash arc116_lab.sh

