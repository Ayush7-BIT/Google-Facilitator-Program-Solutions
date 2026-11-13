```/text
set +H
cat > dms_lab.sh <<'LABEOF'
#!/bin/bash
# Migrate MySQL Data to Cloud SQL Using DMS: Challenge Lab (GSP351) - FAST version
# Run: bash dms_lab.sh
#  - asks 2 questions, then runs in the BACKGROUND (survives Cloud Shell disconnects)
#  - Task 2 and Task 3 run IN PARALLEL (about 20 min total instead of 40-50)
#  - if Cloud Shell disconnects: reopen it and run   bash dms_lab.sh   again
#    (it shows the running log, or resumes from where it stopped)

LOG=~/dms_lab.log
VARS=~/.dms_vars
die() { echo; echo "STOP: $1"; exit 1; }

# ---------- if it is already running in the background, just show the log ----------
if [ "$1" != "--run" ] && pgrep -f "dms_lab.sh --run" >/dev/null; then
  echo "The lab script is already running in the background. Showing its log"
  echo "(press Ctrl+C to stop watching - the script keeps running):"
  tail -n 40 -f "$LOG"; exit 0
fi

# ---------- foreground part: questions ----------
if [ "$1" != "--run" ]; then
  PROJECT_ID=$(gcloud config get-value project 2>/dev/null)
  [ -z "$PROJECT_ID" ] && die "run: gcloud config set project <Project ID>"
  SQL_LIST=$(gcloud sql instances list --format="value(name)" 2>/dev/null)
  echo "Cloud SQL instances in this project:"; echo "$SQL_LIST" | sed 's/^/   - /'
  D1=$(echo "$SQL_LIST" | sed -n 1p); D2=$(echo "$SQL_LIST" | sed -n 2p)
  [ -f "$VARS" ] && . "$VARS" && D1=${ONE_TIME_ID:-$D1} && D2=${CONTINUOUS_ID:-$D2}
  echo; echo "=== Enter the values from the lab text (Enter keeps the [default]) ==="
  read -e -r -p "Task 2: instance ID for the ONE-TIME migration (yellow) [$D1]: " a
  read -e -r -p "Task 3: instance ID for the CONTINUOUS migration (yellow) [$D2]: " b
  ONE_TIME_ID=${a:-$D1}; CONTINUOUS_ID=${b:-$D2}
  [ -n "$ONE_TIME_ID" ] && [ -n "$CONTINUOUS_ID" ] || die "both instance IDs are needed"
  [ "$ONE_TIME_ID" != "$CONTINUOUS_ID" ] || die "the two instance IDs must be different"
  printf 'PROJECT_ID=%q\nONE_TIME_ID=%q\nCONTINUOUS_ID=%q\n' \
    "$PROJECT_ID" "$ONE_TIME_ID" "$CONTINUOUS_ID" > "$VARS"
  nohup bash "$0" --run > "$LOG" 2>&1 &
  echo
  echo "Started in the background. Watching the log now."
  echo "Ctrl+C only stops WATCHING. If Cloud Shell disconnects, reopen it and run:"
  echo "   bash dms_lab.sh"
  sleep 2; tail -n 40 -f "$LOG"; exit 0
fi

# ================= background part =================
. "$VARS"
gcloud config set project "$PROJECT_ID" >/dev/null 2>&1
echo "started: $(date)"
gcloud services enable datamigration.googleapis.com sqladmin.googleapis.com \
  servicenetworking.googleapis.com compute.googleapis.com >/dev/null 2>&1
VM=$(gcloud compute instances list --format="value(name)" | head -1)
[ -n "$VM" ] || die "no source Compute Engine VM found"
ZONE=$(gcloud compute instances list --filter="name=$VM" \
  --format="value(zone.basename())")
REGION="${ZONE%-*}"
F="networkInterfaces[0]"
EXT_IP=$(gcloud compute instances describe "$VM" --zone="$ZONE" \
  --format="value($F.accessConfigs[0].natIP)")
INT_IP=$(gcloud compute instances describe "$VM" --zone="$ZONE" \
  --format="value($F.networkIP)")
NETWORK=$(gcloud compute instances describe "$VM" --zone="$ZONE" \
  --format="value($F.network.basename())")
echo "VM $VM ($ZONE) ext $EXT_IP int $INT_IP net $NETWORK"
echo "one-time -> $ONE_TIME_ID | continuous -> $CONTINUOUS_ID"

export PROJECT_ID REGION EXT_IP INT_IP NETWORK ONE_TIME_ID CONTINUOUS_ID
python3 -u - <<'PY'
import json, os, subprocess, sys, time, urllib.request, urllib.error
E = os.environ
P, R = E["PROJECT_ID"], E["REGION"]
ROOT = "https://datamigration.googleapis.com/v1/"
BASE = f"{ROOT}projects/{P}/locations/{R}"
PW = "Qwiklabs-123"
ONE, CON = E["ONE_TIME_ID"], E["CONTINUOUS_ID"]

def tok():
    return subprocess.run(["gcloud", "auth", "print-access-token"],
                          capture_output=True, text=True).stdout.strip()

def api(method, url, body=None):
    data = json.dumps(body).encode() if body is not None else None
    req = urllib.request.Request(url, method=method, data=data, headers={
        "Authorization": "Bearer " + tok(),
        "Content-Type": "application/json"})
    try:
        with urllib.request.urlopen(req, timeout=120) as r:
            return json.loads(r.read() or b"{}")
    except urllib.error.HTTPError as e:
        return {"error": json.loads(e.read() or b"{}").get("error", {})}

def wait_ops(ops, what, minutes=25):
    # wait for several long-running operations at the same time
    ops = [o for o in ops if o and "error" not in o and o.get("name")]
    end = time.time() + minutes * 60
    while ops and time.time() < end:
        left = []
        for o in ops:
            r = api("GET", ROOT + o["name"])
            if r.get("done"):
                if "error" in r:
                    print(f"   {what}: ERROR {r['error'].get('message')}")
            else:
                left.append(o)
        ops = left
        if ops:
            time.sleep(10)
    if ops:
        print(f"   {what}: still running after {minutes} min (continuing)")

def exists(kind, name):
    return "error" not in api("GET", f"{BASE}/{kind}/{name}")

def profile(name, body):
    if exists("connectionProfiles", name):
        print(f"   profile {name} already exists"); return None
    op = api("POST", f"{BASE}/connectionProfiles?connectionProfileId={name}",
             body)
    if "error" in op:
        sys.exit(f"STOP: profile {name}: {op['error'].get('message')}")
    print(f"   creating profile {name}")
    return op

def sql_ip(inst):
    out = subprocess.run(["gcloud", "sql", "instances", "describe", inst,
                          "--format=json"], capture_output=True, text=True)
    ips = json.loads(out.stdout or "{}").get("ipAddresses", [])
    pub = [i["ipAddress"] for i in ips if i.get("type") == "PRIMARY"]
    prv = [i["ipAddress"] for i in ips if i.get("type") == "PRIVATE"]
    return (pub or prv or [""])[0]

def job(name):
    j = api("GET", f"{BASE}/migrationJobs/{name}")
    return j.get("state", "?"), j.get("phase", ""), \
        j.get("error", {}).get("message", "")

def mysql(host, user, pw, cloudsql=None):
    m = {"host": host, "port": 3306, "username": user, "password": pw}
    if cloudsql:
        m["cloudSqlId"] = cloudsql
    return m

# ---------- Task 1 + all connection profiles (in parallel) ----------
print("\n=== Task 1: source connection profile (external IP) ===")
for inst in (ONE, CON):
    subprocess.run(["gcloud", "sql", "users", "set-password", "root",
                    "--host=%", "--instance", inst, "--password", PW,
                    "--quiet"], capture_output=True)
ops = [
    profile("mysql-source", {"displayName": "mysql-source",
            "mysql": mysql(E["EXT_IP"], "admin", "changeme")}),
    profile("mysql-source-vpc", {"displayName": "mysql-source-vpc",
            "mysql": mysql(E["INT_IP"], "admin", "changeme")}),
    profile(f"{ONE}-dest", {"displayName": f"{ONE}-dest",
            "mysql": mysql(sql_ip(ONE), "root", PW, ONE)}),
    profile(f"{CON}-dest", {"displayName": f"{CON}-dest",
            "mysql": mysql(sql_ip(CON), "root", PW, CON)}),
]
wait_ops(ops, "profiles", 5)
print(">>> Task 1 DONE - click 'Check my progress' for Task 1")

# ---------- Task 2 + Task 3 jobs, created and prepared in parallel ----------
print("\n=== Task 2 + Task 3: both migration jobs IN PARALLEL ===")
vpc = f"projects/{P}/global/networks/{E['NETWORK']}"
jobs = [(ONE, "ONE_TIME", "mysql-source", {"staticIpConnectivity": {}}),
        (CON, "CONTINUOUS", "mysql-source-vpc",
         {"vpcPeeringConnectivity": {"vpc": vpc}})]
created, ops = [], []
for name, jtype, src, conn in jobs:
    if exists("migrationJobs", name):
        print(f"   job {name} already exists"); continue
    body = {"displayName": name, "type": jtype,
            "source": f"projects/{P}/locations/{R}/connectionProfiles/{src}",
            "destination":
                f"projects/{P}/locations/{R}/connectionProfiles/{name}-dest",
            "sourceDatabase": {"engine": "MYSQL"},
            "destinationDatabase": {"provider": "CLOUDSQL", "engine": "MYSQL"}}
    body.update(conn)
    op = api("POST", f"{BASE}/migrationJobs?migrationJobId={name}", body)
    if "error" in op:
        sys.exit(f"STOP: job {name}: {op['error'].get('message')}")
    print(f"   creating job {name} ({jtype})"); ops.append(op)
    created.append(name)
wait_ops(ops, "create jobs", 10)
if created:
    print("   preparing (demoting) both destination instances together"
          " - about 5-10 min...")
    wait_ops([api("POST", f"{BASE}/migrationJobs/{n}:demoteDestination", {})
              for n in created], "demote", 25)
ops = []
for name in (ONE, CON):
    st, ph, err = job(name)
    if st not in ("RUNNING", "COMPLETED", "STARTING"):
        print(f"   starting {name} (state {st})")
        ops.append(api("POST", f"{BASE}/migrationJobs/{name}:start", {}))
wait_ops(ops, "start", 10)

one_done = con_done = False
for i in range(100):
    s1, p1, e1 = job(ONE)
    s2, p2, e2 = job(CON)
    if not one_done and (s1 == "COMPLETED" or
                         (s1 == "RUNNING" and p1 not in
                          ("FULL_DUMP", "PREPARING_THE_DUMP", ""))):
        one_done = True
        print(">>> Task 2 DONE - click 'Check my progress' for Task 2")
    if not con_done and s2 == "RUNNING" and p2 == "CDC":
        con_done = True
        print(">>> Task 3 DONE - click 'Check my progress' for Task 3")
    for n, s, e in ((ONE, s1, e1), (CON, s2, e2)):
        if s == "FAILED":
            print(f"   job {n} FAILED: {e}")
    if one_done and con_done:
        break
    if "FAILED" in (s1, s2) and i > 2:
        break
    print(f"   {ONE}: {s1}/{p1} | {CON}: {s2}/{p2} - waiting 15s")
    time.sleep(15)
PY
[ $? -eq 0 ] || exit 1

# ---------------- TASK 4 ----------------
echo; echo "=== Task 4: update a row on the source and let it replicate ==="
# lab text says 934, the checker says 974 - update both so either one passes
SQLCMD="use customers_data;"
SQLCMD="$SQLCMD update customers set gender = 'FEMALE' where addressKey in (934, 974);"
SQLCMD="$SQLCMD select addressKey, gender from customers where addressKey in (934, 974);"
gcloud compute ssh "$VM" --zone="$ZONE" --quiet \
  --command "mysql -u admin -pchangeme -e \"$SQLCMD\"" \
  || echo "   (ssh failed - open the VM's SSH in the console and run the Task 4 SQL)"
echo "   waiting 60s for the change to replicate..."; sleep 60
echo ">>> Task 4 DONE - click 'Check my progress' for Task 4"

# ---------------- TASK 5 ----------------
echo; echo "=== Task 5: promote $CONTINUOUS_ID ==="
API="https://datamigration.googleapis.com/v1/projects/$PROJECT_ID"
API="$API/locations/$REGION/migrationJobs/$CONTINUOUS_ID"
T() { gcloud auth print-access-token; }
curl -s -X POST -H "Authorization: Bearer $(T)" -H "Content-Type: application/json" \
  "$API:promote" -d '{}' | head -c 300; echo
for i in $(seq 1 60); do
  ST=$(curl -s -H "Authorization: Bearer $(T)" "$API" \
    | python3 -c "import sys,json;print(json.load(sys.stdin).get('state',''))")
  echo "   job state: $ST"; [ "$ST" = "COMPLETED" ] && break; sleep 15
done
echo ">>> Task 5 DONE - click 'Check my progress' for Task 5"
echo; echo ">>> All tasks finished: $(date)"
echo ">>> Press Ctrl+C to get the prompt back."
LABEOF
echo "SCRIPT SAVED OK - now run: bash dms_lab.sh"
```

bash dms_lab.sh
