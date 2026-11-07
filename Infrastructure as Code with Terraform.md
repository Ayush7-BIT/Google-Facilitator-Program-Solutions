```/text
cat > iac_lab.sh <<'LABEOF'
#!/bin/bash
# Infrastructure as Code with Terraform (GSP750) - universal solution
# Run in Cloud Shell:  bash iac_lab.sh
# Safe to re-run: it remembers your answers and resumes from the last finished step.

SCRIPT_PATH=$(readlink -f "$0")

PROJECT_ID=$(gcloud config get-value project 2>/dev/null)
if [ -z "$PROJECT_ID" ] || [ "$PROJECT_ID" = "(unset)" ]; then
  echo "ERROR: no project set. Run: gcloud config set project <Project ID from the lab panel>"
  exit 1
fi
export GOOGLE_CLOUD_PROJECT="$PROJECT_ID"
echo "Project: $PROJECT_ID"

WORKDIR="$HOME/tf-iac-lab"
mkdir -p "$WORKDIR"
VARS_FILE="$WORKDIR/.lab_vars2"   # (older .lab_vars answers are ignored)
PROGRESS_FILE="$WORKDIR/.lab_done"
[ -f "$VARS_FILE" ] && . "$VARS_FILE"
DONE=$(cat "$PROGRESS_FILE" 2>/dev/null)
DONE=${DONE:-0}

ask() {   # ask "prompt" "default" VARNAME
  local ans
  read -r -p "$1 [$2]: " ans
  ans=$(echo "${ans:-$2}" | xargs)
  printf -v "$3" '%s' "$ans"
}
fail() { echo; echo "FAIL: $1"; echo "Fix the error above, then re-run: bash $SCRIPT_PATH  (it resumes where it stopped)"; exit 1; }
mark() { echo "$1" > "$PROGRESS_FILE"; DONE=$1; }
pause() { read -r -p ">>> Lab page: click 'Check my progress' for: $1 -- wait for green, then press ENTER here " _; }

# ---------- install Terraform (lab prerequisite) ----------
# Cloud Shell ships a placeholder "terraform" that only prints install instructions,
# so we look for a binary that really prints "Terraform vX.Y.Z".
find_tf() {
  TF=""
  for c in /usr/bin/terraform "$HOME/bin/terraform" $(type -ap terraform 2>/dev/null); do
    if [ -x "$c" ] && "$c" version 2>/dev/null | head -n 1 | grep -q '^Terraform v'; then TF="$c"; return 0; fi
  done
  return 1
}
if ! find_tf; then
  echo
  echo "=== Installing Terraform (lab prerequisite, takes about 1 minute) ==="
  cat <<'EOF' > ~/.customize_environment
# Set up HashiCorp repository and install Terraform
wget -q -O - https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor --yes -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(grep -oP '(?<=UBUNTU_CODENAME=).*' /etc/os-release || lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install -y terraform
EOF
  bash ~/.customize_environment
  hash -r
fi
if ! find_tf; then
  echo "apt install did not give a working Terraform - downloading the binary into ~/bin instead"
  mkdir -p "$HOME/bin"
  curl -fsSL -o /tmp/terraform.zip https://releases.hashicorp.com/terraform/1.9.8/terraform_1.9.8_linux_amd64.zip \
    && unzip -o -q /tmp/terraform.zip -d "$HOME/bin" && chmod +x "$HOME/bin/terraform"
fi
find_tf || fail "Terraform could not be installed - run the install commands from the lab's Prerequisites section, then re-run this script"
# The lab pins Google provider 3.5.0, which cannot read Cloud Shell's built-in credentials
# ("invalid token JSON from metadata"). Give it a fresh access token on every call instead.
terraform() { GOOGLE_OAUTH_ACCESS_TOKEN="$(gcloud auth print-access-token 2>/dev/null </dev/null)" "$TF" "$@"; }
echo "Using $TF: $(terraform version | head -n 1)"

# ---------- detect the zone (region is taken from it) ----------
meta() { gcloud compute project-info describe --quiet --format="value(commonInstanceMetadata.items[$1])" 2>/dev/null </dev/null; }
DET_ZONE=$(meta google-compute-default-zone)
if [ -z "$DET_ZONE" ]; then DET_ZONE=$(gcloud config get-value compute/zone 2>/dev/null); fi
[ "$DET_ZONE" = "(unset)" ] && DET_ZONE=""
IMAGE="debian-cloud/debian-12"

echo
echo "=== The only yellow value in this lab is the zone/region in Task 1 (the provider block) ==="
while :; do
  read -r -p "Zone -> Task 1, provider block 'zone = ...' (e.g. us-east1-b) [${ZONE:-$DET_ZONE}]: " ans
  ZONE=$(echo "${ans:-${ZONE:-$DET_ZONE}}" | xargs)
  if echo "$ZONE" | grep -Eq '^[a-z]+-[a-z]+[0-9]+-[a-z]$'; then break; fi
  echo "   Please type the zone exactly as shown in yellow in Task 1 (format like us-east1-b)."
  ZONE=""
done
REGION="${ZONE%-*}"
echo "   Region: $REGION   (taken from the zone)"
{ for v in REGION ZONE; do printf '%s=%q\n' "$v" "${!v}"; done; } > "$VARS_FILE"
echo "Bucket name (Task 3): $PROJECT_ID  (the lab uses your Project ID)"

gcloud config set compute/region "$REGION" >/dev/null 2>&1
gcloud config set compute/zone "$ZONE" >/dev/null 2>&1
cd "$WORKDIR" || exit 1

# ---------- what main.tf contains at each stage ----------
INST=0; TAGS=0; COS=0; STATIC=0; BUCKET=0; PROV=0
[ "$DONE" -ge 2 ] && { INST=1; TAGS=1; }
[ "$DONE" -ge 3 ] && COS=1
[ "$DONE" -ge 5 ] && STATIC=2
[ "$DONE" -eq 6 ] && BUCKET=1
[ "$DONE" -ge 8 ] && PROV=1

write_main() {
  cat > main.tf <<EOF
terraform {
  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "3.5.0"
    }
  }
}

provider "google" {
  project = "$PROJECT_ID"
  region  = "$REGION"
  zone    = "$ZONE"
}

resource "google_compute_network" "vpc_network" {
  name = "terraform-network"
}
EOF
  if [ "$INST" = 1 ]; then
    local img="$IMAGE"; [ "$COS" = 1 ] && img="cos-cloud/cos-stable"
    {
      echo
      echo 'resource "google_compute_instance" "vm_instance" {'
      echo '  name         = "terraform-instance"'
      echo '  machine_type = "e2-micro"'
      [ "$TAGS" = 1 ] && echo '  tags         = ["web", "dev"]'
      if [ "$PROV" = 1 ]; then cat <<'EOF'

  provisioner "local-exec" {
    command = "echo ${self.name}: ${self.network_interface[0].access_config[0].nat_ip} >> ip_address.txt"
  }
EOF
      fi
      cat <<EOF

  boot_disk {
    initialize_params {
      image = "$img"
    }
  }

EOF
      if [ "$STATIC" = 2 ]; then cat <<'EOF'
  network_interface {
    network = google_compute_network.vpc_network.self_link
    access_config {
      nat_ip = google_compute_address.vm_static_ip.address
    }
  }
}
EOF
      else cat <<'EOF'
  network_interface {
    network = google_compute_network.vpc_network.name
    access_config {
    }
  }
}
EOF
      fi
    } >> main.tf
  fi
  if [ "$STATIC" -ge 1 ]; then cat >> main.tf <<'EOF'

resource "google_compute_address" "vm_static_ip" {
  name = "terraform-static-ip"
}
EOF
  fi
  if [ "$BUCKET" = 1 ]; then cat >> main.tf <<EOF

# New resource for the storage bucket our application will use.
resource "google_storage_bucket" "example_bucket" {
  name     = "$PROJECT_ID"
  location = "US"

  website {
    main_page_suffix = "index.html"
    not_found_page   = "404.html"
  }
}

# Create a new instance that uses the bucket
resource "google_compute_instance" "another_instance" {
  # Tells Terraform that this VM instance must be created only after the
  # storage bucket has been created.
  depends_on = [google_storage_bucket.example_bucket]

  name         = "terraform-instance-2"
  machine_type = "e2-micro"

  boot_disk {
    initialize_params {
      image = "cos-cloud/cos-stable"
    }
  }

  network_interface {
    network = google_compute_network.vpc_network.self_link
    access_config {
    }
  }
}
EOF
  fi
}

tf_apply() { terraform apply -auto-approve -input=false || fail "terraform apply failed"; }

write_main
[ "$DONE" -gt 0 ] && echo && echo ">>> Resuming: steps 1-$DONE are already done."
terraform init -input=false || fail "terraform init failed"

# ---------- TASK 1: build infrastructure ----------
if [ "$DONE" -lt 1 ]; then
  echo; echo "=== TASK 1: create terraform-network ==="
  tf_apply
  terraform show -no-color | head -n 20
  mark 1
  pause "Create resources in Terraform (30)"
fi

# ---------- TASK 2: add the instance, then add tags ----------
if [ "$DONE" -lt 2 ]; then
  echo; echo "=== TASK 2a: add terraform-instance ($IMAGE) ==="
  INST=1; write_main
  tf_apply
  echo; echo "=== TASK 2b: add tags web, dev (in-place update) ==="
  TAGS=1; write_main
  tf_apply
  mark 2
  pause "Change the infrastructure (20)"
fi

# ---------- TASK 2: destructive change ----------
if [ "$DONE" -lt 3 ]; then
  echo; echo "=== TASK 2c: change boot image to cos-cloud/cos-stable (instance is recreated) ==="
  COS=1; write_main
  tf_apply
  terraform show -no-color | grep -m 3 -E 'image|name '
  mark 3
  pause "Make destructive changes (20)"
fi

# ---------- TASK 2: destroy ----------
if [ "$DONE" -lt 4 ]; then
  echo; echo "=== TASK 2d: terraform destroy ==="
  terraform destroy -auto-approve -input=false || fail "terraform destroy failed"
  mark 4
fi

# ---------- TASK 3: recreate, then static IP ----------
if [ "$DONE" -lt 5 ]; then
  echo; echo "=== TASK 3a: recreate the network and instance ==="
  tf_apply
  echo; echo "=== TASK 3b: add terraform-static-ip and plan ==="
  STATIC=1; write_main
  terraform plan -input=false || fail "terraform plan failed"
  echo; echo "=== TASK 3c: attach the static IP to the instance (saved plan 'static_ip') ==="
  STATIC=2; write_main
  terraform plan -input=false -out static_ip || fail "terraform plan failed"
  terraform apply -input=false "static_ip" || fail "terraform apply static_ip failed"
  mark 5
  pause "Create resource dependencies (20)"
fi

# ---------- TASK 3: bucket + dependent instance ----------
if [ "$DONE" -lt 6 ]; then
  echo; echo "=== TASK 3d: add bucket $PROJECT_ID and terraform-instance-2 (depends_on the bucket) ==="
  BUCKET=1; write_main
  terraform plan -input=false || fail "terraform plan failed"
  tf_apply
  mark 6
  pause "Create a bucket dependent instance (10)"
fi

if [ "$DONE" -lt 7 ]; then
  echo; echo "=== TASK 3e: remove the bucket and terraform-instance-2 ==="
  BUCKET=0; write_main
  tf_apply
  mark 7
fi

# ---------- TASK 4: provisioner (no checkpoint) ----------
if [ "$DONE" -lt 8 ]; then
  echo; echo "=== TASK 4: add local-exec provisioner, taint and re-apply ==="
  PROV=1; write_main
  tf_apply
  terraform taint google_compute_instance.vm_instance || fail "terraform taint failed"
  tf_apply
  echo "ip_address.txt:"; cat ip_address.txt 2>/dev/null
  mark 8
fi

# ---------- SELF-CHECK ----------
echo
echo "=================== SELF-CHECK ==================="
ok()  { echo "  PASS  $1"; }
bad() { echo "  FAIL  $1"; }
gcloud compute networks describe terraform-network >/dev/null 2>&1 </dev/null && ok "terraform-network exists" || bad "terraform-network missing"
INFO=$(gcloud compute instances describe terraform-instance --zone="$ZONE" --format=json 2>/dev/null </dev/null)
[ -n "$INFO" ] && ok "terraform-instance exists" || bad "terraform-instance missing"
echo "$INFO" | grep -q '"web"' && echo "$INFO" | grep -q '"dev"' && ok "tags web, dev" || bad "tags web, dev missing"
echo "$INFO" | grep -q 'cos-cloud' && ok "boot image is cos-stable" || bad "boot image is not cos-stable"
SIP=$(gcloud compute addresses describe terraform-static-ip --region="$REGION" --format="value(address)" 2>/dev/null </dev/null)
[ -n "$SIP" ] && echo "$INFO" | grep -q "$SIP" && ok "terraform-static-ip ($SIP) is attached" || bad "static IP not attached"
echo
echo ">>> Done. All 5 checkpoints should be green on the lab page."
LABEOF
```


bash iac_lab.sh
