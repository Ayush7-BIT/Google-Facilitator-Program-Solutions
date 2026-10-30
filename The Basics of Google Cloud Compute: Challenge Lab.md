```text
cat > lab6.sh <<'LABEOF'
#!/bin/bash
# Universal solver: The Basics of Google Cloud Compute - Challenge Lab
# Run in Cloud Shell with:  bash lab6.sh

ask() { local r; read -e -r -p "$2 [$3]: " r; printf -v "$1" '%s' "${r:-$3}"; }
die() { echo; echo "STOP: $1"; exit 1; }
retry() { local i; for i in 1 2 3 4 5 6; do "$@" && return 0; echo "   retrying in 10s..."; sleep 10; done; return 1; }

# Fixed by the lab text (not yellow)
VM_NAME="my-instance"
DISK_NAME="mydisk"
MACHINE="e2-medium"
BOOT_SIZE="10GB"
DISK_SIZE="200GB"

PROJECT_ID=$(gcloud config get-value project 2>/dev/null)
[ -z "$PROJECT_ID" ] && die "no project set. Run: gcloud config set project <Project ID from the lab panel>"
echo "Project: $PROJECT_ID"

echo
echo "=== Enter the YELLOW values from the lab (press Enter to keep the default) ==="
ask ZONE   "Zone (challenge scenario: '... region and ... zone')" "europe-west4-c"
ask BUCKET "Bucket name (Task 1)" "${PROJECT_ID}-bucket"
REGION="${ZONE%-*}"
gcloud config set compute/zone "$ZONE" >/dev/null 2>&1
gcloud config set compute/region "$REGION" >/dev/null 2>&1
gcloud services enable compute.googleapis.com

echo
echo "=== Task 1: bucket gs://$BUCKET (US multi-region) ==="
if gcloud storage buckets describe "gs://$BUCKET" >/dev/null 2>&1; then
  echo "bucket already exists"
else
  gcloud storage buckets create "gs://$BUCKET" --location=US || die "could not create the bucket. Check the name matches the lab exactly."
fi

echo
echo "=== Task 2: VM $VM_NAME and disk $DISK_NAME ==="
# 'Allow HTTP traffic' in the console = network tag http-server + a firewall rule for port 80
gcloud compute firewall-rules describe default-allow-http >/dev/null 2>&1 || \
  gcloud compute firewall-rules create default-allow-http --direction=INGRESS --priority=1000 \
    --network=default --action=ALLOW --rules=tcp:80 --source-ranges=0.0.0.0/0 --target-tags=http-server \
  || echo "(warning: could not create the HTTP firewall rule - Task 3 web check may fail)"

gcloud compute instances describe "$VM_NAME" --zone="$ZONE" >/dev/null 2>&1 || \
  gcloud compute instances create "$VM_NAME" --zone="$ZONE" --machine-type="$MACHINE" \
    --image-family=debian-12 --image-project=debian-cloud \
    --boot-disk-size="$BOOT_SIZE" --boot-disk-type=pd-balanced --tags=http-server \
  || die "could not create $VM_NAME"

gcloud compute disks describe "$DISK_NAME" --zone="$ZONE" >/dev/null 2>&1 || \
  gcloud compute disks create "$DISK_NAME" --zone="$ZONE" --size="$DISK_SIZE" \
  || die "could not create disk $DISK_NAME"

if gcloud compute instances describe "$VM_NAME" --zone="$ZONE" --format="value(disks[].source)" | grep -q "/$DISK_NAME"; then
  echo "disk already attached"
else
  gcloud compute instances attach-disk "$VM_NAME" --disk="$DISK_NAME" --zone="$ZONE" || die "could not attach $DISK_NAME"
fi

echo
echo "=== Task 3: NGINX on $VM_NAME ==="
retry gcloud compute ssh "$VM_NAME" --zone="$ZONE" --quiet --command="echo ssh-ok" || die "could not SSH into $VM_NAME"
gcloud compute ssh "$VM_NAME" --zone="$ZONE" --quiet --command="
sudo apt-get update -y -qq &&
sudo DEBIAN_FRONTEND=noninteractive apt-get install -y -qq nginx &&
sudo systemctl enable --now nginx &&
echo nginx status: \$(sudo systemctl is-active nginx)
" || die "NGINX install failed"

IP=$(gcloud compute instances describe "$VM_NAME" --zone="$ZONE" --format="value(networkInterfaces[0].accessConfigs[0].natIP)")
echo "External IP: $IP"
for i in 1 2 3 4 5 6 7 8; do
  curl -s -m 5 "http://$IP/" | grep -qi "welcome to nginx" && break
  echo "   waiting for the web page..."; sleep 8
done

echo
echo "=== Self-check ==="
check() { if eval "$1"; then echo "  PASS  $2"; else echo "  FAIL  $2"; fi; }
check '[ "$(gcloud storage buckets describe gs://$BUCKET --format="value(location)" 2>/dev/null)" = "US" ]' "Task 1: bucket gs://$BUCKET is in US"
check 'gcloud compute instances describe "$VM_NAME" --zone="$ZONE" --format="value(machineType)" | grep -q "$MACHINE"' "Task 2: $VM_NAME is $MACHINE"
check 'gcloud compute instances describe "$VM_NAME" --zone="$ZONE" --format="value(tags.items)" | grep -q http-server' "Task 2: HTTP traffic allowed (tag http-server)"
check 'gcloud compute instances describe "$VM_NAME" --zone="$ZONE" --format="value(disks[].source)" | grep -q "/$DISK_NAME"' "Task 2: $DISK_NAME is attached"
check 'curl -s -m 8 "http://$IP/" | grep -qi "welcome to nginx"' "Task 3: NGINX answers on http://$IP/"
echo
echo ">>> Done. On the lab page, click 'Check my progress' for Tasks 1, 2 and 3."
LABEOF
```

bash lab6.sh
