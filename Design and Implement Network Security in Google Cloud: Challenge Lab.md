```/text
set +H
cat > netsec_lab.sh <<'LABEOF'
#!/bin/bash
# Design and Implement Network Security in Google Cloud: Challenge Lab (GSP399)
# Run: bash netsec_lab.sh   (2 questions with auto-filled answers, ~8 min)
# Safe to run again: finished tasks are skipped.
ask() { local r; read -e -r -p "$2 [$3]: " r; printf -v "$1" '%s' "${r:-$3}"; }
die() { echo; echo "STOP: $1 - send Claude a screenshot"; exit 1; }
P=$(gcloud config get-value project 2>/dev/null)
[ -z "$P" ] && die "run: gcloud config set project <Project ID>"
NET=unified-vpc; POL=unified-fw-policy; NAT=unified-nat; RTR=unified-router
D=~/.netsec_done/$P; mkdir -p $D; W=~/netsec_work; mkdir -p $W; cd $W
step() { if [ -f $D/$1 ]; then echo; echo "=== $2: already done ==="; return 1
  fi; echo; echo "=== $2 ==="; }
done_() { touch $D/$1; }
gcloud services enable compute.googleapis.com cloudresourcemanager.googleapis.com \
  iap.googleapis.com -q >/dev/null 2>&1

# ---------- find the values the lab does not print ----------
R=$(gcloud compute routers list --filter="name=$RTR" \
  --format="value(region.basename())" | head -1)
BIP=$(gcloud compute instances list --filter="name~bastion" \
  --format="value(networkInterfaces[0].accessConfigs[0].natIP)" | head -1)
CVM_SUB=$(gcloud compute instances list --filter="name=compromised-vm" \
  --format="value(networkInterfaces[0].subnetwork.basename())" | head -1)
APPSUB=$(gcloud compute networks subnets list --network=$NET \
  --format="value(name)" | grep -m1 -i app)
echo "Project $P | router region ${R:-?}"
echo "=== Values (press Enter to accept the detected ones) ==="
echo "    Bastion = the VM with 'bastion' in its name (Compute Engine > VM instances)"
ask BIP "Bastion external IP" "${BIP:-}"
echo "    Application subnet = the subnet of compromised-vm / named *app*"
ask APPSUB "Application subnet for flow logs" "${APPSUB:-$CVM_SUB}"
[ -z "$BIP" ] && die "bastion external IP is empty"

# =================== TASK 1 ===================
if step t1 "Task 1: legacy firewall rules -> $POL"; then
gcloud compute firewall-rules list --filter="network~/$NET\$" \
  --format=json > legacy.json
echo "   legacy rules on $NET:"
python3 -c 'import json
for r in json.load(open("legacy.json")):
    print("    ", r["name"], r["direction"], r.get("targetTags",""))'

# 1a. tag key + one tag value per network tag used by the legacy rules
KEY=$(gcloud resource-manager tags keys list --parent=projects/$P \
  --format="value(name,shortName)" | awk '$2=="unified-vpc-tags"{print $1}')
[ -z "$KEY" ] && KEY=$(gcloud resource-manager tags keys create unified-vpc-tags \
  --parent=projects/$P --purpose=GCE_FIREWALL \
  --purpose-data=network=$P/$NET --format="value(name)")
[ -z "$KEY" ] && die "could not create the IAM tag key"
echo "   tag key: $KEY (unified-vpc-tags, purpose GCE_FIREWALL)"
TAGS=$(python3 -c 'import json
t=set()
for r in json.load(open("legacy.json")):
    if "egress" in r["name"].lower(): continue
    t.update(r.get("targetTags",[])); t.update(r.get("sourceTags",[]))
print(" ".join(sorted(t)))')
echo "{" > map.json; FIRST=1
for t in $TAGS; do
  V=$(gcloud resource-manager tags values list --parent=$KEY \
    --format="value(name,shortName)" | awk -v s="$t" '$2==s{print $1}')
  [ -z "$V" ] && V=$(gcloud resource-manager tags values create "$t" \
    --parent=$KEY --format="value(name)")
  echo "   network tag '$t' -> $V"
  [ $FIRST = 1 ] || echo "," >> map.json; FIRST=0
  printf '"%s": "%s"\n' "$t" "$V" >> map.json
done
echo "}" >> map.json

# 1b. bind the tag values to every VM that carries the network tag
gcloud compute instances list --filter="networkInterfaces[0].network~/$NET\$" \
  --format="value(name,zone.basename(),id,tags.items.list())" > vms.txt
while read -r VM Z ID VT; do
  for t in ${VT//,/ }; do
    V=$(python3 -c 'import json,sys
print(json.load(open("map.json")).get(sys.argv[1],""))' "$t")
    [ -z "$V" ] && continue
    gcloud resource-manager tags bindings create --tag-value=$V --location=$Z \
      --parent=//compute.googleapis.com/projects/$P/zones/$Z/instances/$ID \
      >/dev/null 2>&1 && echo "   bound $t to $VM"
  done
done < vms.txt

# 1c. migrate (Google's migration tool first, own copy as fallback)
gcloud beta compute firewall-rules migrate --source-network=$NET \
  --target-firewall-policy=$POL --tag-mapping-file=map.json --force -q \
  > migrate.log 2>&1 && echo "   migration tool: OK" \
  || { echo "   migration tool did not finish:"; tail -3 migrate.log; }
gcloud compute network-firewall-policies describe $POL --global \
  >/dev/null 2>&1 || gcloud compute network-firewall-policies create $POL \
  --global --description="Migrated from $NET" -q || die "policy create failed"
N=$(gcloud compute network-firewall-policies describe $POL --global \
  --format=json | python3 -c 'import json,sys
print(sum(1 for r in json.load(sys.stdin).get("rules",[])
          if r.get("priority",0) < 2147483644))')
if [ "${N:-0}" -lt 1 ]; then
  echo "   copying the rules into $POL myself..."
  python3 - "$P" "$POL" > rules.sh <<'PY'
import json, shlex, sys
P, POL = sys.argv[1], sys.argv[2]
m = json.load(open("map.json")); used = set()
for r in sorted(json.load(open("legacy.json")), key=lambda r: r["priority"]):
    if "egress" in r["name"].lower(): continue
    pr = r["priority"]
    while pr in used: pr += 1
    used.add(pr)
    allow = "allowed" in r
    l4 = []
    for x in r.get("allowed", r.get("denied", [])):
        proto = x["IPProtocol"]
        l4 += [f"{proto}:{p}" for p in x.get("ports", [])] or [proto]
    a = ["gcloud", "compute", "network-firewall-policies", "rules", "create",
         str(pr), f"--firewall-policy={POL}", "--global-firewall-policy",
         f"--direction={r['direction']}", "--action=" + ("allow" if allow else "deny"),
         "--layer4-configs=" + ",".join(l4), f"--description={r['name']}", "-q"]
    if r.get("sourceRanges"): a.append("--src-ip-ranges=" + ",".join(r["sourceRanges"]))
    if r.get("destinationRanges"):
        a.append("--dest-ip-ranges=" + ",".join(r["destinationRanges"]))
    tt = [m[t] for t in r.get("targetTags", []) if t in m]
    st = [m[t] for t in r.get("sourceTags", []) if t in m]
    if tt: a.append("--target-secure-tags=" + ",".join(tt))
    if st: a.append("--src-secure-tags=" + ",".join(st))
    if r.get("targetServiceAccounts"):
        a.append("--target-service-accounts=" + ",".join(r["targetServiceAccounts"]))
    if r.get("logConfig", {}).get("enable"): a.append("--enable-logging")
    if r.get("disabled"): a.append("--disabled")
    print(" ".join(shlex.quote(x) for x in a) + " || echo '   FAILED: " + r["name"] + "'")
PY
  bash rules.sh
fi
gcloud compute network-firewall-policies rules list --firewall-policy=$POL \
  --global-firewall-policy 2>/dev/null \
  || gcloud compute network-firewall-policies describe $POL --global \
     --format="table(rules[].priority,rules[].direction,rules[].action)"

# 1d. associate the policy and let it win over classic rules
gcloud compute network-firewall-policies associations list \
  --firewall-policy=$POL --global-firewall-policy 2>/dev/null | grep -q "$NET" \
  || gcloud compute network-firewall-policies associations create \
     --firewall-policy=$POL --network=$NET --global-firewall-policy \
     --name=$POL-$NET -q || die "association failed"
gcloud compute networks update $NET -q \
  --network-firewall-policy-enforcement-order=BEFORE_CLASSIC_FIREWALL \
  || die "enforcement order update failed"
echo "   enforcement order: $(gcloud compute networks describe $NET \
  --format='value(networkFirewallPolicyEnforcementOrder)')"

# 1e. clean up legacy rules, keep the *egress* baseline
OLD=$(python3 -c 'import json
print(" ".join(r["name"] for r in json.load(open("legacy.json"))
               if "egress" not in r["name"].lower()))')
for f in $OLD; do
  gcloud compute firewall-rules update $f --disabled -q >/dev/null 2>&1
done
[ -n "$OLD" ] && gcloud compute firewall-rules delete $OLD -q
echo "   remaining classic rules on $NET:"
gcloud compute firewall-rules list --filter="network~/$NET\$" \
  --format="table(name,direction,disabled)"
echo ">>> Task 1 DONE - click 'Check my progress' (Migrate legacy firewall rules)"
done_ t1; fi

# =================== TASK 2 ===================
if step t2 "Task 2: fix Cloud NAT $NAT"; then
PZ=$(gcloud compute instances list --filter="name=private-instance" \
  --format="value(zone.basename())" | head -1)
PSUB=$(gcloud compute instances describe private-instance --zone=$PZ \
  --format="value(networkInterfaces[0].subnetwork.basename())")
echo "   NAT now uses: $(gcloud compute routers nats describe $NAT --router=$RTR \
  --region=$R --format='value(subnetworks[].name.basename())')"
echo "   private-instance is in subnet: $PSUB (zone $PZ)"
gcloud compute routers nats update $NAT --router=$RTR --region=$R \
  --nat-custom-subnet-ip-ranges=$PSUB -q || die "NAT update failed"
echo "   NAT now uses: $(gcloud compute routers nats describe $NAT --router=$RTR \
  --region=$R --format='value(subnetworks[].name.basename())')"
echo "   testing DNS + internet from private-instance (via IAP)..."
for i in 1 2 3 4; do
  gcloud compute ssh private-instance --zone=$PZ --tunnel-through-iap -q \
    --command="nslookup google.com | tail -3; curl -s -m 10 -o /dev/null \
    -w 'internet HTTP %{http_code}\n' https://www.google.com" \
    2>/dev/null && break
  echo "   (SSH not ready yet, retry $i in 20s)"; sleep 20
done
echo ">>> Task 2 DONE - click 'Check my progress' (Cloud NAT)"
done_ t2; fi

# =================== TASK 3 ===================
if step t3 "Task 3: contain compromised-vm + flow logs"; then
CZ=$(gcloud compute instances list --filter="name=compromised-vm" \
  --format="value(zone.basename())" | head -1)
gcloud compute instances add-tags compromised-vm --zone=$CZ \
  --tags=compromised-vm -q >/dev/null
gcloud compute firewall-rules describe containment-deny-http >/dev/null 2>&1 \
  || gcloud compute firewall-rules create containment-deny-http --network=$NET \
     --direction=INGRESS --action=DENY --rules=tcp:80 --priority=100 \
     --source-ranges=0.0.0.0/0 --target-tags=compromised-vm -q \
  || die "containment-deny-http failed"
gcloud compute firewall-rules describe allow-forensics-ssh >/dev/null 2>&1 \
  || gcloud compute firewall-rules create allow-forensics-ssh --network=$NET \
     --direction=INGRESS --action=ALLOW --rules=tcp:22 --priority=100 \
     --source-ranges=$BIP/32 --target-tags=compromised-vm -q \
  || die "allow-forensics-ssh failed"
SR=$(gcloud compute networks subnets list --filter="name=$APPSUB" \
  --format="value(region.basename())" | head -1)
gcloud compute networks subnets update $APPSUB --region=$SR \
  --enable-flow-logs -q || die "flow logs failed"
gcloud compute firewall-rules list \
  --filter="name=(containment-deny-http,allow-forensics-ssh)" \
  --format="table(name,direction,priority,sourceRanges.list())"
echo "   flow logs on $APPSUB: $(gcloud compute networks subnets describe $APPSUB \
  --region=$SR --format='value(logConfig.enable)')"
echo ">>> Task 3 DONE - click 'Check my progress' (Contain + telemetry)"
done_ t3; fi
echo ">>> All done."
LABEOF
echo "SCRIPT SAVED OK - now run: bash netsec_lab.sh"
```


bash netsec_lab.sh
