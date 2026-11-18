```/text
set +H
cat > enrich_lab.sh <<'LABEOF'
#!/bin/bash
# Enrich Metadata and Discovery of Lakehouse Data (ARC123)
# Run: bash enrich_lab.sh   (no questions, no ENTER; safe to re-run)
die() { echo; echo "STOP: $1"; exit 1; }
PROJECT_ID=$(gcloud config get-value project 2>/dev/null)
[ -z "$PROJECT_ID" ] && die "run: gcloud config set project <Project ID>"
DATASET=ecommerce
CONN=customer_data_connection
TABLE=customer_online_sessions
LOC=US
BUCKET="${PROJECT_ID}-bucket"
FILE=$(gcloud storage ls "gs://$BUCKET/" 2>/dev/null \
  | grep -m1 'customer-online-sessions.csv')
FILE=${FILE:-gs://$BUCKET/customer-online-sessions.csv}
echo "Project: $PROJECT_ID | file: $FILE"
gcloud services enable bigquery.googleapis.com \
  bigqueryconnection.googleapis.com dataplex.googleapis.com \
  datacatalog.googleapis.com >/dev/null 2>&1

# ============ TASK 1 ============
echo; echo "=== Task 1: dataset $DATASET (US) ==="
bq --location=$LOC show "$PROJECT_ID:$DATASET" >/dev/null 2>&1 \
  || bq --location=$LOC mk --dataset "$PROJECT_ID:$DATASET" \
  || die "dataset failed"
echo ">>> Task 1 DONE - click 'Check my progress' for Task 1"

# ============ TASK 2 ============
echo; echo "=== Task 2: connection + Lakehouse table ==="
CID="$PROJECT_ID.$LOC.$CONN"
bq show --connection "$CID" >/dev/null 2>&1 \
  || bq mk --connection --location=$LOC --project_id="$PROJECT_ID" \
       --connection_type=CLOUD_RESOURCE "$CONN" \
  || die "connection failed"
CONN_SA=$(bq show --format=json --connection "$CID" | python3 -c \
  "import sys,json;print(json.load(sys.stdin)['cloudResource']['serviceAccountId'])")
echo "   connection service account: $CONN_SA"
# a brand-new service account takes a minute to exist - retry
ok=0
for i in $(seq 1 20); do
  if gcloud projects add-iam-policy-binding "$PROJECT_ID" \
      --member="serviceAccount:$CONN_SA" --role=roles/storage.objectViewer \
      --condition=None --quiet >/dev/null 2>&1; then ok=1; break; fi
  echo "   service account not ready yet ($i/20) - waiting 10s"; sleep 10
done
[ $ok = 1 ] || die "could not grant Storage Object Viewer to $CONN_SA"
echo "   granted Storage Object Viewer to the connection"
if ! bq show "$PROJECT_ID:$DATASET.$TABLE" >/dev/null 2>&1; then
  bq mkdef --autodetect --source_format=CSV --connection_id="$CID" \
    "$FILE" > ~/tabledef.json
  for i in $(seq 1 10); do
    bq mk --table --external_table_definition=$HOME/tabledef.json \
      "$PROJECT_ID:$DATASET.$TABLE" >/dev/null 2>&1 && break
    echo "   permission still spreading ($i/10) - waiting 20s"; sleep 20
  done
fi
bq query --use_legacy_sql=false --format=pretty \
  "SELECT COUNT(*) AS rows_in_table FROM \`$PROJECT_ID.$DATASET.$TABLE\`" \
  || die "table not readable"
echo ">>> Task 2 DONE - click 'Check my progress' for Task 2"

# ============ TASK 3 ============
echo; echo "=== Task 3: Sensitive Data Aspect on $TABLE ==="
export PROJECT_ID DATASET TABLE
python3 - <<'PY'
import json, os, subprocess, time, urllib.request, urllib.error
P = os.environ["PROJECT_ID"]
D, T = os.environ["DATASET"], os.environ["TABLE"]
API = f"https://dataplex.googleapis.com/v1/projects/{P}/locations/us"
AID = "sensitive-data-aspect"

def tok():
    return subprocess.run(["gcloud", "auth", "print-access-token"],
                          capture_output=True, text=True).stdout.strip()

def call(method, url, body=None):
    data = json.dumps(body).encode() if body is not None else None
    req = urllib.request.Request(url, data=data, method=method, headers={
        "Authorization": "Bearer " + tok(),
        "Content-Type": "application/json"})
    try:
        with urllib.request.urlopen(req, timeout=60) as r:
            return json.loads(r.read() or b"{}")
    except urllib.error.HTTPError as e:
        return {"error": json.loads(e.read() or b"{}").get("error", {})}

template = {
    "name": "sensitive_data_aspect", "type": "record",
    "recordFields": [
        {"name": "has_sensitive_data", "type": "bool", "index": 1,
         "annotations": {"displayName": "Has Sensitive Data"}},
        {"name": "sensitive_data_type", "type": "enum", "index": 2,
         "annotations": {"displayName": "Sensitive Data Type"},
         "enumValues": [{"name": "Location Info", "index": 1},
                        {"name": "Contact Info", "index": 2},
                        {"name": "None", "index": 3}]}]}
if "error" in call("GET", f"{API}/aspectTypes/{AID}"):
    r = call("POST", f"{API}/aspectTypes?aspectTypeId={AID}",
             {"displayName": "Sensitive Data Aspect",
              "metadataTemplate": template})
    if "error" in r:
        print("   aspect type error:", r["error"].get("message"))
    for _ in range(40):
        if "error" not in call("GET", f"{API}/aspectTypes/{AID}"):
            break
        time.sleep(5)
print("   aspect type 'Sensitive Data Aspect' ready")

entry = (f"{API}/entryGroups/@bigquery/entries/bigquery.googleapis.com"
         f"/projects/{P}/datasets/{D}/tables/{T}")
for _ in range(30):
    if "error" not in call("GET", entry):
        break
    print("   waiting for the table in the catalog..."); time.sleep(10)

num = subprocess.run(["gcloud", "projects", "describe", P,
                      "--format=value(projectNumber)"],
                     capture_output=True, text=True).stdout.strip()
data = {"has_sensitive_data": True, "sensitive_data_type": "Location Info"}
done = False
for ref in (P, num):
    key = f"{ref}.us.{AID}"
    r = call("PATCH", f"{entry}?updateMask=aspects&aspectKeys={key}",
             {"aspects": {key: {"data": data}}})
    if AID in json.dumps(r) and "error" not in r:
        done = True; break
    last = r.get("error", {}).get("message", r)
if done:
    print("   applied: Has Sensitive Data = true, "
          "Sensitive Data Type = Location Info")
    print(">>> Task 3 DONE - click 'Check my progress' for Task 3")
else:
    print("   could not apply the aspect:", last)
    print(" Do it in the console (1 min): BigQuery > ecommerce >")
    print(" customer_online_sessions > Add aspect > Sensitive Data Aspect")
    print(" > Has Sensitive Data = true, Type = Location Info > Save")
PY
echo; echo ">>> All tasks finished."
LABEOF
echo "SCRIPT SAVED OK - now run: bash enrich_lab.sh"
```

bash enrich_lab.sh

