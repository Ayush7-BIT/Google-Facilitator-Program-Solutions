```/text
cat > lakehouse_lab.sh <<'LABEOF'
#!/bin/bash
# Secure Lakehouse Data: Challenge Lab (ARC129)
# Run in Cloud Shell:  bash lakehouse_lab.sh     (runs all 3 tasks automatically, no ENTER presses; safe to re-run)

die() { echo; echo "STOP: $1"; exit 1; }
ask() { local r; read -e -r -p "$2 [$3]: " r; printf -v "$1" '%s' "${r:-$3}"; }

PROJECT_ID=$(gcloud config get-value project 2>/dev/null)
[ -z "$PROJECT_ID" ] || [ "$PROJECT_ID" = "(unset)" ] && die "run: gcloud config set project <Project ID from the lab panel>"
PROJECT_NUMBER=$(gcloud projects describe "$PROJECT_ID" --format="value(projectNumber)")
ME=$(gcloud config get-value account 2>/dev/null)
DATASET=online_shop; CONN=user_data_connection; TABLE=user_online_sessions; LOC=US
BUCKET="${PROJECT_ID}-bucket"
echo "Project: $PROJECT_ID ($PROJECT_NUMBER)"

# Username 2 = the other user who has a Cloud Storage role
DET_USER2=$(gcloud projects get-iam-policy "$PROJECT_ID" --flatten="bindings[].members" \
  --format="value(bindings.role,bindings.members)" | awk '$1 ~ /roles\/storage\./ && $2 ~ /^user:/ {sub("user:","",$2); print $2}' \
  | grep -vi "^${ME}$" | head -1)
echo
ask USER2 "Task 3: Username 2 (lab panel 'Username 2')" "$DET_USER2"
[ -n "$USER2" ] || die "Username 2 is required"
FILE=$(gcloud storage ls "gs://$BUCKET/" 2>/dev/null | grep -m1 'user-online-sessions.csv')
FILE=${FILE:-gs://$BUCKET/user-online-sessions.csv}
echo "Source file: $FILE"
echo "Running all tasks now - click 'Check my progress' as each task prints DONE."

gcloud services enable bigquery.googleapis.com bigqueryconnection.googleapis.com dataplex.googleapis.com \
  datacatalog.googleapis.com >/dev/null 2>&1

# ===================== TASK 1 =====================
echo; echo "=== Task 1: dataset, connection, Lakehouse (BigLake) table ==="
bq --location=$LOC show "$PROJECT_ID:$DATASET" >/dev/null 2>&1 || bq --location=$LOC mk --dataset "$PROJECT_ID:$DATASET"
bq show --connection "$PROJECT_ID.$LOC.$CONN" >/dev/null 2>&1 || \
  bq mk --connection --location=$LOC --project_id="$PROJECT_ID" --connection_type=CLOUD_RESOURCE "$CONN" \
  || die "could not create the connection"
CONN_SA=$(bq show --format=json --connection "$PROJECT_ID.$LOC.$CONN" | python3 -c "import sys,json;print(json.load(sys.stdin)['cloudResource']['serviceAccountId'])")
echo "   connection service account: $CONN_SA"
gcloud projects add-iam-policy-binding "$PROJECT_ID" --member="serviceAccount:$CONN_SA" \
  --role=roles/storage.objectViewer --condition=None --quiet >/dev/null && echo "   granted Storage Object Viewer to the connection"
echo "   waiting 30s for the permission to apply..."; sleep 30
if bq show "$PROJECT_ID:$DATASET.$TABLE" >/dev/null 2>&1; then
  echo "   table already exists"
else
  bq mkdef --autodetect --source_format=CSV --connection_id="$PROJECT_ID.$LOC.$CONN" "$FILE" > ~/tabledef.json
  for i in 1 2 3; do
    bq mk --table --external_table_definition=$HOME/tabledef.json "$PROJECT_ID:$DATASET.$TABLE" && break
    echo "   retrying in 30s ($i/3)"; sleep 30
  done
fi
bq show --schema --format=prettyjson "$PROJECT_ID:$DATASET.$TABLE" | grep '"name"' || die "table was not created"
bq query --use_legacy_sql=false --format=pretty "SELECT COUNT(*) AS rows_in_table FROM \`$PROJECT_ID.$DATASET.$TABLE\`"
echo ">>> Task 1 DONE - click 'Check my progress' for Task 1"

# ===================== TASK 2 =====================
echo; echo "=== Task 2: aspect 'Sensitive Data Aspect' on zip, latitude, ip_address, longitude ==="
API="https://dataplex.googleapis.com/v1/projects/$PROJECT_ID/locations/us"
T() { gcloud auth print-access-token; }
ASPECT_ID=sensitive-data-aspect
if ! curl -s -H "Authorization: Bearer $(T)" "$API/aspectTypes/$ASPECT_ID" | grep -q '"name"'; then
  curl -s -X POST -H "Authorization: Bearer $(T)" -H "Content-Type: application/json" \
    "$API/aspectTypes?aspectTypeId=$ASPECT_ID" -d '{
      "displayName": "Sensitive Data Aspect",
      "metadataTemplate": {
        "name": "sensitive_data_aspect", "type": "record",
        "recordFields": [{"name": "has_sensitive_data", "type": "bool", "index": 1,
                          "annotations": {"displayName": "Has Sensitive Data"},
                          "constraints": {"required": true}}]
      }}' | head -c 400; echo
  for i in $(seq 1 30); do
    curl -s -H "Authorization: Bearer $(T)" "$API/aspectTypes/$ASPECT_ID" | grep -q '"name"' && break; sleep 5
  done
fi
echo "   aspect type ready: $ASPECT_ID (Sensitive Data Aspect)"

ENTRY="$API/entryGroups/@bigquery/entries/bigquery.googleapis.com/projects/$PROJECT_ID/datasets/$DATASET/tables/$TABLE"
echo "   waiting for the table to appear in the Dataplex catalog..."
for i in $(seq 1 30); do curl -s -H "Authorization: Bearer $(T)" "$ENTRY" | grep -q '"name"' && break; sleep 10; done
apply() {   # apply <project-ref>
  python3 - "$1" > ~/aspects.json <<'PY'
import json, sys
ref = sys.argv[1]
cols = ["zip", "latitude", "ip_address", "longitude"]
print(json.dumps({"aspects": {f"{ref}.us.sensitive-data-aspect@Schema.{c}": {"data": {"has_sensitive_data": True}} for c in cols}}))
PY
  KEYS=$(for c in zip latitude ip_address longitude; do printf '&aspectKeys=%s.us.sensitive-data-aspect@Schema.%s' "$1" "$c"; done)
  curl -s -X PATCH -H "Authorization: Bearer $(T)" -H "Content-Type: application/json" \
    "$ENTRY?updateMask=aspects$KEYS" -d @"$HOME/aspects.json"
}
RES=$(apply "$PROJECT_ID")
echo "$RES" | grep -q "sensitive-data-aspect@Schema" || RES=$(apply "$PROJECT_NUMBER")
if echo "$RES" | grep -q "sensitive-data-aspect@Schema"; then
  echo "   applied to columns:"; echo "$RES" | grep -o 'sensitive-data-aspect@Schema\.[a-z_]*' | sort -u | sed 's/^/     /'
  echo ">>> Task 2 DONE - click 'Check my progress' for Task 2"
else
  echo "$RES" | head -c 600; echo
  echo " Could not attach the aspect from the script. Do it in the console (2 min):"
  echo "  BigQuery > $DATASET > $TABLE > Schema tab > tick zip, latitude, ip_address, longitude"
  echo "  > 'Add aspect' (or Dataplex Universal Catalog > search $TABLE > Schema > Add aspect)"
  echo "  > choose 'Sensitive Data Aspect' > Has Sensitive Data = true > Save"
fi

# ===================== TASK 3 =====================
echo; echo "=== Task 3: remove Cloud Storage roles from $USER2 (keep Viewer) ==="
ROLES=$(gcloud projects get-iam-policy "$PROJECT_ID" --flatten="bindings[].members" \
  --filter="bindings.members:user:$USER2" --format="value(bindings.role)" | grep '^roles/storage\.')
if [ -z "$ROLES" ]; then echo "   $USER2 has no Cloud Storage roles left"; fi
for r in $ROLES; do
  gcloud projects remove-iam-policy-binding "$PROJECT_ID" --member="user:$USER2" --role="$r" --all --quiet >/dev/null \
    && echo "   removed $r"
done
echo "   roles $USER2 still has:"
gcloud projects get-iam-policy "$PROJECT_ID" --flatten="bindings[].members" \
  --filter="bindings.members:user:$USER2" --format="value(bindings.role)" | sed 's/^/     /'
echo ">>> Task 3 DONE - click 'Check my progress' for Task 3"
echo; echo ">>> All tasks finished."
LABEOF
```

bash lakehouse_lab.sh
