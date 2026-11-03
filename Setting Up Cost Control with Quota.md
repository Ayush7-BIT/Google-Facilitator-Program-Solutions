```/text
cat > quota_lab.sh <<'LABEOF'
#!/bin/bash
# Setting Up Cost Control with Quota (GSP651) - universal solution

PROJECT_ID=$(gcloud config get-value project 2>/dev/null)
if [ -z "$PROJECT_ID" ] || [ "$PROJECT_ID" = "(unset)" ]; then
  echo "ERROR: no project set. Run: gcloud config set project <Project ID from the lab panel>"
  exit 1
fi
export GOOGLE_CLOUD_PROJECT="$PROJECT_ID"
export DEVSHELL_PROJECT_ID="$PROJECT_ID"
echo "Project: $PROJECT_ID"

ask() {   # ask "prompt" "default" VARNAME
  local ans
  read -r -p "$1 [$2]: " ans
  printf -v "$3" '%s' "${ans:-$2}"
}

echo
echo "=== Press ENTER to accept each default (the defaults are THIS lab's values) ==="
ask "Public dataset table                 " "bigquery-public-data.wise_all_sky_data_release.mep_wise" TABLE
ask "Per-user daily quota in MiB (.25 TiB)" "262144" QUOTA_VALUE
ask "Max no-cache query runs in Task 4    " "5" MAX_RUNS
ask "Pause so you can click 'Check my progress'? (y/n)" "y" PAUSE

SERVICE="bigquery.googleapis.com"
METRIC="bigquery.googleapis.com/quota/query/usage"
UNIT='1/d/{project}/{user}'
QUERY="SELECT w1mpro_ep, mjd, load_id, frame_id FROM \`${TABLE}\` ORDER BY mjd ASC LIMIT 500"

pause() {
  if [ "$PAUSE" = "y" ]; then
    read -r -p ">>> Lab page: click 'Check my progress' for: $1 -- then press ENTER here to continue " _
  fi
}

gcloud services enable bigquery.googleapis.com serviceusage.googleapis.com --quiet 2>/dev/null || true

# ---------- TASK 1: query the public dataset ----------
echo
echo "=== TASK 1: query a public dataset ==="
echo "Query:"
echo "  $QUERY"
echo
echo "Dry run (what the validator shows - nothing is processed yet):"
bq query --project_id="$PROJECT_ID" --use_legacy_sql=false --dry_run "$QUERY"
echo
echo "Running the query now..."
for i in 1 2 3; do
  if bq query --project_id="$PROJECT_ID" --use_legacy_sql=false --format=pretty --max_rows=5 "$QUERY"; then
    TASK1_OK=1; break
  fi
  echo "Query failed (try $i/3) - waiting 10s"; sleep 10
done
[ "$TASK1_OK" = "1" ] || { echo "FAIL: Task 1 query did not run"; exit 1; }
pause "Query a public dataset in BigQuery"

# ---------- TASK 3: update the quota ----------
echo
echo "=== TASK 3: update the BigQuery per-user quota ==="
echo "--- quota BEFORE ---"
gcloud alpha services quota list --service="$SERVICE" --consumer="projects/${PROJECT_ID}" --filter="usage" --quiet

for i in 1 2 3; do
  if gcloud alpha services quota update --consumer="projects/${PROJECT_ID}" --service "$SERVICE" \
      --metric "$METRIC" --value "$QUOTA_VALUE" --unit "$UNIT" --force --quiet; then
    break
  fi
  echo "Quota update failed (try $i/3) - waiting 15s"; sleep 15
done

echo "--- quota AFTER ---"
QUOTA_OUT=$(gcloud alpha services quota list --service="$SERVICE" --consumer="projects/${PROJECT_ID}" --filter="usage" --quiet 2>&1)
echo "$QUOTA_OUT"
if echo "$QUOTA_OUT" | grep -q "consumerOverride" && echo "$QUOTA_OUT" | grep -q "$QUOTA_VALUE"; then
  OVERRIDE_OK=1
  echo "PASS  consumerOverride with value ${QUOTA_VALUE} is in place"
else
  OVERRIDE_OK=0
  echo "FAIL  override not visible yet"
fi

# ---------- TASK 4: rerun the query ----------
echo
echo "=== TASK 4: rerun the query ==="
echo "Step 1: rerun WITH cache (answer to the lab's question: the cached result is free, so no data is processed)"
bq query --project_id="$PROJECT_ID" --use_legacy_sql=false --format=none "$QUERY" 2>&1 | tail -n 3

echo
echo "Step 2: rerun with the query cache DISABLED so it counts against the quota"
HIT_QUOTA=0
for i in $(seq 1 "$MAX_RUNS"); do
  echo "--- no-cache run $i of $MAX_RUNS ---"
  OUT=$(bq query --project_id="$PROJECT_ID" --use_legacy_sql=false --nouse_cache --format=none "$QUERY" 2>&1)
  echo "$OUT" | tail -n 5
  if echo "$OUT" | grep -qi "quota exceeded"; then
    HIT_QUOTA=1
    echo
    echo ">>> Got the custom quota error - that is exactly what the lab wants."
    break
  fi
  echo "Query ran (no quota error yet). The quota can take a minute to take effect - waiting 60s, then running again..."
  sleep 60
done

echo
echo "Most recent BigQuery jobs:"
bq ls -j -n 8 --project_id="$PROJECT_ID"
pause "Rerun your Query"

# ---------- SELF-CHECK ----------
echo
echo "=================== SELF-CHECK ==================="
echo "PASS  Task 1 query ran"
[ "$OVERRIDE_OK" = "1" ] && echo "PASS  per-user quota override set to ${QUOTA_VALUE} MiB" || echo "FAIL  quota override missing"
[ "$HIT_QUOTA" = "1" ] && echo "PASS  rerun was blocked by the custom quota" || echo "CHECK no quota error seen yet - wait a few minutes and run:  bq query --project_id=$PROJECT_ID --use_legacy_sql=false --nouse_cache --format=none \"\$QUERY\""
echo
echo "Now click 'Check my progress' for any checkpoint that is not green yet."
LABEOF
```



bash quota_lab.sh
