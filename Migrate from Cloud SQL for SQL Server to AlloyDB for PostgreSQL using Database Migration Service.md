```/text
set +H
cat > dms_alloy_helper.sh <<'LABEOF'
#!/bin/bash
# GSP1341 helper: SQL Server -> AlloyDB with DMS
# Run: bash dms_alloy_helper.sh        (finds IPs, waits for dms-pc)
#      bash dms_alloy_helper.sh watch  (watches demo-migration-job)
P=$(gcloud config get-value project 2>/dev/null)
[ -z "$P" ] && { echo "run: gcloud config set project <Project ID>"; exit 1; }
# find the lab's resources instead of assuming their names
CLP=$(gcloud alloydb clusters list --format="value(name)" | head -1)
[ -z "$CLP" ] && { echo "STOP: no AlloyDB cluster found (is the lab ready?)"; exit 1; }
R=$(echo "$CLP" | cut -d/ -f4); CL=${CLP##*/}
INP=$(gcloud alloydb instances list --cluster="$CL" --region="$R" \
  --format="value(name)" | head -1); IN=${INP##*/}
SQ=$(gcloud sql instances list --format="value(name,databaseVersion)" \
  | awk '/SQLSERVER/{print $1; exit}')
echo "AlloyDB cluster: $CL  instance: ${IN:-?}  SQL Server: ${SQ:-?}"
echo "Project: $P   Region: ${R:-?}"

if [ "$1" = "watch" ]; then
  J=$(gcloud database-migration migration-jobs list --region="$R" \
    --format="value(name,displayName)" 2>/dev/null \
    | awk '/demo-migration-job/{print $1; exit}')
  [ -z "$J" ] && { echo "No demo-migration-job yet - create it first."; exit 1; }
  J=${J##*/}; LAST=""; SAID=""
  echo "Watching $J (Ctrl+C stops watching, not the job)"
  while true; do
    S=$(gcloud database-migration migration-jobs describe "$J" \
      --region="$R" --format="value(state,phase)" 2>/dev/null | tr '\t' ' ')
    [ "$S" != "$LAST" ] && echo "   $(date +%H:%M:%S)  state/phase: $S"
    LAST=$S
    case "$S" in
      *RUNNING*CDC*) [ -z "$SAID" ] && { SAID=1; echo
        echo ">>> Full dump finished, job is in CDC."
        echo ">>> Do Task 6 now: check counts 8/91, insert TEST1, see 92,"
        echo ">>> then click PROMOTE on the job. Keep this running."; };;
      *COMPLETED*) echo
        echo ">>> Promotion COMPLETED - click 'Check my progress'"
        echo ">>> for 'Promote the destination instance'."; exit 0;;
      *FAILED*)
        echo ">>> Job FAILED - send Claude a screenshot of the job page."
        gcloud database-migration migration-jobs describe "$J" \
          --region="$R" --format="value(error)"; exit 1;;
    esac
    sleep 20
  done
fi

echo; echo "=== YELLOW values for the connection profiles (Task 3) ==="
AIP=$(gcloud alloydb instances describe "$IN" --cluster="$CL" --region="$R" \
  --format="value(publicIpAddress)")
if [ -z "$AIP" ]; then
  echo "   (no public IP on $IN - the destination profile needs it: AlloyDB >"
  echo "    $IN > Edit > Connectivity > tick 'Enable public IP' > Update)"
fi
SIP=$(gcloud sql instances describe "$SQ" --format=json \
  | python3 -c 'import json,sys
for a in json.load(sys.stdin).get("ipAddresses",[]):
    if a.get("type")=="PRIVATE": print(a["ipAddress"]); break')
echo "   AlloyDB PUBLIC IP (destination profile): ${AIP:-not found}"
echo "   SQL Server INTERNAL IP (source profile): ${SIP:-not found}"
echo "   Region for the profiles:                 $R"

echo; echo "=== Waiting for private connection dms-pc (source profile) ==="
for i in $(seq 1 40); do
  PC=$(gcloud database-migration private-connections list --region="$R" \
    --format="value(name.basename(),state)" 2>/dev/null | head -1)
  ST=$(echo "$PC" | awk '{print $2}')
  echo "   dms-pc state: ${ST:-not visible yet}"
  [ "$ST" = "CREATED" ] && { echo ">>> dms-pc is READY - create the"
    echo ">>> source profile now (Private connectivity: dms-pc)."; break; }
  sleep 30
done
echo
echo "Values again: AlloyDB public IP = ${AIP:-?} | SQL Server internal IP = ${SIP:-?}"
echo "After you start the migration job, run: bash dms_alloy_helper.sh watch"
LABEOF
echo "SCRIPT SAVED OK - now run: bash dms_alloy_helper.sh"
```

bash dms_alloy_helper.sh

