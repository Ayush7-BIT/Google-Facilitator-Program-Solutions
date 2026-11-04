```text
cat > lab5.sh <<'LABEOF'
#!/bin/bash
# Configure Service Accounts and IAM Roles for Google Cloud: Challenge Lab (ARC134) - universal solution
# Run in Cloud Shell with:  bash lab5.sh

ask() { local r; read -e -r -p "$2 [$3]: " r; printf -v "$1" '%s' "${r:-$3}"; }
die() { echo; echo "STOP: $1"; exit 1; }

PROJECT_ID=$(gcloud config get-value project 2>/dev/null)
if [ -z "$PROJECT_ID" ] || [ "$PROJECT_ID" = "(unset)" ]; then
  die "no project set. Run: gcloud config set project <Project ID from the lab panel>"
fi
export PROJECT_ID
echo "Project: $PROJECT_ID"

# ---------- detect region / zone ----------
meta() { gcloud compute project-info describe --quiet --format="value(commonInstanceMetadata.items[$1])" 2>/dev/null </dev/null; }
DET_REGION=$(gcloud config get-value compute/region 2>/dev/null)
[ -z "$DET_REGION" ] || [ "$DET_REGION" = "(unset)" ] && DET_REGION=$(meta google-compute-default-region)
DET_ZONE=$(gcloud config get-value compute/zone 2>/dev/null)
[ -z "$DET_ZONE" ] || [ "$DET_ZONE" = "(unset)" ] && DET_ZONE=$(meta google-compute-default-zone)
if [ -z "$DET_REGION" ]; then
  echo "Could not detect the region - type the one from the lab text."
  DET_REGION="us-west1"
fi

echo
echo "=== Enter the YELLOW values from the lab text (press Enter to keep the default) ==="
ask REGION       "Region"                                              "$DET_REGION"
if [ "$REGION" = "$DET_REGION" ] && [ -n "$DET_ZONE" ]; then ZDEF="$DET_ZONE"; else ZDEF="${REGION}-b"; fi
ask ZONE         "Zone"                                                "$ZDEF"
ask SA_NAME      "Task 2: service account name"                        "devops"
ask VM1          "Task 4: VM name (uses the devops service account)"   "vm-2"
ask ROLE_ID      "Task 5: custom role name / ID (check the hint box)"  "editor"
ask ROLE_TITLE   "Task 5: custom role title"                           "Editor"
ask BQ_SA_NAME   "Task 6: service account name"                        "bigquery-qwiklab"
ask BQ_VM        "Task 6: VM name"                                     "bigquery-instance"
ask MACHINE      "Machine type for both VMs"                           "e2-medium"
ask PAUSE        "Pause so you can click 'Check my progress'? (y/n)"   "y"

SA_EMAIL="${SA_NAME}@${PROJECT_ID}.iam.gserviceaccount.com"
BQ_SA_EMAIL="${BQ_SA_NAME}@${PROJECT_ID}.iam.gserviceaccount.com"
ROLE_FILE="role-definition.yaml"
gcloud config set compute/region "$REGION" >/dev/null 2>&1
gcloud config set compute/zone "$ZONE" >/dev/null 2>&1

pause() {
  if [ "$PAUSE" = "y" ]; then
    read -r -p ">>> Lab page: click 'Check my progress' for: $1 -- then press ENTER here to continue " _
  fi
}

grant() {   # grant <member> <role>
  local i
  for i in 1 2 3 4 5 6; do
    gcloud projects add-iam-policy-binding "$PROJECT_ID" --member="$1" --role="$2" --condition=None >/dev/null 2>&1 && { echo "   granted $2"; return 0; }
    sleep 8
  done
  echo "   (warning: could not grant $2 to $1)"; return 1
}

ssh_retry() {   # ssh_retry <vm> <command>
  local i
  for i in 1 2 3 4 5 6 7 8; do
    if gcloud compute ssh "$1" --zone="$ZONE" --quiet --command="$2"; then return 0; fi
    echo "   ssh not ready yet (try $i/8) - waiting 20s"; sleep 20
  done
  return 1
}

make_sa() {   # make_sa <name> <email>
  if gcloud iam service-accounts describe "$2" >/dev/null 2>&1; then
    echo "service account $1 already exists"
  else
    gcloud iam service-accounts create "$1" --display-name="$1" || die "could not create service account $1"
  fi
  local i
  for i in 1 2 3 4 5 6 7 8 9 10; do
    gcloud iam service-accounts describe "$2" >/dev/null 2>&1 && return 0
    sleep 5
  done
}

echo
echo "=== Enabling APIs ==="
gcloud services enable compute.googleapis.com iam.googleapis.com bigquery.googleapis.com \
  cloudresourcemanager.googleapis.com --quiet

# ---------- TASK 2: service account ----------
echo
echo "=== Task 2: service account $SA_NAME ==="
make_sa "$SA_NAME" "$SA_EMAIL"
pause "Create a service account using gcloud (20)"

# ---------- TASK 3: IAM roles ----------
echo
echo "=== Task 3: grant IAM roles to $SA_EMAIL ==="
export SA="$SA_EMAIL"
grant "serviceAccount:$SA_EMAIL" roles/iam.serviceAccountUser
grant "serviceAccount:$SA_EMAIL" roles/compute.instanceAdmin.v1
grant "serviceAccount:$SA_EMAIL" roles/compute.instanceAdmin
pause "Grant IAM permissions to service account using gcloud (20)"

# ---------- TASK 4: VM with the service account ----------
echo
echo "=== Task 4: VM $VM1 with service account $SA_NAME ==="
if gcloud compute instances describe "$VM1" --zone="$ZONE" >/dev/null 2>&1; then
  echo "VM $VM1 already exists"
else
  gcloud compute instances create "$VM1" --zone="$ZONE" --machine-type="$MACHINE" \
    --service-account="$SA_EMAIL" --scopes=cloud-platform || die "could not create $VM1"
fi
echo "--- SSH into $VM1 and list instances with the service account's permissions ---"
ssh_retry "$VM1" "gcloud compute instances list --project=$PROJECT_ID --format='table(name,zone.basename(),status)'" \
  || echo "(ssh check did not finish - this does not affect the checkpoint)"
pause "Create a compute instance with service account attached using gcloud (20)"

# ---------- TASK 5: custom role ----------
echo
echo "=== Task 5: custom role $ROLE_ID from $ROLE_FILE ==="
cat > "$ROLE_FILE" <<EOF
title: "${ROLE_TITLE}"
description: "Custom role with Cloud SQL connect and get permissions"
stage: "GA"
includedPermissions:
- cloudsql.instances.connect
- cloudsql.instances.get
EOF
cat "$ROLE_FILE"
if gcloud iam roles describe "$ROLE_ID" --project="$PROJECT_ID" >/dev/null 2>&1; then
  echo "role exists - updating it from the file"
  gcloud iam roles undelete "$ROLE_ID" --project="$PROJECT_ID" --quiet >/dev/null 2>&1 || true
  gcloud iam roles update "$ROLE_ID" --project="$PROJECT_ID" --file="$ROLE_FILE" --quiet
else
  gcloud iam roles create "$ROLE_ID" --project="$PROJECT_ID" --file="$ROLE_FILE" || die "could not create the role"
fi
pause "Create a custom role using a YAML file (20)"

# ---------- TASK 6: BigQuery from a service account ----------
echo
echo "=== Task 6: service account $BQ_SA_NAME + VM $BQ_VM + query ==="
make_sa "$BQ_SA_NAME" "$BQ_SA_EMAIL"
grant "serviceAccount:$BQ_SA_EMAIL" roles/bigquery.dataViewer
grant "serviceAccount:$BQ_SA_EMAIL" roles/bigquery.user

if gcloud compute instances describe "$BQ_VM" --zone="$ZONE" >/dev/null 2>&1; then
  echo "VM $BQ_VM already exists"
else
  gcloud compute instances create "$BQ_VM" --zone="$ZONE" --machine-type="$MACHINE" \
    --service-account="$BQ_SA_EMAIL" --scopes=cloud-platform || die "could not create $BQ_VM"
fi

cat > query.py <<'PYEOF'
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
PYEOF
sed -i "s/YOUR_SERVICE_ACCOUNT/${BQ_SA_EMAIL}/; s/YOUR_PROJECT_ID/${PROJECT_ID}/" query.py

cat > remote_bq.sh <<'REMOTEEOF'
sudo apt-get update -y
sudo apt-get install -y python3-pip python3-venv git
python3 -m venv myvenv
source myvenv/bin/activate
pip3 install --upgrade pip
pip3 install google-cloud-bigquery pyarrow pandas db-dtypes
for i in 1 2 3 4 5; do
  python3 query.py && break
  echo "query failed (try $i/5) - the role may still be spreading, waiting 30s"
  sleep 30
done
REMOTEEOF

echo "--- copying files to $BQ_VM (waits for the VM to accept SSH) ---"
copied=0
for i in 1 2 3 4 5 6 7 8; do
  if gcloud compute scp query.py remote_bq.sh "${BQ_VM}:~" --zone="$ZONE" --quiet; then copied=1; break; fi
  echo "   not ready yet (try $i/8) - waiting 20s"; sleep 20
done
[ "$copied" = 1 ] || die "could not copy files to $BQ_VM over SSH"
echo "--- installing dependencies and running query.py on $BQ_VM (a few minutes) ---"
gcloud compute ssh "$BQ_VM" --zone="$ZONE" --quiet --command="bash remote_bq.sh" 2>&1 | tee bq_output.txt
pause "Use the client libraries to access BigQuery from a service account (20)"

# ---------- SELF-CHECK ----------
echo
echo "=================== SELF-CHECK ==================="
ok()  { echo "  PASS  $1"; }
bad() { echo "  FAIL  $1"; }
roles_of() { gcloud projects get-iam-policy "$PROJECT_ID" --flatten="bindings[].members" --filter="bindings.members:serviceAccount:$1" --format="value(bindings.role)" 2>/dev/null; }

gcloud iam service-accounts describe "$SA_EMAIL" >/dev/null 2>&1 && ok "Task 2: service account $SA_NAME exists" || bad "Task 2: service account missing"
R1=$(roles_of "$SA_EMAIL")
echo "$R1" | grep -q "roles/iam.serviceAccountUser" && ok "Task 3: $SA_NAME has iam.serviceAccountUser" || bad "Task 3: serviceAccountUser missing"
echo "$R1" | grep -q "roles/compute.instanceAdmin" && ok "Task 3: $SA_NAME has compute.instanceAdmin" || bad "Task 3: compute.instanceAdmin missing"
[ "$(gcloud compute instances describe "$VM1" --zone="$ZONE" --format='value(serviceAccounts[0].email)' 2>/dev/null)" = "$SA_EMAIL" ] \
  && ok "Task 4: $VM1 runs as $SA_NAME" || bad "Task 4: $VM1 missing or wrong service account"
PERMS=$(gcloud iam roles describe "$ROLE_ID" --project="$PROJECT_ID" --format="value(includedPermissions)" 2>/dev/null)
echo "$PERMS" | grep -q "cloudsql.instances.connect" && echo "$PERMS" | grep -q "cloudsql.instances.get" \
  && ok "Task 5: custom role $ROLE_ID has both Cloud SQL permissions" || bad "Task 5: custom role missing or incomplete"
R2=$(roles_of "$BQ_SA_EMAIL")
echo "$R2" | grep -q "roles/bigquery.dataViewer" && echo "$R2" | grep -q "roles/bigquery.user" \
  && ok "Task 6: $BQ_SA_NAME has BigQuery Data Viewer + User" || bad "Task 6: BigQuery roles missing"
[ "$(gcloud compute instances describe "$BQ_VM" --zone="$ZONE" --format='value(serviceAccounts[0].email)' 2>/dev/null)" = "$BQ_SA_EMAIL" ] \
  && ok "Task 6: $BQ_VM runs as $BQ_SA_NAME" || bad "Task 6: $BQ_VM missing or wrong service account"
grep -q "total_people" bq_output.txt 2>/dev/null && ok "Task 6: query.py printed the BigQuery results" || bad "Task 6: query output not found (see bq_output.txt)"
echo
echo ">>> Done. Click 'Check my progress' for any of the 5 checkpoints that is not green yet."
LABEOF
```

bash lab5.sh




