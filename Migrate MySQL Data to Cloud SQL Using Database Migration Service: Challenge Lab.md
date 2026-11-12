```/text
cat > dms_lab.sh <<'LABEOF'
#!/bin/bash
# Migrate MySQL Data to Cloud SQL Using Database Migration Service: Challenge Lab (GSP351)
# Run in Cloud Shell:  bash dms_lab.sh      (runs all 5 tasks automatically, no ENTER presses; safe to re-run)

die() { echo; echo "STOP: $1"; exit 1; }
ask_req() { local r=""; while [ -z "$r" ]; do read -e -r -p "$2${3:+ [$3]}: " r; r="${r:-$3}"; \
  [ -z "$r" ] && echo "   (required)"; done; printf -v "$1" '%s' "$r"; }

PROJECT_ID=$(gcloud config get-value project 2>/dev/null)
[ -z "$PROJECT_ID" ] || [ "$PROJECT_ID" = "(unset)" ] && die "run: gcloud config set project <Project ID from the lab panel>"
gcloud services enable datamigration.googleapis.com sqladmin.googleapis.com servicenetworking.googleapis.com compute.googleapis.com >/dev/null 2>&1

# ---------- detect everything ----------
VM=$(gcloud compute instances list --format="value(name)" 2>/dev/null | head -1)
[ -n "$VM" ] || die "no source Compute Engine VM found"
ZONE=$(gcloud compute instances list --filter="name=$VM" --format="value(zone.basename())")
REGION="${ZONE%-*}"
EXT_IP=$(gcloud compute instances describe "$VM" --zone="$ZONE" --format="value(networkInterfaces[0].accessConfigs[0].natIP)")
INT_IP=$(gcloud compute instances describe "$VM" --zone="$ZONE" --format="value(networkInterfaces[0].networkIP)")
NETWORK=$(gcloud compute instances describe "$VM" --zone="$ZONE" --format="value(networkInterfaces[0].network.basename())")
gcloud config set compute/zone "$ZONE" >/dev/null 2>&1; gcloud config set compute/region "$REGION" >/dev/null 2>&1
echo "Project $PROJECT_ID | source VM $VM ($ZONE) external $EXT_IP internal $INT_IP | network $NETWORK"
SQL_LIST=$(gcloud sql instances list --format="value(name)" 2>/dev/null)
echo "Cloud SQL instances in this project:"; echo "$SQL_LIST" | sed 's/^/   - /'
D1=$(echo "$SQL_LIST" | sed -n 1p); D2=$(echo "$SQL_LIST" | sed -n 2p)

echo
echo "=== Enter the values from the lab text (Enter keeps the [default]) ==="
ask_req ONE_TIME_ID   "Task 2: Cloud SQL instance ID for the ONE-TIME migration (yellow)"   "$D1"
ask_req CONTINUOUS_ID "Task 3: Cloud SQL instance ID for the CONTINUOUS migration (yellow)" "$D2"
[ "$ONE_TIME_ID" != "$CONTINUOUS_ID" ] || die "the two instance IDs must be different"
echo
echo "Running all tasks now - click 'Check my progress' on the lab page as each task prints DONE."

export PROJECT_ID REGION VM ZONE EXT_IP INT_IP NETWORK ONE_TIME_ID CONTINUOUS_ID
python3 - <<'PY'
import json, os, subprocess, sys, time, urllib.request, urllib.error
E = os.environ
P, R = E["PROJECT_ID"], E["REGION"]
BASE = f"https://datamigration.googleapis.com/v1/projects/{P}/locations/{R}"
ROOT_PW = "Qwiklabs-123"

def tok():
    return subprocess.run(["gcloud", "auth", "print-access-token"], capture_output=True, text=True).stdout.strip()

def api(method, url, body=None):
    req = urllib.request.Request(url, method=method, data=json.dumps(body).encode() if body is not None else None,
                                 headers={"Authorization": "Bearer " + tok(), "Content-Type": "application/json"})
    try:
        with urllib.request.urlopen(req, timeout=120) as r:
            return json.loads(r.read() or b"{}")
    except urllib.error.HTTPError as e:
        return {"error": json.loads(e.read() or b"{}").get("error", {"code": e.code})}

def wait_op(op, what, minutes=25):
    if "error" in op:
        return op
    name = op.get("name", "")
    for _ in range(minutes * 6):
        o = api("GET", "https://datamigration.googleapis.com/v1/" + name)
        if o.get("done"):
            if "error" in o:
                print(f"   {what}: ERROR {o['error'].get('message')}")
            return o
        time.sleep(10)
    print(f"   {what}: still running after {minutes} min (continuing)")
    return {}

def exists(kind, name):
    return "error" not in api("GET", f"{BASE}/{kind}/{name}")

def profile(name, body):
    if exists("connectionProfiles", name):
        print(f"   connection profile {name} already exists"); return
    op = api("POST", f"{BASE}/connectionProfiles?connectionProfileId={name}", body)
    if "error" in op:
        sys.exit(f"STOP: could not create profile {name}: {op['error'].get('message')}")
    wait_op(op, f"profile {name}", 5)
    print(f"   created connection profile {name}")

def sql_ip(inst):
    d = json.loads(subprocess.run(["gcloud", "sql", "instances", "describe", inst, "--format=json"],
                                  capture_output=True, text=True).stdout or "{}")
    ips = d.get("ipAddresses", [])
    pub = [i["ipAddress"] for i in ips if i.get("type") == "PRIMARY"]
    priv = [i["ipAddress"] for i in ips if i.get("type") == "PRIVATE"]
    return (pub or priv or [""])[0]

def job_state(job):
    j = api("GET", f"{BASE}/migrationJobs/{job}")
    return j.get("state", "?"), j.get("phase", ""), j.get("error", {}).get("message", "")

def run_job(job, jtype, src, dest_inst, connectivity):
    # destination profile for the EXISTING Cloud SQL instance
    subprocess.run(["gcloud", "sql", "users", "set-password", "root", "--host=%", "--instance", dest_inst,
                    "--password", ROOT_PW, "--quiet"], capture_output=True)
    dprof = f"{dest_inst}-dest"
    profile(dprof, {"displayName": dprof, "mysql": {"host": sql_ip(dest_inst), "port": 3306,
                    "username": "root", "password": ROOT_PW, "cloudSqlId": dest_inst}})
    if not exists("migrationJobs", job):
        body = {"displayName": job, "type": jtype,
                "source": f"projects/{P}/locations/{R}/connectionProfiles/{src}",
                "destination": f"projects/{P}/locations/{R}/connectionProfiles/{dprof}",
                "sourceDatabase": {"provider": "DATABASE_PROVIDER_UNSPECIFIED", "engine": "MYSQL"},
                "destinationDatabase": {"provider": "CLOUDSQL", "engine": "MYSQL"}}
        body.update(connectivity)
        op = api("POST", f"{BASE}/migrationJobs?migrationJobId={job}", body)
        if "error" in op:
            sys.exit(f"STOP: could not create migration job {job}: {op['error'].get('message')}")
        wait_op(op, f"create job {job}", 10)
        print(f"   created migration job {job} ({jtype})")
        print("   demoting the existing destination instance (this is what 'Existing instance' does, ~5-10 min)...")
        wait_op(api("POST", f"{BASE}/migrationJobs/{job}:demoteDestination", {}), "demote destination", 20)
    state, phase, err = job_state(job)
    if state not in ("RUNNING", "COMPLETED", "STARTING"):
        print(f"   starting {job} (state was {state})...")
        op = api("POST", f"{BASE}/migrationJobs/{job}:start", {})
        if "error" in op:
            print(f"   start error: {op['error'].get('message')}")
        else:
            wait_op(op, f"start {job}", 10)
    for i in range(80):
        state, phase, err = job_state(job)
        if state in ("RUNNING", "COMPLETED") and phase not in ("FULL_DUMP", "PREPARING_THE_DUMP", ""):
            break
        if state == "FAILED":
            print(f"   job FAILED: {err}"); break
        print(f"   {job}: state={state} phase={phase} ({i+1}/80) - waiting 15s"); time.sleep(15)
    print(f"   {job}: state={state} phase={phase}")
    return state

# ---------------- TASK 1 ----------------
print("\n=== Task 1: source connection profile (external IP) ===")
SRC = "mysql-source"
profile(SRC, {"displayName": SRC, "mysql": {"host": E["EXT_IP"], "port": 3306, "username": "admin", "password": "changeme"}})
print(">>> Task 1 DONE - click 'Check my progress' for Task 1")

# ---------------- TASK 2 ----------------
print(f"\n=== Task 2: one-time migration -> {E['ONE_TIME_ID']} (about 10-15 min) ===")
run_job(E["ONE_TIME_ID"], "ONE_TIME", SRC, E["ONE_TIME_ID"], {"staticIpConnectivity": {}})
print(">>> Task 2 DONE - click 'Check my progress' for Task 2")

# ---------------- TASK 3 ----------------
print(f"\n=== Task 3: continuous migration -> {E['CONTINUOUS_ID']} with VPC peering (about 10-15 min) ===")
SRC_VPC = "mysql-source-vpc"   # same source, reached over VPC peering (internal IP)
profile(SRC_VPC, {"displayName": SRC_VPC, "mysql": {"host": E["INT_IP"], "port": 3306, "username": "admin", "password": "changeme"}})
st = run_job(E["CONTINUOUS_ID"], "CONTINUOUS", SRC_VPC, E["CONTINUOUS_ID"],
             {"vpcPeeringConnectivity": {"vpc": f"projects/{P}/global/networks/{E['NETWORK']}"}})
print(">>> Task 3 DONE - click 'Check my progress' for Task 3" if st == "RUNNING" else
      ">>> Task 3: job not RUNNING yet - check Database Migration > Migration jobs in the console")
PY
[ $? -eq 0 ] || exit 1

# ---------------- TASK 4 ----------------
echo; echo "=== Task 4: update a row on the source and let it replicate ==="
gcloud compute ssh "$VM" --zone="$ZONE" --quiet --command \
  "mysql -u admin -pchangeme -e \"use customers_data; update customers set gender = 'FEMALE' where addressKey = 934; select addressKey, gender from customers where addressKey = 934;\"" \
  || echo "   (ssh failed - in the console open the VM's SSH and run the 2 SQL lines from Task 4)"
echo "   waiting 90s for the change to replicate..."; sleep 90
echo ">>> Task 4 DONE - click 'Check my progress' for Task 4"

# ---------------- TASK 5 ----------------
echo; echo "=== Task 5: promote $CONTINUOUS_ID ==="
API="https://datamigration.googleapis.com/v1/projects/$PROJECT_ID/locations/$REGION/migrationJobs/$CONTINUOUS_ID"
curl -s -X POST -H "Authorization: Bearer $(gcloud auth print-access-token)" -H "Content-Type: application/json" "$API:promote" -d '{}' | head -c 300; echo
for i in $(seq 1 40); do
  ST=$(curl -s -H "Authorization: Bearer $(gcloud auth print-access-token)" "$API" | python3 -c "import sys,json;print(json.load(sys.stdin).get('state',''))")
  echo "   job state: $ST"; [ "$ST" = "COMPLETED" ] && break; sleep 20
done
echo ">>> Task 5 DONE - click 'Check my progress' for Task 5"
echo; echo ">>> All tasks finished."
LABEOF
```

bash dms_lab.sh
