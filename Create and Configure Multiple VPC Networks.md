```/text
cat > vpc_lab.sh <<'LABEOF'
#!/bin/bash
# Create and Configure Multiple VPC Networks (GSP211) - universal solution
# Run in Cloud Shell:  bash vpc_lab.sh
# Safe to re-run: anything that already exists is skipped.

SCRIPT_PATH=$(readlink -f "$0")

PROJECT_ID=$(gcloud config get-value project 2>/dev/null)
if [ -z "$PROJECT_ID" ] || [ "$PROJECT_ID" = "(unset)" ]; then
  echo "ERROR: no project set. Run: gcloud config set project <Project ID from the lab panel>"
  exit 1
fi
export GOOGLE_CLOUD_PROJECT="$PROJECT_ID"
echo "Project: $PROJECT_ID"

VARS_FILE="$HOME/.vpc_lab_vars"
[ -f "$VARS_FILE" ] && . "$VARS_FILE"

ask() {   # ask "prompt" "default" VARNAME
  local ans
  read -r -p "$1 [$2]: " ans
  ans=$(echo "${ans:-$2}" | xargs)
  printf -v "$3" '%s' "$ans"
}
fail() { echo; echo "FAIL: $1"; echo "Fix the error above, then re-run: bash $SCRIPT_PATH  (finished steps are skipped)"; exit 1; }

# ---------- detect regions and zone from the pre-built mynet-vm-1 / mynet-vm-2 ----------
meta() { gcloud compute project-info describe --quiet --format="value(commonInstanceMetadata.items[$1])" 2>/dev/null </dev/null; }
vm_zone() { gcloud compute instances list --filter="name=$1" --format="value(zone.basename())" 2>/dev/null </dev/null | head -n 1; }
DET_ZONE=$(vm_zone mynet-vm-1)
[ -z "$DET_ZONE" ] && DET_ZONE=$(meta google-compute-default-zone)
DET_REGION1=""; [ -n "$DET_ZONE" ] && DET_REGION1="${DET_ZONE%-*}"
[ -z "$DET_REGION1" ] && DET_REGION1=$(meta google-compute-default-region)
Z2=$(vm_zone mynet-vm-2); DET_REGION2=""; [ -n "$Z2" ] && DET_REGION2="${Z2%-*}"

echo
echo "=== Copy each value from the lab text (yellow), or press ENTER to accept the default ==="
ask "Region 1 -> Task 1, 'Create the managementnet network' step 4 (Region of managementsubnet-1)" "${REGION1:-${DET_REGION1:-us-central1}}" REGION1
ask "Region 2 -> Task 1, privatenet commands (--region of privatesubnet-2)                      " "${REGION2:-${DET_REGION2:-europe-west1}}" REGION2
ZDEF="${ZONE:-$DET_ZONE}"
case "$ZDEF" in "${REGION1}"-*) ;; *) ZDEF="${REGION1}-a" ;; esac
ask "Zone     -> Task 2, privatenet-vm-1 command (--zone=...)                                    " "$ZDEF" ZONE
case "$ZONE" in "${REGION1}"-*) ;; *) echo "WARNING: zone $ZONE is not in region $REGION1 - check the lab text" ;; esac
{ for v in REGION1 REGION2 ZONE; do printf '%s=%q\n' "$v" "${!v}"; done; } > "$VARS_FILE"

gcloud config set compute/region "$REGION1" >/dev/null 2>&1
gcloud config set compute/zone "$ZONE" >/dev/null 2>&1

have_net()    { gcloud compute networks describe "$1" >/dev/null 2>&1 </dev/null; }
have_subnet() { gcloud compute networks subnets describe "$1" --region="$2" >/dev/null 2>&1 </dev/null; }
have_fw()     { gcloud compute firewall-rules describe "$1" >/dev/null 2>&1 </dev/null; }
have_vm()     { gcloud compute instances describe "$1" --zone="$ZONE" >/dev/null 2>&1 </dev/null; }
skip() { echo "   $1 already exists - skipping"; }

# ---------- TASK 1: managementnet ----------
echo; echo "=== TASK 1a: managementnet + managementsubnet-1 ($REGION1, 10.130.0.0/20) ==="
have_net managementnet && skip managementnet || \
  gcloud compute networks create managementnet --subnet-mode=custom </dev/null || fail "could not create managementnet"
have_subnet managementsubnet-1 "$REGION1" && skip managementsubnet-1 || \
  gcloud compute networks subnets create managementsubnet-1 --network=managementnet --region="$REGION1" --range=10.130.0.0/20 </dev/null || fail "could not create managementsubnet-1"

# ---------- TASK 1: privatenet ----------
echo; echo "=== TASK 1b: privatenet + privatesubnet-1 ($REGION1) + privatesubnet-2 ($REGION2) ==="
have_net privatenet && skip privatenet || \
  gcloud compute networks create privatenet --subnet-mode=custom </dev/null || fail "could not create privatenet"
have_subnet privatesubnet-1 "$REGION1" && skip privatesubnet-1 || \
  gcloud compute networks subnets create privatesubnet-1 --network=privatenet --region="$REGION1" --range=172.16.0.0/24 </dev/null || fail "could not create privatesubnet-1"
have_subnet privatesubnet-2 "$REGION2" && skip privatesubnet-2 || \
  gcloud compute networks subnets create privatesubnet-2 --network=privatenet --region="$REGION2" --range=172.20.0.0/20 </dev/null || fail "could not create privatesubnet-2"

# ---------- TASK 1: firewall rules ----------
echo; echo "=== TASK 1c: firewall rules (icmp, tcp:22, tcp:3389 from 0.0.0.0/0) ==="
for net in managementnet privatenet; do
  have_fw "$net-allow-icmp-ssh-rdp" && skip "$net-allow-icmp-ssh-rdp" || \
    gcloud compute firewall-rules create "$net-allow-icmp-ssh-rdp" --direction=INGRESS --priority=1000 \
      --network="$net" --action=ALLOW --rules=icmp,tcp:22,tcp:3389 --source-ranges=0.0.0.0/0 </dev/null \
      || fail "could not create $net-allow-icmp-ssh-rdp"
done

# ---------- TASK 2: VM instances ----------
echo; echo "=== TASK 2: managementnet-vm-1 and privatenet-vm-1 (e2-micro, $ZONE) ==="
have_vm managementnet-vm-1 && skip managementnet-vm-1 || \
  gcloud compute instances create managementnet-vm-1 --zone="$ZONE" --machine-type=e2-micro --subnet=managementsubnet-1 </dev/null \
    || fail "could not create managementnet-vm-1"
have_vm privatenet-vm-1 && skip privatenet-vm-1 || \
  gcloud compute instances create privatenet-vm-1 --zone="$ZONE" --machine-type=e2-micro --subnet=privatesubnet-1 </dev/null \
    || fail "could not create privatenet-vm-1"

# ---------- TASK 4: VM with three network interfaces ----------
echo; echo "=== TASK 4: vm-appliance (e2-standard-4, nic0 privatesubnet-1, nic1 managementsubnet-1, nic2 mynetwork) ==="
have_vm vm-appliance && skip vm-appliance || \
  gcloud compute instances create vm-appliance --zone="$ZONE" --machine-type=e2-standard-4 \
    --network-interface=subnet=privatesubnet-1 \
    --network-interface=subnet=managementsubnet-1,no-address \
    --network-interface=subnet=mynetwork,no-address </dev/null \
    || fail "could not create vm-appliance"

# ---------- overview ----------
echo
gcloud compute networks subnets list --sort-by=NETWORK --filter="network~(managementnet|privatenet)" </dev/null
echo
gcloud compute instances list --sort-by=ZONE </dev/null

# ---------- SELF-CHECK ----------
echo
echo "=================== SELF-CHECK ==================="
ok()  { echo "  PASS  $1"; }
bad() { echo "  FAIL  $1"; }
r=$(gcloud compute networks subnets describe managementsubnet-1 --region="$REGION1" --format="value(ipCidrRange)" 2>/dev/null </dev/null)
[ "$r" = "10.130.0.0/20" ] && ok "managementnet / managementsubnet-1 10.130.0.0/20" || bad "managementsubnet-1 missing or wrong range"
r=$(gcloud compute networks subnets describe privatesubnet-1 --region="$REGION1" --format="value(ipCidrRange)" 2>/dev/null </dev/null)
[ "$r" = "172.16.0.0/24" ] && ok "privatenet / privatesubnet-1 172.16.0.0/24" || bad "privatesubnet-1 missing or wrong range"
r=$(gcloud compute networks subnets describe privatesubnet-2 --region="$REGION2" --format="value(ipCidrRange)" 2>/dev/null </dev/null)
[ "$r" = "172.20.0.0/20" ] && ok "privatenet / privatesubnet-2 172.20.0.0/20" || bad "privatesubnet-2 missing or wrong range"
for f in managementnet-allow-icmp-ssh-rdp privatenet-allow-icmp-ssh-rdp; do
  have_fw "$f" && ok "firewall rule $f" || bad "firewall rule $f missing"
done
for v in managementnet-vm-1 privatenet-vm-1; do
  have_vm "$v" && ok "VM $v" || bad "VM $v missing"
done
n=$(gcloud compute instances describe vm-appliance --zone="$ZONE" --format="value(networkInterfaces[].name)" 2>/dev/null </dev/null | tr ';' '\n' | grep -c nic)
[ "$n" = 3 ] && ok "vm-appliance has 3 network interfaces" || bad "vm-appliance missing or does not have 3 interfaces"
echo
echo ">>> Done. On the lab page click 'Check my progress' for all 7 checkpoints."
echo "    (Task 3 and the ping tests in Task 4 are not graded - they are done from the VMs' SSH windows if you want to try them.)"
LABEOF
```


bash vpc_lab.sh
