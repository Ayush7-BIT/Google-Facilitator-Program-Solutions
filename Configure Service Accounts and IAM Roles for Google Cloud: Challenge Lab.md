```text
cat > lab5.sh <<'LABEOF'
#!/bin/bash
# Universal solver: Configure Service Accounts and IAM Roles for Google Cloud - Challenge Lab
# Run in Cloud Shell with:  bash lab5.sh

ask() { local r; read -e -r -p "$2 [$3]: " r; printf -v "$1" '%s' "${r:-$3}"; }
die() { echo; echo "STOP: $1"; exit 1; }
retry() { local i; for i in 1 2 3 4 5 6; do "$@" && return 0; echo "   retrying in 10s..."; sleep 10; done; return 1; }

# Names fixed by the lab text (not yellow). Change only if a new lab version uses other names.
SA_NAME="devops"
VM2_NAME="vm-2"
YAML_FILE="role-definition.yaml"
BQ_SA_NAME="bigquery-qwiklab"
BQ_VM_NAME="bigquery-instance"
SCOPES="https://www.googleapis.com/auth/cloud-platform"

PROJECT_ID=$(gcloud config get-value project 2>/dev/null)
[ -z "$PROJECT_ID" ] && die "no project set. Run: gcloud config set project <Project ID from the lab panel>"
echo "Project: $PROJECT_ID"

echo
echo "=== Enter the values from the lab (press Enter to keep the default) ==="
ask ZONE    "Zone (challenge scenario: 'region ... and ... zone')" "us-east4-a"
ask ROLE_ID "Custom role ID for Task 5 (use the name in the hint if it gives one)" "custom_sql_role"
REGION="${ZONE%-*}"
gcloud config set compute/zone "$ZONE" >/dev/null 2>&1
gcloud config set compute/region "$REGION" >/dev/null 2>&1
gcloud services enable iam.googleapis.com compute.googleapis.com bigquery.googleapis.com cloudresourcemanager.googleapis.com
cd ~

bind() { gcloud projects add-iam-policy-binding "$PROJECT_ID" --member="serviceAccount:$1" --role="$2" --condition=None >/dev/null; }

echo
echo "=== Task 2: service account '$SA_NAME' ==="
SA="${SA_NAME}@${PROJECT_ID}.iam.gserviceaccount.com"
export PROJECT_ID SA
gcloud iam service-accounts describe "$SA" >/dev/null 2>&1 || gcloud iam service-accounts create "$SA_NAME" --display-name="$SA_NAME"
retry gcloud iam service-accounts describe "$SA" >/dev/null 2>&1 || die "service account $SA was not created"
echo "SA = $SA"

echo
echo "=== Task 3: IAM roles for '$SA_NAME' ==="
retry bind "$SA" roles/iam.serviceAccountUser || die "could not grant iam.serviceAccountUser"
retry bind "$SA" roles/compute.instanceAdmin.v1 || die "could not grant compute.instanceAdmin.v1"
bind "$SA" roles/compute.instanceAdmin 2>/dev/null || true   # extra variant, harmless if unavailable
echo "roles granted"

echo
echo "=== Task 4: VM '$VM2_NAME' with the '$SA_NAME' service account ==="
gcloud compute instances describe "$VM2_NAME" --zone="$ZONE" >/dev/null 2>&1 || \
  gcloud compute instances create "$VM2_NAME" --zone="$ZONE" --service-account="$SA" --scopes="$SCOPES"
echo "Attached service account: $(gcloud compute instances describe "$VM2_NAME" --zone="$ZONE" --format='value(serviceAccounts[0].email)')"
echo "--- verifying from inside $VM2_NAME (list, create, delete an instance) ---"
retry gcloud compute ssh "$VM2_NAME" --zone="$ZONE" --quiet --command="echo ssh-ok" || die "could not SSH into $VM2_NAME"
gcloud compute ssh "$VM2_NAME" --zone="$ZONE" --quiet --command="
gcloud compute instances list --project=$PROJECT_ID
gcloud compute instances create verify-from-vm2 --zone=$ZONE --machine-type=e2-micro --no-service-account --no-scopes --project=$PROJECT_ID
gcloud compute instances list --project=$PROJECT_ID
gcloud compute instances delete verify-from-vm2 --zone=$ZONE --quiet --project=$PROJECT_ID
" || echo "(the check from inside $VM2_NAME reported an error - the VM itself is still set up)"

echo
echo "=== Task 5: custom role from $YAML_FILE ==="
cat > "$YAML_FILE" <<'YAML1'
title: "Cloud SQL Connect Role"
description: "Custom role with Cloud SQL connect and get permissions"
stage: "GA"
includedPermissions:
- cloudsql.instances.connect
- cloudsql.instances.get
YAML1
cat "$YAML_FILE"
if gcloud iam roles describe "$ROLE_ID" --project="$PROJECT_ID" >/dev/null 2>&1; then
  gcloud iam roles update "$ROLE_ID" --project="$PROJECT_ID" --file="$YAML_FILE" --quiet
else
  gcloud iam roles create "$ROLE_ID" --project="$PROJECT_ID" --file="$YAML_FILE"
fi

echo
echo "=== Task 6: BigQuery from a service account ==="
BQ_SA="${BQ_SA_NAME}@${PROJECT_ID}.iam.gserviceaccount.com"
gcloud iam service-accounts describe "$BQ_SA" >/dev/null 2>&1 || gcloud iam service-accounts create "$BQ_SA_NAME" --display-name="$BQ_SA_NAME"
retry gcloud iam service-accounts describe "$BQ_SA" >/dev/null 2>&1 || die "service account $BQ_SA was not created"
retry bind "$BQ_SA" roles/bigquery.dataViewer || die "could not grant BigQuery Data Viewer"
retry bind "$BQ_SA" roles/bigquery.user || die "could not grant BigQuery User"
gcloud compute instances describe "$BQ_VM_NAME" --zone="$ZONE" >/dev/null 2>&1 || \
  gcloud compute instances create "$BQ_VM_NAME" --zone="$ZONE" --service-account="$BQ_SA" --scopes="$SCOPES"

cat > query.py <<'PY1'
from google.auth import compute_engine
from google.cloud import bigquery
credentials = compute_engine.Credentials(
    service_account_email='YOUR_SERVICE_ACCOUNT')
query = '''
SELECT name, SUM(number) as total_people
FROM `bigquery-public-data.usa_names.usa_1910_2013`
WHERE state = 'TX'
GROUP BY name, state
ORDER BY total_people DESC
LIMIT 20
'''
client = bigquery.Client(
    project='YOUR_PROJECT_ID',
    credentials=credentials)
print(client.query(query).to_dataframe())
PY1
sed -i "s/YOUR_PROJECT_ID/${PROJECT_ID}/; s/YOUR_SERVICE_ACCOUNT/${BQ_SA}/" query.py

retry gcloud compute ssh "$BQ_VM_NAME" --zone="$ZONE" --quiet --command="echo ssh-ok" || die "could not SSH into $BQ_VM_NAME"
gcloud compute scp query.py "$BQ_VM_NAME":~/query.py --zone="$ZONE" --quiet
gcloud compute ssh "$BQ_VM_NAME" --zone="$ZONE" --quiet --command="
export DEBIAN_FRONTEND=noninteractive
sudo apt-get update -y -qq && sudo apt-get install -y -qq python3-venv python3-pip &&
python3 -m venv ~/bqenv && source ~/bqenv/bin/activate &&
pip install -q google-cloud-bigquery pyarrow pandas db-dtypes &&
python3 ~/query.py
" | tee bq_output.txt

echo
echo "=== Self-check ==="
check() { if eval "$1"; then echo "  PASS  $2"; else echo "  FAIL  $2"; fi; }
roles_of() { gcloud projects get-iam-policy "$PROJECT_ID" --flatten="bindings[].members" --filter="bindings.members:serviceAccount:$1" --format="value(bindings.role)" 2>/dev/null; }
check 'gcloud iam service-accounts describe "$SA" >/dev/null 2>&1' "Task 2: service account $SA_NAME exists"
check 'roles_of "$SA" | grep -q "roles/iam.serviceAccountUser"' "Task 3: iam.serviceAccountUser granted"
check 'roles_of "$SA" | grep -q "roles/compute.instanceAdmin"' "Task 3: compute.instanceAdmin granted"
check '[ "$(gcloud compute instances describe "$VM2_NAME" --zone="$ZONE" --format="value(serviceAccounts[0].email)" 2>/dev/null)" = "$SA" ]' "Task 4: $VM2_NAME runs as $SA_NAME"
check 'gcloud iam roles describe "$ROLE_ID" --project="$PROJECT_ID" --format="value(includedPermissions)" 2>/dev/null | grep -q "cloudsql.instances.connect"' "Task 5: custom role $ROLE_ID has the permissions"
check 'roles_of "$BQ_SA" | grep -q "roles/bigquery.dataViewer" && roles_of "$BQ_SA" | grep -q "roles/bigquery.user"' "Task 6: BigQuery roles granted to $BQ_SA_NAME"
check '[ "$(gcloud compute instances describe "$BQ_VM_NAME" --zone="$ZONE" --format="value(serviceAccounts[0].email)" 2>/dev/null)" = "$BQ_SA" ]' "Task 6: $BQ_VM_NAME runs as $BQ_SA_NAME"
check 'grep -q total_people bq_output.txt' "Task 6: the BigQuery query ran"
echo
echo ">>> Done. On the lab page, click 'Check my progress' for Tasks 2, 3, 4, 5 and 6."
LABEOF
```


bash lab5.sh











