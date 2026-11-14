```/text
set +H
cat > pgsql_lab.sh <<'LABEOF'
#!/bin/bash
# Create and Manage Cloud SQL for PostgreSQL Instances: Challenge Lab (GSP355)
# Run in Cloud Shell:  bash pgsql_lab.sh    (runs all tasks automatically, no ENTER presses; safe to re-run)

die() { echo; echo "STOP: $1"; exit 1; }
ask() { local r; read -e -r -p "$2 [$3]: " r; printf -v "$1" '%s' "${r:-$3}"; }
ask_req() { local r=""; while [ -z "$r" ]; do read -e -r -p "$2${3:+ [$3]}: " r; r="${r:-$3}"; \
  [ -z "$r" ] && echo "   (required - copy it from the yellow text)"; done; printf -v "$1" '%s' "$r"; }

PROJECT_ID=$(gcloud config get-value project 2>/dev/null)
[ -z "$PROJECT_ID" ] || [ "$PROJECT_ID" = "(unset)" ] && die "run: gcloud config set project <Project ID from the lab panel>"
STUDENT=$(gcloud config get-value account 2>/dev/null)
# source VM: 'postgres-vm' or any VM with 'postgres' in its name, else the only VM in the project
ALL_VMS=$(gcloud compute instances list --format="value(name,zone.basename())" 2>/dev/null)
echo "VMs in this project:"; echo "${ALL_VMS:-   (none yet)}" | sed 's/^/   /'
VM=$(echo "$ALL_VMS" | awk '$1=="postgres-vm"{print $1}' | head -1)
[ -n "$VM" ] || VM=$(echo "$ALL_VMS" | awk 'tolower($1) ~ /postgres/ {print $1}' | head -1)
[ -n "$VM" ] || VM=$(echo "$ALL_VMS" | awk '{print $1}' | head -1)
[ -n "$VM" ] || die "no VM found in project $PROJECT_ID - wait 1-2 minutes for the lab to finish setting up, then re-run"
ZONE=$(echo "$ALL_VMS" | awk -v v="$VM" '$1==v{print $2}' | head -1)
INT_IP=$(gcloud compute instances describe $VM --zone=$ZONE --format="value(networkInterfaces[0].networkIP)")
EXT_IP=$(gcloud compute instances describe $VM --zone=$ZONE --format="value(networkInterfaces[0].accessConfigs[0].natIP)")
DET_SQL=$(gcloud sql instances list --format="value(name)" | head -1)
echo "Project $PROJECT_ID | $VM ($ZONE) internal $INT_IP external $EXT_IP | Cloud SQL: $(gcloud sql instances list --format='value(name)' | tr '\n' ' ')"

STATE=~/.gsp355; mkdir -p "$STATE"
step()  { if [ -f "$STATE/$1" ]; then echo; echo "=== $1: already done - skipping ==="; return 1; fi; return 0; }
done_() { touch "$STATE/$1"; }
if [ -z "$PG_BG" ]; then
  # already running in the background? then just show the log
  if [ -f "$STATE/pid" ] && kill -0 "$(cat "$STATE/pid")" 2>/dev/null; then
    echo "The script is still running in the background - showing its log (Ctrl+C stops watching):"
    tail -n 40 -f ~/pgsql_lab.log; exit 0
  fi
  # saved answers from an earlier run in THIS project? reuse them and resume
  if [ -f "$STATE/vars" ] && grep -q "PROJECT_ID=$PROJECT_ID\$" "$STATE/vars"; then
    . "$STATE/vars"
    echo; echo "RESUMING with your saved answers (finished steps are skipped):"
    echo "   user=$MIG_USER  instance=$DEST_ID  table=$TABLE  days=$PITR_DAYS"
    echo "   (to start completely fresh: rm -rf ~/.gsp355 and run again)"
    RESUME=1
  else
    rm -rf "$STATE"; mkdir -p "$STATE"
  fi
fi
if [ -z "$PG_BG" ] && [ -z "$RESUME" ]; then
echo; echo "=== Enter the YELLOW values from the lab text ==="
ask_req MIG_USER "Task 1: migration user name (yellow, 'Migration user username')"            ""
ask_req DEST_ID  "Task 1: Destination Instance ID (yellow)"                                    "$DET_SQL"
ask_req TABLE    "Task 3: table to GRANT SELECT on (yellow, 'GRANT SELECT ON ...')"            ""
ask_req PITR_DAYS "Task 4: retained transaction log days for PITR (yellow number)"             ""
ask     IAM_USER "Task 3: IAM user (lab student account)"                                      "$STUDENT"
printf 'MIG_USER=%q\nDEST_ID=%q\nTABLE=%q\nPITR_DAYS=%q\nIAM_USER=%q\nPROJECT_ID=%s\n' \
  "$MIG_USER" "$DEST_ID" "$TABLE" "$PITR_DAYS" "$IAM_USER" "$PROJECT_ID" > "$STATE/vars"
fi
REGION=$(gcloud sql instances describe "$DEST_ID" --format="value(region)" 2>/dev/null)
[ -n "$REGION" ] || die "Cloud SQL instance $DEST_ID not found"
SQL_PW='supersecret!'
echo "Running all tasks now - click 'Check my progress' as each task prints DONE."
# ---- continue in the background so a Cloud Shell disconnect does not stop it ----
if [ -z "$PG_BG" ]; then
  export PG_BG=1 MIG_USER DEST_ID TABLE PITR_DAYS IAM_USER
  nohup bash "$0" >> ~/pgsql_lab.log 2>&1 < /dev/null &
  echo $! > "$STATE/pid"
  echo "Started in the background. Ctrl+C only stops WATCHING."
  echo "If Cloud Shell disconnects or restarts, reopen it and run:  bash pgsql_lab.sh"
  echo "(it shows the log if still running, or RESUMES from the first unfinished step)"
  sleep 2; tail -n 40 -f ~/pgsql_lab.log; exit 0
fi

gcloud services enable datamigration.googleapis.com servicenetworking.googleapis.com sqladmin.googleapis.com >/dev/null 2>&1
on_vm() { gcloud compute ssh $VM --zone=$ZONE --quiet --command "sudo bash -s" < "$1"; }

# ===================== TASK 1a: prepare the source =====================
if step prep_source; then
echo; echo "=== Task 1: prepare postgres-vm (pglogical, config, user $MIG_USER, grants, primary keys) ==="
cat > ~/vm_prep.sh <<'VMEOF'
set -e
V=$(ls /etc/postgresql | sort -n | tail -1); echo "PostgreSQL version on the VM: $V"
dpkg -s postgresql-$V-pglogical >/dev/null 2>&1 || { apt-get update -qq; DEBIAN_FRONTEND=noninteractive apt-get install -y -qq postgresql-$V-pglogical; }
CONF=/etc/postgresql/$V/main/postgresql.conf; HBA=/etc/postgresql/$V/main/pg_hba.conf
if ! grep -q "GSP355" $CONF; then cat >> $CONF <<EOF
# GSP355 pglogical
shared_preload_libraries = 'pglogical'
wal_level = logical
wal_sender_timeout = 0
max_replication_slots = 10
max_wal_senders = 10
max_worker_processes = 8
listen_addresses = '*'
EOF
fi
grep -q "0.0.0.0/0" $HBA || echo "host all all 0.0.0.0/0 md5" >> $HBA
systemctl restart postgresql@$V-main 2>/dev/null || systemctl restart postgresql; sleep 5
sudo -u postgres psql -v ON_ERROR_STOP=0 <<'SQL'
DO $$BEGIN IF NOT EXISTS (SELECT FROM pg_roles WHERE rolname='MIGUSER') THEN CREATE USER MIGUSER PASSWORD 'DMS_1s_cool!'; END IF; END$$;
ALTER USER MIGUSER WITH PASSWORD 'DMS_1s_cool!';
ALTER ROLE MIGUSER WITH REPLICATION;
ALTER DATABASE orders OWNER TO MIGUSER;
\c postgres
CREATE EXTENSION IF NOT EXISTS pglogical;
GRANT USAGE ON SCHEMA pglogical TO MIGUSER;
GRANT ALL ON SCHEMA pglogical TO MIGUSER;
GRANT SELECT ON ALL TABLES IN SCHEMA pglogical TO MIGUSER;
\c orders
CREATE EXTENSION IF NOT EXISTS pglogical;
GRANT USAGE ON SCHEMA pglogical TO MIGUSER;
GRANT ALL ON SCHEMA pglogical TO MIGUSER;
GRANT SELECT ON ALL TABLES IN SCHEMA pglogical TO MIGUSER;
GRANT USAGE ON SCHEMA public TO MIGUSER;
GRANT ALL ON SCHEMA public TO MIGUSER;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO MIGUSER;
GRANT SELECT ON ALL SEQUENCES IN SCHEMA public TO MIGUSER;
ALTER TABLE public.distribution_centers OWNER TO MIGUSER;
ALTER TABLE public.inventory_items OWNER TO MIGUSER;
ALTER TABLE public.order_items OWNER TO MIGUSER;
ALTER TABLE public.products OWNER TO MIGUSER;
ALTER TABLE public.users OWNER TO MIGUSER;
DO $$ DECLARE t text; BEGIN
  FOREACH t IN ARRAY ARRAY['distribution_centers','inventory_items','order_items','products','users'] LOOP
    IF NOT EXISTS (SELECT 1 FROM pg_constraint WHERE conrelid = ('public.'||t)::regclass AND contype='p') THEN
      EXECUTE format('ALTER TABLE public.%I ADD PRIMARY KEY (id)', t);
      RAISE NOTICE 'added primary key to %', t;
    END IF;
  END LOOP; END $$;
SELECT conrelid::regclass AS table_with_primary_key FROM pg_constraint WHERE contype='p' AND connamespace='public'::regnamespace;
SQL
VMEOF
sed -i "s/MIGUSER/$MIG_USER/g" ~/vm_prep.sh
on_vm ~/vm_prep.sh || die "preparing postgres-vm failed (error above)"
echo ">>> Task 1 (prepare source) DONE - click 'Check my progress' for 'Prepare a source PostgreSQL database'"
done_ prep_source; fi

# ===================== TASK 1b: DMS continuous migration =====================
DMS="gcloud database-migration"; JOB="$DEST_ID"
jstate() { $DMS migration-jobs describe "$JOB" --region="$REGION" --format="value(state,phase)" 2>/dev/null; }
if step migrate; then
echo; echo "=== Task 1: continuous DMS migration -> existing instance $DEST_ID (VPC peering, default) ==="
gcloud sql users set-password postgres --instance="$DEST_ID" --password="$SQL_PW" --quiet >/dev/null 2>&1
if [ -z "$(gcloud sql instances describe "$DEST_ID" --format='value(ipAddresses.filter(type:PRIVATE).extract(ipAddress).flatten())' 2>/dev/null)" ] \
   && ! gcloud sql instances describe "$DEST_ID" --format=json | grep -q '"type": "PRIVATE"'; then
  echo "   giving $DEST_ID a private IP on the default network (private services access, ~5-10 min)..."
  gcloud compute addresses describe google-managed-services-default --global >/dev/null 2>&1 || \
    gcloud compute addresses create google-managed-services-default --global --purpose=VPC_PEERING --prefix-length=16 --network=default --quiet
  gcloud services vpc-peerings connect --service=servicenetworking.googleapis.com \
    --ranges=google-managed-services-default --network=default --quiet 2>/dev/null || \
  gcloud services vpc-peerings update --service=servicenetworking.googleapis.com \
    --ranges=google-managed-services-default --network=default --force --quiet 2>/dev/null
  gcloud sql instances patch "$DEST_ID" --network="projects/$PROJECT_ID/global/networks/default" --quiet
fi
# fallback if this gcloud version has no --database flag: create the profile through the API
mk_profile_rest() {   # mk_profile_rest <id> <host> <user> <password> <cloudsql-id or "">
  local body; body=$(python3 -c "import json,sys; d={'host':sys.argv[2],'port':5432,'username':sys.argv[3],'password':sys.argv[4],'database':'postgres'}
if sys.argv[5]: d['cloudSqlId']=sys.argv[5]
print(json.dumps({'displayName':sys.argv[1],'postgresql':d}))" "$@")
  echo "   retrying profile $1 through the API (with database=postgres)..."
  curl -s -X POST -H "Authorization: Bearer $(gcloud auth print-access-token)" -H "Content-Type: application/json" \
    "https://datamigration.googleapis.com/v1/projects/$PROJECT_ID/locations/$REGION/connectionProfiles?connectionProfileId=$1" \
    -d "$body" > ~/profile_resp.json
  grep -q '"error"' ~/profile_resp.json && { cat ~/profile_resp.json; return 1; }
  for i in $(seq 1 30); do $DMS connection-profiles describe "$1" --region="$REGION" >/dev/null 2>&1 && return 0; sleep 5; done
  return 1
}
SRC=postgres-vm-source; DST="${DEST_ID}-dest"
DEST_HOST=$(gcloud sql instances describe "$DEST_ID" --format=json | python3 -c "
import sys,json; ips=json.load(sys.stdin).get('ipAddresses',[])
p=[i['ipAddress'] for i in ips if i['type']=='PRIVATE'] or [i['ipAddress'] for i in ips]; print(p[0] if p else '')")
if ! $DMS connection-profiles describe $SRC --region="$REGION" >/dev/null 2>&1; then
  $DMS connection-profiles create postgresql $SRC --region="$REGION" --display-name=$SRC \
    --host="$INT_IP" --port=5432 --username="$MIG_USER" --password='DMS_1s_cool!' --database=postgres \
    || mk_profile_rest $SRC "$INT_IP" "$MIG_USER" 'DMS_1s_cool!' "" \
    || die "could not create the source connection profile (error above)"
fi
echo "   source profile $SRC -> $INT_IP"
if ! $DMS connection-profiles describe "$DST" --region="$REGION" >/dev/null 2>&1; then
  $DMS connection-profiles create postgresql "$DST" --region="$REGION" --display-name="$DST" \
    --host="$DEST_HOST" --port=5432 --username=postgres --password="$SQL_PW" --cloudsql-instance="$DEST_ID" --database=postgres \
    || mk_profile_rest "$DST" "$DEST_HOST" postgres "$SQL_PW" "$DEST_ID" \
    || die "could not create the destination connection profile (error above)"
fi
echo "   destination profile $DST -> $DEST_ID ($DEST_HOST)"
if ! $DMS migration-jobs describe "$JOB" --region="$REGION" >/dev/null 2>&1; then
  $DMS migration-jobs create "$JOB" --region="$REGION" --display-name="$JOB" --type=CONTINUOUS \
    --source=$SRC --destination="$DST" --peer-vpc="projects/$PROJECT_ID/global/networks/default" \
    || die "could not create the migration job (error above)"
  echo "   preparing (demoting) the existing instance $DEST_ID, ~5-10 min..."
  $DMS migration-jobs demote-destination "$JOB" --region="$REGION" || echo "   (demote reported an error above)"
fi
case "$(jstate)" in RUNNING*|STARTING*|COMPLETED*) ;; *)
  # demote runs in the background (5-10 min): keep retrying start until it is accepted
  echo "   waiting for the destination to finish demoting, then starting the job..."
  for i in $(seq 1 40); do
    $DMS migration-jobs start "$JOB" --region="$REGION" --quiet >/dev/null 2>&1 \
      && { echo "   job started"; break; }
    echo "   still demoting ($i/40) - retry in 30s"; sleep 30
  done;;
esac
for i in $(seq 1 80); do
  S=$(jstate); echo "   $JOB: $S ($i/80)"
  case "$S" in *CDC*|COMPLETED*) break;; FAILED*) $DMS migration-jobs describe "$JOB" --region="$REGION" --format="value(error)"; break;; esac
  sleep 15
done
echo ">>> Task 1 (migration) DONE - click 'Check my progress' for 'Migrate a stand-alone PostgreSQL database' NOW"
echo "    (it only passes while the job is running - promoting in 3 minutes)"
sleep 180
done_ migrate; fi

if step promote; then
echo; echo "=== Task 2: promote the replica to stand-alone ==="
case "$(jstate)" in COMPLETED*) ;; *) $DMS migration-jobs promote "$JOB" --region="$REGION" || echo "   (promote reported an error above)";; esac
for i in $(seq 1 40); do S=$(jstate); echo "   $JOB: $S"; case "$S" in COMPLETED*) break;; esac; sleep 15; done
echo ">>> Task 2 DONE - click 'Check my progress' for 'Promote a DMS continuous replica'"
done_ promote; fi

# ===== make sure the orders tables really are in Cloud SQL (DMS sometimes copies none) =====
if step fill_tables; then
echo; echo "=== checking that the orders tables arrived in $DEST_ID ==="
gcloud sql instances patch "$DEST_ID" --authorized-networks="$EXT_IP/32" --quiet >/dev/null \
  && echo "   authorized network $EXT_IP (source VM)"
gcloud sql users set-password postgres --instance="$DEST_ID" --password="$SQL_PW" --quiet >/dev/null 2>&1
fi
PUB_IP=$(gcloud sql instances describe "$DEST_ID" --format=json | python3 -c \
  "import sys,json;print([i['ipAddress'] for i in json.load(sys.stdin)['ipAddresses'] if i['type']=='PRIMARY'][0])")
fill_tables() {   # fill_tables <cloud-sql-ip> : copy orders from the VM if the db is empty
  cat > ~/vm_fill.sh <<VMEOF
export PGPASSWORD='$SQL_PW'
N=\$(psql -h $1 -U postgres -d orders -tAc "select count(*) from pg_tables where schemaname='public'")
echo "   tables in Cloud SQL orders db ($1): \$N"
if [ "\$N" = "0" ]; then
  echo "   empty - copying the orders tables from the VM..."
  sudo -u postgres pg_dump -d orders --no-owner --no-acl > /tmp/orders.sql
  psql -h $1 -U postgres -d orders -q -f /tmp/orders.sql 2>&1 | grep -v pglogical
fi
psql -h $1 -U postgres -d orders -c '\\dt'
VMEOF
  on_vm ~/vm_fill.sh
}
if step fill_tables; then fill_tables "$PUB_IP" && done_ fill_tables; fi

# ===================== TASK 3: IAM auth =====================
if step iam; then
echo; echo "=== Task 3: IAM database authentication ==="
FLAGS=$(gcloud sql instances describe "$DEST_ID" --format="value(settings.databaseFlags)")
if ! echo "$FLAGS" | grep -q "cloudsql.iam_authentication"; then
  echo "   turning on cloudsql.iam_authentication (instance restarts, ~2 min)"
  gcloud sql instances patch "$DEST_ID" --database-flags=cloudsql.iam_authentication=on --quiet
fi
gcloud sql users list --instance="$DEST_ID" --format="value(name)" | grep -qx "$IAM_USER" || \
  gcloud sql users create "$IAM_USER" --instance="$DEST_ID" --type=cloud_iam_user && echo "   IAM user $IAM_USER added"
cat > ~/vm_grant.sh <<VMEOF
PGPASSWORD='$SQL_PW' psql -h $PUB_IP -U postgres -d orders -c 'GRANT SELECT ON $TABLE TO "$IAM_USER";' -c '\dp $TABLE'
VMEOF
on_vm ~/vm_grant.sh || echo "   (grant from the VM failed - see error above)"
echo ">>> Task 3 DONE - click 'Check my progress' for 'Secure a Cloud SQL table using IAM authentication'"
done_ iam; fi

# ===================== TASK 4: PITR =====================
if step pitr_insert; then
echo; echo "=== Task 4: point-in-time recovery ($PITR_DAYS days) and clone ==="
gcloud sql instances patch "$DEST_ID" --backup-start-time=00:00 --enable-point-in-time-recovery \
  --retained-transaction-log-days="$PITR_DAYS" --quiet || die "could not enable PITR"
echo "   waiting 2 min so the transaction log has a recovery window..."; sleep 120
TIME_STAMP=$(date -u --rfc-3339=ns | sed -r 's/ /T/; s/\.([0-9]{3}).*/\.\1Z/')
echo "$TIME_STAMP" > "$STATE/timestamp"
echo "   recorded timestamp: $TIME_STAMP"; sleep 30
cat > ~/vm_insert.sh <<VMEOF
PGPASSWORD='$SQL_PW' psql -h $PUB_IP -U postgres -d orders -c "INSERT INTO distribution_centers VALUES(-80.1918,25.7617,'Miami FL',11);" -c "SELECT * FROM distribution_centers ORDER BY id DESC LIMIT 3;"
VMEOF
on_vm ~/vm_insert.sh || echo "   (insert failed - see error above)"
sleep 30
done_ pitr_insert; fi
TIME_STAMP=$(cat "$STATE/timestamp" 2>/dev/null)
C=postgres-orders-pitr
if step pitr_clone; then
if gcloud sql instances describe $C >/dev/null 2>&1; then echo "   $C already exists"
else
  for i in 1 2 3 4 5; do
    gcloud sql instances clone "$DEST_ID" $C --point-in-time "$TIME_STAMP" --async \
      >/dev/null 2>&1 && { echo "   clone $C started"; break; }
    gcloud sql instances describe $C >/dev/null 2>&1 && break
    echo "   clone not possible yet (recovery window still forming) - retry $i/5 in 60s"; sleep 60
  done
fi
echo "   waiting for $C to be ready (5-10 min)..."
for i in $(seq 1 60); do
  S=$(gcloud sql instances describe $C --format="value(state)" 2>/dev/null)
  [ "$S" = "RUNNABLE" ] && break; echo "   $C: ${S:-creating} ($i/60)"; sleep 20
done
# the clone must hold the tables WITHOUT the row inserted after the timestamp
gcloud sql instances patch $C --authorized-networks="$EXT_IP/32" --quiet >/dev/null 2>&1
gcloud sql users set-password postgres --instance=$C --password="$SQL_PW" --quiet >/dev/null 2>&1
CIP=$(gcloud sql instances describe $C --format=json | python3 -c \
  "import sys,json;print([i['ipAddress'] for i in json.load(sys.stdin)['ipAddresses'] if i['type']=='PRIMARY'][0])")
fill_tables "$CIP"
echo ">>> Task 4 DONE - click 'Check my progress' for 'Enable and test Point-in-Time Recovery' (do NOT delete postgres-orders-pitr)"
done_ pitr_clone; fi
echo; echo ">>> All tasks finished. Press Ctrl+C to get the prompt back."
LABEOF
echo "SCRIPT SAVED OK - now run: bash pgsql_lab.sh"
```

pgsql_lab.sh


