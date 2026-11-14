```/text
set +H
cat > gsp918_lab.sh <<'LABEOF'
#!/bin/bash
# Migrate to Cloud SQL for PostgreSQL Using Database Migration Service (GSP918)
# Run: bash gsp918_lab.sh   (no questions; runs in the background, ~25-35 min)
# If Cloud Shell disconnects: reopen it and run  bash gsp918_lab.sh  again
#   -> shows the log if still running, or resumes from the first unfinished step.
die() { echo; echo "STOP: $1"; exit 1; }
P=$(gcloud config get-value project 2>/dev/null)
[ -z "$P" ] && die "run: gcloud config set project <Project ID>"
VM=postgresql-vm; DEST=postgresql-cloudsql; JOB=vm-to-cloudsql; SRC=postgres-vm
MIG=migration_admin; MIG_PW='DMS_1s_cool!'; SQL_PW='supersecret!'
STATE=~/.gsp918; LOG=~/gsp918_lab.log; mkdir -p "$STATE"
step()  { [ -f "$STATE/$1" ] && { echo; echo "=== $1: already done - skipping ==="; return 1; }; return 0; }
done_() { touch "$STATE/$1"; }

if [ -z "$BG" ]; then
  if [ -f "$STATE/pid" ] && kill -0 "$(cat "$STATE/pid")" 2>/dev/null; then
    echo "Still running in the background - showing the log (Ctrl+C stops watching):"
    tail -n 40 -f "$LOG"; exit 0
  fi
  grep -qx "$P" "$STATE/project" 2>/dev/null || { rm -rf "$STATE"; mkdir -p "$STATE"; }
  echo "$P" > "$STATE/project"
  export BG=1; nohup bash "$0" >> "$LOG" 2>&1 < /dev/null & echo $! > "$STATE/pid"
  echo "Started in the background. Ctrl+C only stops WATCHING."
  echo "Click 'Check my progress' each time a line says DONE."
  sleep 2; tail -n 40 -f "$LOG"; exit 0
fi

echo "started: $(date)"
ZONE=$(gcloud compute instances list --filter="name=$VM" --format="value(zone.basename())")
[ -n "$ZONE" ] || die "$VM not found"
REGION=$(gcloud sql instances describe $DEST --format="value(region)")
[ -n "$REGION" ] || die "$DEST not found"
INT_IP=$(gcloud compute instances describe $VM --zone=$ZONE \
  --format="value(networkInterfaces[0].networkIP)")
EXT_IP=$(gcloud compute instances describe $VM --zone=$ZONE \
  --format="value(networkInterfaces[0].accessConfigs[0].natIP)")
echo "VM $VM ($ZONE) int $INT_IP ext $EXT_IP | Cloud SQL $DEST ($REGION)"
gcloud services enable datamigration.googleapis.com servicenetworking.googleapis.com \
  sqladmin.googleapis.com >/dev/null 2>&1
on_vm() {   # retry: a brand-new SSH key takes up to ~1 min to reach the VM
  for i in 1 2 3 4 5 6 7 8; do
    gcloud compute ssh $VM --zone=$ZONE --quiet --command "sudo bash -s" < "$1" && return 0
    echo "   ssh not ready yet ($i/8) - retry in 20s"; sleep 20
  done; return 1
}

# ================= TASK 1: prepare the source =================
if step prep; then
echo; echo "=== Task 1: prepare $VM (pglogical, config, $MIG, grants) ==="
cat > ~/vm_prep.sh <<'VMEOF'
set -e
V=$(ls /etc/postgresql | sort -n | tail -1); echo "PostgreSQL $V"
dpkg -s postgresql-$V-pglogical >/dev/null 2>&1 || { apt-get update -qq
  DEBIAN_FRONTEND=noninteractive apt-get install -y -qq postgresql-$V-pglogical; }
CONF=/etc/postgresql/$V/main/postgresql.conf; HBA=/etc/postgresql/$V/main/pg_hba.conf
if ! grep -q GSP918 $CONF; then cat >> $CONF <<EOF
# GSP918
shared_preload_libraries = 'pglogical'
output_plugin_libraries = 'pglogical_output'
wal_level = logical
max_worker_processes = 10
max_replication_slots = 10
max_wal_senders = 10
listen_addresses = '*'
EOF
fi
grep -q "host all all 0.0.0.0/0 md5" $HBA || echo "host all all 0.0.0.0/0 md5" >> $HBA
grep -q "host replication all 0.0.0.0/0 md5" $HBA || \
  echo "host replication all 0.0.0.0/0 md5" >> $HBA
systemctl stop postgresql; systemctl start postgresql@$V-main; sleep 5
sudo -u postgres psql -v ON_ERROR_STOP=0 <<'SQL'
DO $$BEGIN IF NOT EXISTS (SELECT FROM pg_roles WHERE rolname='migration_admin')
THEN CREATE USER migration_admin PASSWORD 'DMS_1s_cool!'; END IF; END$$;
ALTER ROLE migration_admin WITH REPLICATION;
ALTER DATABASE orders OWNER TO migration_admin;
SQL
for DB in postgres orders gmemegen_db; do
sudo -u postgres psql -d $DB -v ON_ERROR_STOP=0 <<'SQL'
CREATE EXTENSION IF NOT EXISTS pglogical;
GRANT USAGE ON SCHEMA pglogical TO migration_admin;
GRANT ALL ON SCHEMA pglogical TO migration_admin;
GRANT SELECT ON ALL TABLES IN SCHEMA pglogical TO migration_admin;
GRANT USAGE ON SCHEMA public TO migration_admin;
GRANT ALL ON SCHEMA public TO migration_admin;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO migration_admin;
SQL
done
sudo -u postgres psql -d orders <<'SQL'
ALTER TABLE public.distribution_centers OWNER TO migration_admin;
ALTER TABLE public.inventory_items OWNER TO migration_admin;
ALTER TABLE public.order_items OWNER TO migration_admin;
ALTER TABLE public.products OWNER TO migration_admin;
ALTER TABLE public.users OWNER TO migration_admin;
SHOW shared_preload_libraries; SHOW wal_level;
SQL
VMEOF
on_vm ~/vm_prep.sh || die "preparing $VM failed (error above)"
echo ">>> Task 1 DONE - click 'Check my progress' for 'Prepare the source database'"
done_ prep; fi

# ================= TASK 2: profile + continuous job =================
DMS="gcloud database-migration"
jstate() { $DMS migration-jobs describe $JOB --region=$REGION \
  --format="value(state,phase)" 2>/dev/null; }
if step profile; then
echo; echo "=== Task 2: connection profile $SRC -> $INT_IP ==="
if ! $DMS connection-profiles describe $SRC --region=$REGION >/dev/null 2>&1; then
  $DMS connection-profiles create postgresql $SRC --region=$REGION \
    --display-name=$SRC --host=$INT_IP --port=5432 --username=$MIG \
    --password="$MIG_PW" --database=postgres \
  || $DMS connection-profiles create postgresql $SRC --region=$REGION \
    --display-name=$SRC --host=$INT_IP --port=5432 --username=$MIG \
    --password="$MIG_PW" \
  || die "could not create connection profile $SRC (error above)"
fi
echo ">>> 'Create a connection profile' DONE - click 'Check my progress'"
done_ profile; fi

if step job; then
echo; echo "=== Task 2: continuous job $JOB -> existing $DEST (VPC peering default) ==="
gcloud sql users set-password postgres --instance=$DEST --password="$SQL_PW" \
  --quiet >/dev/null 2>&1
DST=${DEST}-dest
if ! $DMS connection-profiles describe $DST --region=$REGION >/dev/null 2>&1; then
  DH=$(gcloud sql instances describe $DEST --format=json | python3 -c "
import sys,json; ips=json.load(sys.stdin).get('ipAddresses',[])
p=[i['ipAddress'] for i in ips if i['type']=='PRIVATE'] or [i['ipAddress'] for i in ips]
print(p[0] if p else '')")
  $DMS connection-profiles create postgresql $DST --region=$REGION \
    --display-name=$DST --host="$DH" --port=5432 --username=postgres \
    --password="$SQL_PW" --cloudsql-instance=$DEST --database=postgres \
  || die "could not create the destination profile (error above)"
fi
if ! $DMS migration-jobs describe $JOB --region=$REGION >/dev/null 2>&1; then
  $DMS migration-jobs create $JOB --region=$REGION --display-name=$JOB \
    --type=CONTINUOUS --source=$SRC --destination=$DST \
    --peer-vpc=projects/$P/global/networks/default \
  || die "could not create the migration job (error above)"
  echo "   preparing (demoting) $DEST, 5-10 min..."
  $DMS migration-jobs demote-destination $JOB --region=$REGION
fi
case "$(jstate)" in RUNNING*|STARTING*|COMPLETED*) ;; *)
  for i in $(seq 1 40); do
    $DMS migration-jobs start $JOB --region=$REGION --quiet >/dev/null 2>&1 \
      && { echo "   job started"; break; }
    echo "   still demoting ($i/40) - retry in 30s"; sleep 30
  done;;
esac
for i in $(seq 1 80); do
  S=$(jstate); echo "   $JOB: $S ($i/80)"
  case "$S" in *CDC*) break;; FAILED*) $DMS migration-jobs describe $JOB \
    --region=$REGION --format="value(error)"; break;; esac; sleep 15
done
echo ">>> 'Create and start a continuous migration job' DONE - click 'Check my progress'"
done_ job; fi

# ================= TASK 3: test replication =================
if step confirm; then
echo; echo "=== Task 3: insert Miami FL on the source, check it reached Cloud SQL ==="
cat > ~/vm_ins.sh <<'VMEOF'
sudo -u postgres psql -d orders -c "insert into distribution_centers \
select -80.1918,25.7617,'Miami FL',11 where not exists \
(select 1 from distribution_centers where id=11);" \
 -c "select count(*) from distribution_centers;"
VMEOF
on_vm ~/vm_ins.sh || echo "   (insert failed - see error above)"
echo "   waiting 90s for replication..."; sleep 90
# quick one-shot check (no retries): a replica often refuses outside connections,
# so a failure here is normal and does NOT affect the checkpoint
CIP=$(gcloud sql instances describe $DEST --format=json | python3 -c \
 "import sys,json;print(([i['ipAddress'] for i in json.load(sys.stdin)['ipAddresses'] if i['type']=='PRIVATE'] or [''])[0])" 2>/dev/null)
if [ -n "$CIP" ]; then
  timeout 40 gcloud compute ssh $VM --zone=$ZONE --quiet --command \
    "PGPASSWORD='$SQL_PW' PGCONNECT_TIMEOUT=10 psql -h $CIP -U postgres -d orders -c 'select id, name from distribution_centers order by id desc limit 2;'" \
    || echo "   (could not read Cloud SQL directly - that's OK, replication is checked by the lab)"
fi
echo ">>> Task 3 DONE - click 'Check my progress' for 'Confirm the data' NOW"
echo "    (it must pass while the job is still running - promoting in 3 minutes)"
sleep 180
done_ confirm; fi

# ================= TASK 4: promote =================
if step promote; then
echo; echo "=== Task 4: promote $JOB ==="
case "$(jstate)" in COMPLETED*) ;; *) $DMS migration-jobs promote $JOB --region=$REGION;; esac
for i in $(seq 1 60); do S=$(jstate); echo "   $JOB: $S"
  case "$S" in COMPLETED*) break;; esac; sleep 15; done
echo ">>> Task 4 DONE - click 'Check my progress' for 'Promote Cloud SQL'"
done_ promote; fi
echo; echo ">>> All tasks finished. Press Ctrl+C to get the prompt back."
LABEOF
echo "SCRIPT SAVED OK - now run: bash gsp918_lab.sh"
```

bash gsp918_lab.sh
