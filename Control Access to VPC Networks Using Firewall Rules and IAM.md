```/text
cat > fw_iam_lab.sh <<'LABEOF'
#!/bin/bash
# Control Access to VPC Networks Using Firewall Rules and IAM (GSP213) - universal solution
# Run in Cloud Shell:  bash fw_iam_lab.sh
# Safe to re-run: anything that already exists is skipped.
# All 6 graded checkpoints can be clicked at the pause OR at the very end (the firewall rule is restored at the end).

SCRIPT_PATH=$(readlink -f "$0")

PROJECT_ID=$(gcloud config get-value project 2>/dev/null)
if [ -z "$PROJECT_ID" ] || [ "$PROJECT_ID" = "(unset)" ]; then
  echo "ERROR: no project set. Run: gcloud config set project <Project ID from the lab panel>"
  exit 1
fi
export GOOGLE_CLOUD_PROJECT="$PROJECT_ID"
echo "Project: $PROJECT_ID"

# remember the student account and always switch back to it when the script ends
STUDENT_ACCOUNT=$(gcloud config get-value account 2>/dev/null)
case "$STUDENT_ACCOUNT" in network-admin@*) STUDENT_ACCOUNT=$(gcloud auth list --format="value(account)" 2>/dev/null </dev/null | grep -v 'gserviceaccount.com' | head -n 1) ;; esac
restore_account() { [ -n "$STUDENT_ACCOUNT" ] && gcloud config set account "$STUDENT_ACCOUNT" >/dev/null 2>&1; }
trap restore_account EXIT
restore_account
echo "Student account: $STUDENT_ACCOUNT"

VARS_FILE="$HOME/.fw_iam_lab_vars"
[ -f "$VARS_FILE" ] && . "$VARS_FILE"

ask() {   # ask "prompt" "default" VARNAME
  local ans
  read -r -p "$1 [$2]: " ans
  ans=$(echo "${ans:-$2}" | xargs)
  printf -v "$3" '%s' "$ans"
}
fail() { echo; echo "FAIL: $1"; echo "Fix the error above, then re-run: bash $SCRIPT_PATH  (finished steps are skipped)"; exit 1; }
skip() { echo "   $1 already exists - skipping"; }

# ---------- detect the zone ----------
meta() { gcloud compute project-info describe --quiet --format="value(commonInstanceMetadata.items[$1])" 2>/dev/null </dev/null; }
DET_ZONE=$(gcloud config get-value compute/zone 2>/dev/null)
if [ -z "$DET_ZONE" ] || [ "$DET_ZONE" = "(unset)" ]; then DET_ZONE=$(meta google-compute-default-zone); fi
if [ -z "$DET_ZONE" ]; then DET_ZONE="us-central1-a"; echo "Could not detect the zone - type the one from the lab text."; fi

echo
echo "=== Copy each value from the lab text (yellow), or press ENTER to accept the default ==="
ask "Zone -> Task 1, blue server table ('Zone' row) - same zone for green and test-vm" "${ZONE:-$DET_ZONE}" ZONE
{ printf 'ZONE=%q\n' "$ZONE"; } > "$VARS_FILE"
REGION="${ZONE%-*}"
gcloud config set compute/region "$REGION" >/dev/null 2>&1
gcloud config set compute/zone "$ZONE" >/dev/null 2>&1

have_vm() { gcloud compute instances describe "$1" --zone="$ZONE" >/dev/null 2>&1 </dev/null; }
have_fw() { gcloud compute firewall-rules describe "$1" >/dev/null 2>&1 </dev/null; }

make_startup() {   # make_startup COLOR FILE
  cat > "$2" <<EOF
#!/bin/bash
apt-get update
apt-get install -y nginx-light
echo "<h1>Welcome to the $1 server!</h1>" > /var/www/html/index.nginx-debian.html
EOF
}

# ---------- TASK 1: blue and green web servers ----------
echo; echo "=== TASK 1: create blue (tag web-server) and green (no tag) with nginx startup scripts ==="
for color in blue green; do
  if have_vm "$color"; then skip "$color"; continue; fi
  make_startup "$color" "/tmp/startup-$color.sh"
  TAGS=""; [ "$color" = blue ] && TAGS="--tags=web-server"
  gcloud compute instances create "$color" --zone="$ZONE" --machine-type=e2-medium $TAGS \
    --metadata-from-file=startup-script="/tmp/startup-$color.sh" </dev/null || fail "could not create $color"
done

# ---------- TASK 2: tagged firewall rule + test-vm ----------
echo; echo "=== TASK 2a: firewall rule allow-http-web-server (tcp:80, icmp -> tag web-server) ==="
have_fw allow-http-web-server && skip allow-http-web-server || \
  gcloud compute firewall-rules create allow-http-web-server --network=default --direction=INGRESS \
    --action=ALLOW --rules=tcp:80,icmp --target-tags=web-server --source-ranges=0.0.0.0/0 </dev/null \
    || fail "could not create allow-http-web-server"

echo; echo "=== TASK 2b: test-vm ==="
have_vm test-vm && skip test-vm || \
  gcloud compute instances create test-vm --machine-type=e2-micro --subnet=default --zone="$ZONE" </dev/null \
    || fail "could not create test-vm"

# ---------- wait for nginx on both servers (max 3 minutes) ----------
BLUE_EXTERNAL_IP=$(gcloud compute instances describe blue --zone="$ZONE" --format='get(networkInterfaces[0].accessConfigs[0].natIP)' </dev/null)
GREEN_EXTERNAL_IP=$(gcloud compute instances describe green --zone="$ZONE" --format='get(networkInterfaces[0].accessConfigs[0].natIP)' </dev/null)
# newer images log 'Script "startup-script" finished with exit status 0', older ones 'startup-script exit status 0'
script_done() { gcloud compute instances get-serial-port-output "$1" --zone="$ZONE" 2>/dev/null </dev/null \
                  | grep -Eqi 'startup-script.*exit status 0|finished running startup scripts'; }
echo; echo "=== Waiting for nginx on blue and green (usually 1-2 minutes, max 3) ==="
blue_ok=0; green_ok=0
for i in $(seq 1 18); do
  [ "$blue_ok" = 0 ] && curl -s -m 3 "http://$BLUE_EXTERNAL_IP" 2>/dev/null | grep -q "blue server" && blue_ok=1
  [ "$green_ok" = 0 ] && script_done green && green_ok=1
  [ "$blue_ok" = 1 ] && [ "$green_ok" = 1 ] && break
  echo "   ($i/18) blue: $([ "$blue_ok" = 1 ] && echo ready || echo installing)  green: $([ "$green_ok" = 1 ] && echo ready || echo installing) - waiting 10s"
  sleep 10
done
echo "   blue:  $([ "$blue_ok" = 1 ] && echo 'nginx ready' || echo 'not confirmed yet')"
echo "   green: $([ "$green_ok" = 1 ] && echo 'nginx ready' || echo 'not confirmed from its log - it is normally ready by now')"

echo; echo "=== TASK 2c: test HTTP from Cloud Shell ==="
echo "blue  ($BLUE_EXTERNAL_IP):  $(curl -s -m 5 "http://$BLUE_EXTERNAL_IP" || echo 'no response')"
echo "green ($GREEN_EXTERNAL_IP): $(curl -s -m 5 "http://$GREEN_EXTERNAL_IP" || echo 'no response (expected - green has no web-server tag)')"

# ---------- TASK 3: network-admin service account + Compute Network Admin ----------
SA="network-admin@$PROJECT_ID.iam.gserviceaccount.com"
echo; echo "=== TASK 3a: service account $SA with role Compute Network Admin ==="
if gcloud iam service-accounts describe "$SA" >/dev/null 2>&1 </dev/null; then
  skip "$SA"
else
  gcloud iam service-accounts create network-admin --display-name="Network-admin" </dev/null || fail "could not create the network-admin service account"
  sleep 10
fi
gcloud projects add-iam-policy-binding "$PROJECT_ID" --member="serviceAccount:$SA" \
  --role="roles/compute.networkAdmin" --condition=None --quiet >/dev/null </dev/null \
  && echo "   granted roles/compute.networkAdmin" || fail "could not grant Compute Network Admin"

# ---------- CHECKPOINTS (before anything is deleted) ----------
echo
echo ">>> Lab page: click 'Check my progress' for ALL 6 checkpoints now:"
echo "      Create the blue server / Create the green server / Install Nginx and customize the welcome page"
echo "      Create the tagged firewall rule / Create a test-vm / Create a Network-admin service account"
read -r -p ">>> Press ENTER to continue (the next, ungraded part deletes the rule for a demo - it is re-created at the end) " _

# ---------- TASK 3 (not graded): test as network-admin, then Security Admin can delete ----------
echo; echo "=== TASK 3b: act as network-admin: list works, delete is denied ==="
activated=0
if gcloud iam service-accounts keys create "$HOME/credentials.json" --iam-account="$SA" </dev/null; then
  echo "   waiting for the new key to become active (a new key can take up to a minute)"
  for i in 1 2 3 4 5 6; do
    sleep 15
    if gcloud auth activate-service-account --key-file="$HOME/credentials.json" >/dev/null 2>&1 </dev/null; then activated=1; break; fi
    echo "   key not active yet ($i/6)"
  done
fi
if [ "$activated" = 1 ]; then
  echo "   now acting as $SA"
  gcloud compute firewall-rules list </dev/null
  echo "   NOTE: the next delete is SUPPOSED to fail with a red 'ERROR ... compute.firewalls.delete' -"
  echo "         that is the lab's demo: Compute Network Admin may not delete firewall rules."
  gcloud compute firewall-rules delete allow-http-web-server --quiet </dev/null \
    && echo "   (unexpected: delete worked)" || echo "   delete denied - expected for Compute Network Admin"
  restore_account

  echo; echo "=== TASK 3c: switch the role to Compute Security Admin, then delete the rule as network-admin ==="
  gcloud projects add-iam-policy-binding "$PROJECT_ID" --member="serviceAccount:$SA" \
    --role="roles/compute.securityAdmin" --condition=None --quiet >/dev/null </dev/null
  gcloud projects remove-iam-policy-binding "$PROJECT_ID" --member="serviceAccount:$SA" \
    --role="roles/compute.networkAdmin" --condition=None --quiet >/dev/null </dev/null
  echo "   waiting 60s for the role change to take effect"; sleep 60
  gcloud config set account "$SA" >/dev/null 2>&1
  deleted=0
  for i in 1 2 3 4; do
    if gcloud compute firewall-rules delete allow-http-web-server --quiet </dev/null; then deleted=1; break; fi
    echo "   not allowed yet ($i/4) - waiting 30s"; sleep 30
  done
  restore_account
  [ "$deleted" = 1 ] && echo "   allow-http-web-server deleted by network-admin (Compute Security Admin)"
  echo "blue ($BLUE_EXTERNAL_IP) now: $(curl -s -m 3 "http://$BLUE_EXTERNAL_IP" || echo 'no response (expected - rule deleted)')"
else
  restore_account
  echo "   Could not create a key for the service account (this part is not graded) - skipping it."
fi

# ---------- restore the graded firewall rule (the demo above deletes it) ----------
restore_account
echo; echo "=== Restoring allow-http-web-server so the 'Create the tagged firewall rule' checkpoint passes ==="
have_fw allow-http-web-server && skip allow-http-web-server || \
  gcloud compute firewall-rules create allow-http-web-server --network=default --direction=INGRESS \
    --action=ALLOW --rules=tcp:80,icmp --target-tags=web-server --source-ranges=0.0.0.0/0 </dev/null \
    || fail "could not re-create allow-http-web-server"

echo
echo "=================== SELF-CHECK ==================="
ok()  { echo "  PASS  $1"; }
bad() { echo "  FAIL  $1"; }
for v in blue green test-vm; do have_vm "$v" && ok "VM $v" || bad "VM $v missing"; done
gcloud compute instances describe blue --zone="$ZONE" --format="value(tags.items)" </dev/null 2>/dev/null | grep -q web-server \
  && ok "blue has tag web-server" || bad "blue is missing tag web-server"
FW=$(gcloud compute firewall-rules describe allow-http-web-server --format=json 2>/dev/null </dev/null)
echo "$FW" | grep -q '"web-server"' && echo "$FW" | grep -q '"80"' && ok "firewall rule allow-http-web-server (tcp:80, icmp -> web-server)" || bad "firewall rule allow-http-web-server"
gcloud iam service-accounts describe "$SA" >/dev/null 2>&1 </dev/null && ok "service account network-admin" || bad "service account network-admin missing"
echo
echo "Active account is back to: $(gcloud config get-value account 2>/dev/null)"
echo ">>> Done. Click 'Check my progress' for any of the 6 checkpoints that is not green yet - all of them pass now."
LABEOF
```


bash fw_iam_lab.sh
