```/text
cat > terraform_lab.sh <<'LABEOF'
#!/bin/bash
# Terraform Fundamentals (GSP156) - universal solution (v2: handles Cloud Shell's placeholder 'terraform' command)

die() { echo; echo "STOP: $1"; exit 1; }
ask() { local r; read -e -r -p "$1 [$2]: " r; printf -v "$3" '%s' "${r:-$2}"; }

PROJECT_ID=$(gcloud config get-value project 2>/dev/null)
if [ -z "$PROJECT_ID" ] || [ "$PROJECT_ID" = "(unset)" ]; then
  die "no project set. Run: gcloud config set project <Project ID from the lab panel>"
fi
export GOOGLE_CLOUD_PROJECT="$PROJECT_ID"
echo "Project: $PROJECT_ID"

# ---------- detect the zone ----------
meta() { gcloud compute project-info describe --quiet --format="value(commonInstanceMetadata.items[$1])" 2>/dev/null </dev/null; }
DET_ZONE=$(gcloud config get-value compute/zone 2>/dev/null)
if [ -z "$DET_ZONE" ] || [ "$DET_ZONE" = "(unset)" ]; then DET_ZONE=$(meta google-compute-default-zone); fi
if [ -z "$DET_ZONE" ]; then
  echo "Could not detect the zone - type the one from the lab text."
  DET_ZONE="us-central1-a"
fi

echo
echo "=== Press ENTER to accept each default (check them against the lab text, Task 2 step 3) ==="
ask "Zone (the 'zone =' line)                          " "$DET_ZONE" ZONE
ask "Debian image (the 'image =' line)                 " "debian-cloud/debian-12" IMAGE
ask "VM name (name = ...)                              " "terraform" VM
ask "Machine type                                      " "e2-medium" MACHINE
ask "Pause so you can click 'Check my progress'? (y/n) " "y" PAUSE

pause() {
  if [ "$PAUSE" = "y" ]; then
    read -r -p ">>> Lab page: click 'Check my progress' for: $1 -- then press ENTER here to continue " _
  fi
}

# The Compute Engine API is normally already enabled in this lab; students usually cannot enable APIs.
if ! gcloud services list --enabled --filter="config.name=compute.googleapis.com" --format="value(config.name)" 2>/dev/null | grep -q .; then
  gcloud services enable compute.googleapis.com --quiet >/dev/null 2>&1 \
    || echo "(note: could not enable the Compute Engine API - it is normally already on in this lab; continuing)"
fi

# ---------- make sure the image exists ----------
img_ok() {   # img_ok <project/family-or-image>
  local p="${1%%/*}" f="${1#*/}"
  gcloud compute images describe-from-family "$f" --project="$p" >/dev/null 2>&1 \
    || gcloud compute images describe "$f" --project="$p" >/dev/null 2>&1
}
if echo "$IMAGE" | grep -q "/" && ! img_ok "$IMAGE"; then
  echo "Image '$IMAGE' not found - looking for a Debian image that exists..."
  for c in debian-cloud/debian-12 debian-cloud/debian-11 debian-cloud/debian-13; do
    if img_ok "$c"; then IMAGE="$c"; echo "Using $IMAGE"; break; fi
  done
fi

# ---------- TASK 1: install Terraform ----------
echo
echo "=== TASK 1: install Terraform ==="
find_tf() {   # prints the path of a REAL terraform (Cloud Shell has a placeholder that only prints instructions)
  local c
  for c in $(type -ap terraform 2>/dev/null) "$HOME/bin/terraform" /usr/bin/terraform /usr/local/bin/terraform; do
    [ -x "$c" ] || continue
    if "$c" version 2>/dev/null | head -n1 | grep -q '^Terraform v'; then echo "$c"; return 0; fi
  done
  return 1
}
TF=$(find_tf)
if [ -n "$TF" ]; then
  echo "Terraform is installed: $TF"
else
  echo "Terraform is not installed yet (Cloud Shell only has a placeholder command). Installing it the way the lab does..."
  cat <<'EOF' > ~/.customize_environment
# Set up HashiCorp repository and install Terraform
wget -O - https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor --yes -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(grep -oP '(?<=UBUNTU_CODENAME=).*' /etc/os-release || lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install -y terraform
EOF
  for i in 1 2 3; do
    bash ~/.customize_environment
    TF=$(find_tf) && break
    echo "   install did not finish (try $i/3) - waiting 15s"; sleep 15
  done
  if [ -z "$TF" ]; then
    echo "apt install did not work - downloading the Terraform binary into ~/bin instead"
    mkdir -p "$HOME/bin"
    if curl -fsSL -o /tmp/terraform.zip "https://releases.hashicorp.com/terraform/1.9.8/terraform_1.9.8_linux_amd64.zip"; then
      (cd "$HOME/bin" && { unzip -o /tmp/terraform.zip >/dev/null 2>&1 || python3 -m zipfile -e /tmp/terraform.zip "$HOME/bin"; })
      chmod +x "$HOME/bin/terraform" 2>/dev/null
    fi
    TF=$(find_tf)
  fi
fi
[ -n "$TF" ] || die "Terraform could not be installed. Send me the messages above."
"$TF" version

# ---------- TASK 2: configuration, init, plan, apply ----------
echo
echo "=== TASK 2: build the infrastructure ==="
WORKDIR="$HOME/terraform-fundamentals"
mkdir -p "$WORKDIR" && cd "$WORKDIR" || die "could not create $WORKDIR"

cat > instance.tf <<EOF
resource "google_compute_instance" "default" {
  project      = "${PROJECT_ID}"
  zone         = "${ZONE}"
  name         = "${VM}"
  machine_type = "${MACHINE}"
  boot_disk {
    initialize_params {
      image = "${IMAGE}"
    }
  }
  network_interface {
    network = "default"
    access_config {
    }
  }
}
EOF
echo "--- instance.tf ---"; cat instance.tf
echo "--- files in $WORKDIR ---"; ls

"$TF" init -input=false || die "terraform init failed"
[ -d .terraform ] || die "terraform init did not download the Google provider (no .terraform folder)"
"$TF" plan -input=false || echo "WARNING: terraform plan reported a problem"

# If a VM with this name already exists but Terraform does not know it, import it
if gcloud compute instances describe "$VM" --zone="$ZONE" >/dev/null 2>&1 \
   && ! "$TF" state list 2>/dev/null | grep -q "google_compute_instance.default"; then
  echo "VM $VM already exists - importing it into the Terraform state"
  "$TF" import -input=false google_compute_instance.default "projects/${PROJECT_ID}/zones/${ZONE}/instances/${VM}"
fi

applied=0
for i in 1 2 3; do
  if "$TF" apply -auto-approve -input=false; then applied=1; break; fi
  echo "   apply failed (try $i/3) - waiting 20s"; sleep 20
done
[ "$applied" = 1 ] || die "terraform apply did not finish - send me the error above"
gcloud compute instances describe "$VM" --zone="$ZONE" >/dev/null 2>&1 \
  || die "terraform apply finished but the VM $VM does not exist in $ZONE - send me the output above"

echo
echo "--- terraform show ---"
"$TF" show | head -n 40
pause "Create a VM instance in the zone with Terraform (100)"

# ---------- SELF-CHECK ----------
echo
echo "=================== SELF-CHECK ==================="
ok()  { echo "  PASS  $1"; }
bad() { echo "  FAIL  $1"; }
INFO=$(gcloud compute instances describe "$VM" --zone="$ZONE" --format=json 2>/dev/null)
echo "$INFO" | grep -q '"status": "RUNNING"' && ok "VM $VM is RUNNING in $ZONE" || bad "VM $VM not running in $ZONE"
echo "$INFO" | grep -q "machineTypes/${MACHINE}\"" && ok "machine type is $MACHINE" || bad "machine type is not $MACHINE"
"$TF" state list 2>/dev/null | grep -q "google_compute_instance.default" \
  && ok "Terraform manages google_compute_instance.default" || bad "resource missing from the Terraform state"
[ -f terraform.tfstate ] && ok "terraform.tfstate exists in $WORKDIR" || bad "no terraform.tfstate"
echo
echo "You can also see the VM in Compute Engine > VM instances."
echo ">>> Done. Click 'Check my progress' on the lab page if it is not green yet."
LABEOF
```



bash terraform_lab.sh
