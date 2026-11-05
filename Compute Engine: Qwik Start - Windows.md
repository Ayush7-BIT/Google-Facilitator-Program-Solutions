```/text
cat > windows_lab.sh <<'LABEOF'
#!/bin/bash
# Compute Engine: Qwik Start - Windows (GSP093) - universal solution

PROJECT_ID=$(gcloud config get-value project 2>/dev/null)
if [ -z "$PROJECT_ID" ] || [ "$PROJECT_ID" = "(unset)" ]; then
  echo "ERROR: no project set. Run: gcloud config set project <Project ID from the lab panel>"
  exit 1
fi
export GOOGLE_CLOUD_PROJECT="$PROJECT_ID"
echo "Project: $PROJECT_ID"

ask() {   # ask "prompt" "default" VARNAME
  local ans
  read -r -p "$1 [$2]: " ans
  printf -v "$3" '%s' "${ans:-$2}"
}

# ---------- detect region and zone ----------
meta() { gcloud compute project-info describe --quiet --format="value(commonInstanceMetadata.items[$1])" 2>/dev/null </dev/null; }
DET_ZONE=$(gcloud config get-value compute/zone 2>/dev/null)
if [ -z "$DET_ZONE" ] || [ "$DET_ZONE" = "(unset)" ]; then DET_ZONE=$(meta google-compute-default-zone); fi
DET_REGION=$(gcloud config get-value compute/region 2>/dev/null)
if [ -z "$DET_REGION" ] || [ "$DET_REGION" = "(unset)" ]; then DET_REGION=$(meta google-compute-default-region); fi
if [ -z "$DET_REGION" ] && [ -n "$DET_ZONE" ]; then DET_REGION="${DET_ZONE%-*}"; fi
if [ -z "$DET_REGION" ]; then
  echo "Could not detect the region/zone - type the ones from the lab text."
  DET_REGION="us-central1"
fi

echo
echo "=== Press ENTER to accept each default (check them against the lab text) ==="
ask "Region (Task 1, 'Region' row)                    " "$DET_REGION" REGION
case "$DET_ZONE" in "${REGION}"-*) ZDEF="$DET_ZONE" ;; *) ZDEF="${REGION}-b" ;; esac
ask "Zone (Task 1, 'Zone' row)                        " "$ZDEF" ZONE
case "$ZONE" in "${REGION}"-*) ;; *) echo "WARNING: zone $ZONE is not in region $REGION - check the lab text" ;; esac
ask "VM name (your choice - the lab does not fix one) " "windows-vm" VM
ask "Machine type (Series E2)                         " "e2-medium" MACHINE
ask "Windows Server version as written in the lab     " "Windows Server 2022 Datacenter" OS_TEXT
ask "Windows username (lab says: admin)               " "admin" RDP_USER
ask "Pause so you can click 'Check my progress'? (y/n)" "y" PAUSE

# ---------- map the version text to an image family ----------
os_lc=$(echo "$OS_TEXT" | tr 'A-Z' 'a-z')
case "$os_lc" in
  *2025*) FAM="windows-2025" ;;
  *2022*) FAM="windows-2022" ;;
  *2019*) FAM="windows-2019" ;;
  *2016*) FAM="windows-2016" ;;
  *)      FAM="windows-2022" ;;
esac
case "$os_lc" in *core*) FAM="${FAM}-core" ;; esac
if ! gcloud compute images describe-from-family "$FAM" --project=windows-cloud >/dev/null 2>&1; then
  echo "Image family $FAM not found - using windows-2022 instead"
  FAM="windows-2022"
fi

pause() {
  if [ "$PAUSE" = "y" ]; then
    read -r -p ">>> Lab page: click 'Check my progress' for: $1 -- then press ENTER here to continue " _
  fi
}

gcloud config set compute/region "$REGION" >/dev/null 2>&1
gcloud config set compute/zone "$ZONE" >/dev/null 2>&1
gcloud services enable compute.googleapis.com --quiet

# ---------- TASK 1: create the Windows VM ----------
echo
echo "=== TASK 1: create VM $VM ($MACHINE, image $FAM) in $ZONE ==="
gcloud compute firewall-rules describe default-allow-rdp >/dev/null 2>&1 \
  || gcloud compute firewall-rules create default-allow-rdp --network=default --direction=INGRESS \
       --action=ALLOW --rules=tcp:3389 --source-ranges=0.0.0.0/0 --quiet

if gcloud compute instances describe "$VM" --zone="$ZONE" >/dev/null 2>&1; then
  echo "VM $VM already exists"
else
  gcloud compute instances create "$VM" --zone="$ZONE" --machine-type="$MACHINE" \
    --image-family="$FAM" --image-project=windows-cloud \
    --boot-disk-size=50GB --boot-disk-type=pd-balanced || { echo "FAIL: could not create the VM"; exit 1; }
fi
IP=$(gcloud compute instances describe "$VM" --zone="$ZONE" --format="get(networkInterfaces[0].accessConfigs[0].natIP)" 2>/dev/null)
echo "External IP: $IP"
pause "Create a virtual machine instance (50)"

# ---------- TASK 2: wait for Windows, reset the password ----------
echo
echo "=== TASK 2: wait until Windows is ready (usually 3-6 minutes) ==="
ready=0
for i in $(seq 1 45); do
  if gcloud compute instances get-serial-port-output "$VM" --zone="$ZONE" --quiet 2>/dev/null </dev/null | grep -q "Instance setup finished"; then
    ready=1; break
  fi
  echo "   Windows is still starting ($i/45) - waiting 20s"; sleep 20
done
[ "$ready" = 1 ] && echo "Windows reports: Instance setup finished - ready to use." \
                 || echo "WARNING: did not see 'Instance setup finished' yet - trying the password reset anyway"

echo
echo "=== Setting the Windows password for user '$RDP_USER' ==="
PASSWORD=""
for i in 1 2 3 4 5 6; do
  CREDS=$(gcloud compute reset-windows-password "$VM" --zone="$ZONE" --user="$RDP_USER" --quiet --format=json 2>reset_err.txt </dev/null)
  PASSWORD=$(echo "$CREDS" | python3 -c 'import sys,json; print(json.load(sys.stdin).get("password",""))' 2>/dev/null)
  [ -n "$PASSWORD" ] && break
  echo "   password reset not ready yet (try $i/6): $(tail -n 2 reset_err.txt | head -c 200)"
  sleep 30
done

echo
echo "=============== RDP LOGIN DETAILS (valid only for this lab) ==============="
echo " IP address : $IP"
echo " Username   : $RDP_USER"
echo " Password   : ${PASSWORD:-<password reset failed - see the error above>}"
echo "==========================================================================="
echo "Copy the password now. Connect with an RDP client to the IP above (see the steps in the answer)."
pause "RDP into the Windows Server (50)"

# ---------- SELF-CHECK ----------
echo
echo "=================== SELF-CHECK ==================="
ok()  { echo "  PASS  $1"; }
bad() { echo "  FAIL  $1"; }
chk() { echo "  CHECK $1"; }
INFO=$(gcloud compute instances describe "$VM" --zone="$ZONE" --format=json 2>/dev/null)
echo "$INFO" | grep -q '"status": "RUNNING"' && ok "Task 1: VM $VM is RUNNING" || bad "Task 1: VM is not running"
echo "$INFO" | grep -qi "windows" && ok "Task 1: the boot disk uses a Windows image" || bad "Task 1: no Windows license found on the boot disk"
echo "$INFO" | grep -q "series\|machineType" && echo "$INFO" | grep -q "$MACHINE" && ok "Task 1: machine type $MACHINE (E2 series)" || chk "Task 1: machine type is not $MACHINE"
gcloud compute instances get-serial-port-output "$VM" --zone="$ZONE" --quiet 2>/dev/null </dev/null | grep -q "Instance setup finished" \
  && ok "Task 2: Windows finished its setup" || chk "Task 2: Windows setup not confirmed yet"
echo "$INFO" | grep -q "windows-keys" && ok "Task 2: Windows password was set (windows-keys metadata present)" || bad "Task 2: no windows-keys metadata - password reset did not work"
[ -n "$PASSWORD" ] && ok "Task 2: password received for $RDP_USER" || bad "Task 2: no password received"
timeout 8 bash -c "</dev/tcp/${IP}/3389" 2>/dev/null && ok "Task 2: RDP port 3389 is reachable on $IP" || chk "Task 2: port 3389 not reachable from Cloud Shell yet"
echo
echo ">>> Done. Click 'Check my progress' for any of the 2 checkpoints that is not green yet."
LABEOF
```

bash windows_lab.sh
