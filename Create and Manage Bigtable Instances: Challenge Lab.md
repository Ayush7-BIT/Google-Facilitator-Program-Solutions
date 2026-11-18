```/text
set +H
cat > bigtable_lab.sh <<'LABEOF'
#!/bin/bash
# Create and Manage Bigtable Instances: Challenge Lab (GSP380)
# Run: bash bigtable_lab.sh          -> Tasks 1-4 (3 questions, ~20 min, no ENTER)
#      bash bigtable_lab.sh delete   -> Task 5 (ONLY after checks 1-4 are green)
# Runs in background; if Cloud Shell disconnects run it again - finished
# tasks are skipped.  Watch: tail -f ~/bigtable_lab.log
LOG=~/bigtable_lab.log; ENVF=~/.bigtable_lab.env
I=ecommerce-recommendations; C1=$I-c1; C2=$I-c2
ask() { local r; read -e -r -p "$2 [$3]: " r; printf -v "$1" '%s' "${r:-$3}"; }

P=$(gcloud config get-value project 2>/dev/null)
[ -z "$P" ] && { echo "run: gcloud config set project <Project ID>"; exit 1; }

# ------------------------- Task 5 (separate) -------------------------
if [ "$1" = "delete" ]; then
  echo "=== Task 5: delete tables, backups and the instance ==="
  for B in $(gcloud bigtable backups list --instance=$I --cluster=$C1 \
      --format="value(name.basename())" 2>/dev/null); do
    gcloud bigtable backups delete "$B" --instance=$I --cluster=$C1 -q \
      && echo "   backup $B deleted"
  done
  for T in $(gcloud bigtable instances tables list --instances=$I \
      --format="value(name.basename())" 2>/dev/null); do
    gcloud bigtable instances tables delete "$T" --instance=$I -q \
      && echo "   table $T deleted"
  done
  gcloud bigtable instances delete $I -q && echo "   instance $I deleted"
  echo ">>> Task 5 DONE - click 'Check my progress' (Delete Bigtable data)"
  exit 0
fi

# ------------------------- questions -------------------------
if [ "$1" != "--bg" ]; then
  if [ "$(grep -s '^P=' $ENVF | cut -d= -f2)" = "$P" ]; then
    source $ENVF; echo "=== Resuming (ZONE=$ZONE ZONE2=$ZONE2) ==="
  else
    MD="value(commonInstanceMetadata.items.google-compute-default"
    DR=$(gcloud compute project-info describe --format="$MD-region)" 2>/dev/null)
    DZ=$(gcloud compute project-info describe --format="$MD-zone)" 2>/dev/null)
    echo "=== 3 values - press Enter to accept the suggestion ==="
    echo " Where to find them: Lab setup panel (left of the lab page) or the"
    echo " Region/Zone rows in Task 1 and Task 3. If the lab leaves them blank,"
    echo " any value works - the suggestions are fine."
    ask REGION "REGION  (Task 1 'Region' / Lab setup panel)" "${DR:-us-east1}"
    ask ZONE "ZONE for cluster c1  (Task 1 'Zone')" "${DZ:-$REGION-b}"
    Z2=$(gcloud compute zones list --filter="region:$REGION" \
      --format="value(name)" 2>/dev/null | grep -v "^$ZONE$" | sort | head -1)
    ask ZONE2 "ZONE for replica c2  (Task 3 'Zone', must differ from c1)" \
      "${Z2:-$REGION-c}"
    if [ "$ZONE2" = "$ZONE" ]; then   # Bigtable: one cluster per zone
      ZONE2=${Z2:-$REGION-a}
      echo "   c2 cannot share c1's zone - using $ZONE2 instead"
    fi
    printf 'P=%s\nREGION=%s\nZONE=%s\nZONE2=%s\n' "$P" "$REGION" "$ZONE" \
      "$ZONE2" > $ENVF
  fi
  if [ -f ~/.bigtable_lab.pid ] && kill -0 "$(cat ~/.bigtable_lab.pid)" 2>/dev/null
  then echo "Already running - showing the log"
  else nohup bash "$0" --bg > $LOG 2>&1 & echo $! > ~/.bigtable_lab.pid
       echo "Started in background. Ctrl+C only stops watching."; fi
  sleep 1; tail -n +1 -f $LOG --pid="$(cat ~/.bigtable_lab.pid)"; exit 0
fi

# ===================== background part =====================
source $ENVF
gcloud config set project "$P" -q 2>/dev/null
D=~/.bigtable_done/$P; mkdir -p $D
step() { if [ -f $D/$1 ]; then echo; echo "=== $2: already done ==="; return 1
  fi; echo; echo "=== $2 ==="; }
done_() { touch $D/$1; }
die() { echo; echo "STOP: $1 - send Claude a screenshot"; exit 1; }
# checker: multi-region bucket named exactly <project-id>
BKT=gs://$P
TPL=gs://dataflow-templates-$REGION/latest/GCS_SequenceFile_to_Cloud_Bigtable

if step t1 "Task 1: instance $I (Enterprise, SSD, autoscaling 1-5 @60%)"; then
gcloud services enable bigtable.googleapis.com bigtableadmin.googleapis.com \
  dataflow.googleapis.com compute.googleapis.com -q
if ! gcloud bigtable instances describe $I >/dev/null 2>&1; then
  CC="id=$C1,zone=$ZONE,autoscaling-min-nodes=1,autoscaling-max-nodes=5"
  CC="$CC,autoscaling-cpu-target=60"
  gcloud bigtable instances create $I --display-name=$I \
    --cluster-config="$CC" --cluster-storage-type=SSD --edition=ENTERPRISE -q \
  || gcloud bigtable instances create $I --display-name=$I \
    --cluster-config="$CC" --cluster-storage-type=SSD -q \
  || die "instance create failed"
fi
gcloud bigtable clusters describe $C1 --instance=$I \
  --format="table(name.basename(),location.basename(),defaultStorageType,
  clusterConfig.clusterAutoscalingConfig.autoscalingLimits)"
echo ">>> Task 1 DONE - click 'Check my progress' (Create a Bigtable instance)"
done_ t1; fi

if step t2 "Task 2+3a: tables + Dataflow imports"; then
mk() { gcloud bigtable instances tables describe $1 --instance=$I >/dev/null 2>&1 \
  || gcloud bigtable instances tables create $1 --instance=$I \
     --column-families=$2 -q || die "table $1 failed"; echo "   table $1 ($2)"; }
mk SessionHistory Engagements,Sales
mk PersonalizedProducts Recommendations
gcloud storage buckets describe $BKT >/dev/null 2>&1 || \
  gcloud storage buckets create $BKT --location=US -q || die "bucket $BKT failed"
# the lab's own tip: restart the Dataflow API before the jobs
gcloud services disable dataflow.googleapis.com --force -q >/dev/null 2>&1
sleep 10; gcloud services enable dataflow.googleapis.com -q; sleep 20
run_job() { # name table source
  ST=$(gcloud dataflow jobs list --region=$REGION --filter="name=$1" \
    --format="value(state)" 2>/dev/null | head -1)
  case "$ST" in Done|Running|Pending|Queued) echo "   job $1: $ST"; return;; esac
  PRM="bigtableProject=$P,bigtableInstanceId=$I"
  PRM="$PRM,bigtableTableId=$2,sourcePattern=$3"
  for i in 1 2 3; do
    gcloud dataflow jobs run $1 --region=$REGION --gcs-location=$TPL \
      --staging-location=$BKT/temp \
      --parameters="$PRM" \
      >/dev/null && { echo "   job $1 started"; return; }
    echo "   job $1 start failed, retry $i in 30s"; sleep 30
  done; die "Dataflow job $1 could not start"; }
run_job import-sessions SessionHistory \
  gs://spls/gsp380/retail-engagements-sales-00000-of-00001
run_job import-recommendations PersonalizedProducts \
  gs://spls/gsp380/retail-recommendations-00000-of-00001
echo "   waiting for both Dataflow jobs (usually 8-15 min)..."
for i in $(seq 1 60); do
  S1=$(gcloud dataflow jobs list --region=$REGION --filter="name=import-sessions" \
    --format="value(state)" | head -1)
  S2=$(gcloud dataflow jobs list --region=$REGION \
    --filter="name=import-recommendations" --format="value(state)" | head -1)
  echo "   $(date +%H:%M:%S) import-sessions: $S1 | import-recommendations: $S2"
  [ "$S1" = "Done" ] && [ "$S2" = "Done" ] && break
  case "$S1$S2" in *Failed*|*Cancelled*) die "a Dataflow job failed (see above)";; esac
  sleep 30
done
[ "$S1" = "Done" ] && [ "$S2" = "Done" ] || die "Dataflow jobs did not finish in 30 min"
echo "   sample rows (the 'queries' step):"
gcloud bigtable instances tables list --instances=$I --format="value(name.basename())"
command -v cbt >/dev/null || gcloud components install cbt -q >/dev/null 2>&1
if command -v cbt >/dev/null; then
  for T in SessionHistory PersonalizedProducts; do
    echo "   $T rows: $(cbt -project $P -instance $I count $T 2>/dev/null)"
    cbt -project $P -instance $I read $T count=2 2>/dev/null | head -12
  done
fi
echo "   Quiz helper - session red3016#1638940844357 (purchased = Sales):"
cbt -project $P -instance $I lookup SessionHistory "red3016#1638940844357" \
  2>/dev/null | sed 's/^/     /'
echo "   Other quiz rows:  cbt -project $P -instance $I lookup <TABLE> <ROW>"
echo ">>> Task 2 DONE - click 'Check my progress' (Create and populate tables)"
echo ">>> and the PersonalizedProducts / import-recommendations check"
done_ t2; fi

if step t3 "Task 3: replica cluster $C2 in $ZONE2"; then
gcloud bigtable clusters describe $C2 --instance=$I >/dev/null 2>&1 || \
  gcloud bigtable clusters create $C2 --instance=$I --zone=$ZONE2 \
    --autoscaling-min-nodes=1 --autoscaling-max-nodes=5 \
    --autoscaling-cpu-target=60 -q || die "replica cluster failed"
gcloud bigtable clusters list --instances=$I \
  --format="table(name.basename(),location.basename(),state)"
echo ">>> Task 3 DONE - click 'Check my progress' (Configure replication)"
done_ t3; fi

if step t4 "Task 4: backup PersonalizedProducts_7 + restore"; then
gcloud bigtable backups describe PersonalizedProducts_7 --instance=$I \
  --cluster=$C1 >/dev/null 2>&1 || \
  gcloud bigtable backups create PersonalizedProducts_7 --instance=$I \
    --cluster=$C1 --table=PersonalizedProducts --retention-period=7d -q \
  || die "backup failed"
echo "   backup PersonalizedProducts_7 ready (expires in 1 week)"
if ! gcloud bigtable instances tables describe PersonalizedProducts_7_restored \
     --instance=$I >/dev/null 2>&1; then
  SRC=projects/$P/instances/$I/clusters/$C1/backups/PersonalizedProducts_7
  gcloud bigtable instances tables restore --source=$SRC \
    --destination=projects/$P/instances/$I/tables/PersonalizedProducts_7_restored \
    -q \
  || die "restore failed"
fi
for i in $(seq 1 30); do
  gcloud bigtable instances tables describe PersonalizedProducts_7_restored \
    --instance=$I >/dev/null 2>&1 && break; sleep 20; done
gcloud bigtable instances tables list --instances=$I --format="value(name.basename())"
echo ">>> Task 4 DONE - click 'Check my progress' (Back up and restore)"
done_ t4; fi

echo
echo "================================================================="
echo " Click 'Check my progress' for Tasks 1-4 NOW (all must be green)."
echo " THEN run this for Task 5 (deletes everything):"
echo "     bash bigtable_lab.sh delete"
echo "================================================================="
LABEOF
echo "SCRIPT SAVED OK - now run: bash bigtable_lab.sh"
```


bash bigtable_lab.sh


