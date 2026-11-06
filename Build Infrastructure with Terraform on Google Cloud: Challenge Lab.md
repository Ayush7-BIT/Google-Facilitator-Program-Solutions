```/text
cat > terraform_lab.sh <<'LABEOF'
#!/bin/bash
# Build Infrastructure with Terraform on Google Cloud: Challenge Lab (GSP345) - universal solution
# Run in Cloud Shell:  bash terraform_lab.sh
# Safe to re-run: it remembers your answers and resumes from the last finished task.
# Every question shows where the value is in the lab text (the yellow values).

SCRIPT_PATH=$(readlink -f "$0")

PROJECT_ID=$(gcloud config get-value project 2>/dev/null)
if [ -z "$PROJECT_ID" ] || [ "$PROJECT_ID" = "(unset)" ]; then
  echo "ERROR: no project set. Run: gcloud config set project <Project ID from the lab panel>"
  exit 1
fi
export GOOGLE_CLOUD_PROJECT="$PROJECT_ID"
echo "Project: $PROJECT_ID"

WORKDIR="$HOME/tf-challenge"
mkdir -p "$WORKDIR/modules/instances" "$WORKDIR/modules/storage"
cd "$WORKDIR" || exit 1
VARS_FILE="$WORKDIR/.lab_vars"
PROGRESS_FILE="$WORKDIR/.lab_done"   # (old .lab_progress from the first version is ignored)
[ -f "$VARS_FILE" ] && . "$VARS_FILE"
DONE=$(cat "$PROGRESS_FILE" 2>/dev/null)
DONE=${DONE:-0}

ask() {   # ask "prompt" "default" VARNAME
  local ans
  read -r -p "$1 [$2]: " ans
  ans=$(echo "${ans:-$2}" | xargs)
  printf -v "$3" '%s' "$ans"
}
ask_req() {   # like ask, but an empty answer is not accepted
  local ans=""
  while [ -z "$ans" ]; do
    read -r -p "$1 [$2]: " ans
    ans=$(echo "${ans:-$2}" | xargs)
    [ -z "$ans" ] && echo "   This value is required - copy it from the lab text."
  done
  printf -v "$3" '%s' "$ans"
}
fail() { echo; echo "FAIL: $1"; echo "Fix the error above, then re-run: bash $SCRIPT_PATH  (it resumes where it stopped)"; exit 1; }
mark() { echo "$1" > "$PROGRESS_FILE"; DONE=$1; }

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
terraform() { "$TF" "$@"; }   # every terraform call below uses the real binary
echo "Using $TF: $(terraform version | head -n 1)"

# ---------- detect values from the pre-created instances ----------
DET_ZONE=$(gcloud compute instances list --filter="name=tf-instance-1" --format="value(zone.basename())" 2>/dev/null </dev/null | head -n 1)
if [ -z "$DET_ZONE" ]; then DET_ZONE=$(gcloud config get-value compute/zone 2>/dev/null); fi
if [ "$DET_ZONE" = "(unset)" ]; then DET_ZONE=""; fi
DET_REGION=""
[ -n "$DET_ZONE" ] && DET_REGION="${DET_ZONE%-*}"
[ -z "$DET_ZONE" ] && echo "Could not find tf-instance-1 - type the region/zone from the lab text."

DET_IMAGE=""; DET_MT1=""; DET_MT2=""
if [ -n "$DET_ZONE" ]; then
  DISK=$(gcloud compute instances describe tf-instance-1 --zone="$DET_ZONE" --format="value(disks[0].source.basename())" 2>/dev/null </dev/null)
  if [ -n "$DISK" ]; then
    IMG=$(gcloud compute disks describe "$DISK" --zone="$DET_ZONE" --format="value(sourceImage)" 2>/dev/null </dev/null)
    [ -n "$IMG" ] && DET_IMAGE="projects/${IMG#*/projects/}"
  fi
  DET_MT1=$(gcloud compute instances describe tf-instance-1 --zone="$DET_ZONE" --format="value(machineType.basename())" 2>/dev/null </dev/null)
  DET_MT2=$(gcloud compute instances describe tf-instance-2 --zone="$DET_ZONE" --format="value(machineType.basename())" 2>/dev/null </dev/null)
fi
# original machine types only matter before Task 4 changes them
[ -z "${ORIG_MT1:-}" ] && ORIG_MT1="${DET_MT1:-e2-micro}"
[ -z "${ORIG_MT2:-}" ] && ORIG_MT2="${DET_MT2:-e2-micro}"

echo
echo "=== Copy each value from the lab text (yellow), or press ENTER to accept the default ==="
ask_req "Region      -> Task 1, step 2 (1st yellow value)              " "${REGION:-$DET_REGION}" REGION
ZDEF="${ZONE:-$DET_ZONE}"
case "$ZDEF" in "${REGION}"-*) ;; *) ZDEF="${REGION}-a" ;; esac
ask_req "Zone        -> Task 1, step 2 (2nd yellow value)              " "$ZDEF" ZONE
case "$ZONE" in "${REGION}"-*) ;; *) echo "WARNING: zone $ZONE is not in region $REGION - check the lab text" ;; esac
echo "   (Boot disk image: the default below is read from tf-instance-1 itself - press ENTER to keep it)"
ask_req "Boot image  -> Task 2, step 2 ('specifying ... for the boot disk image')" "${IMAGE:-$DET_IMAGE}" IMAGE
ask_req "Bucket name -> Task 3, step 1 ('For the bucket name, use ...')" "${BUCKET:-}" BUCKET
ask_req "3rd VM name -> Task 4, step 3 ('Add a third instance ... name it ...')" "${THIRD_VM:-}" THIRD_VM
ask_req "VPC name    -> Task 6, step 2 ('Name the VPC ...')         " "${VPC:-}" VPC
ask     "Pause after each task to click 'Check my progress'? (y/n)  " "${PAUSE:-y}" PAUSE
BUCKET=$(echo "$BUCKET" | tr 'A-Z' 'a-z')

{ for v in REGION ZONE IMAGE BUCKET THIRD_VM VPC PAUSE ORIG_MT1 ORIG_MT2; do printf '%s=%q\n' "$v" "${!v}"; done; } > "$VARS_FILE"

pause() {   # pause "checkpoint" [force]
  if [ "$PAUSE" = "y" ] || [ "${2:-}" = "force" ]; then
    read -r -p ">>> Lab page: click 'Check my progress' for: $1 -- then press ENTER here to continue " _
  fi
}

gcloud config set compute/region "$REGION" >/dev/null 2>&1
gcloud config set compute/zone "$ZONE" >/dev/null 2>&1

# ---------- what the configuration contains at each stage ----------
STORAGE_ON=0; BACKEND_ON=0; THIRD_ON=0; VPC_ON=0; NET_ON=0; FW_ON=0
MT1="$ORIG_MT1"; MT2="$ORIG_MT2"
[ "$DONE" -ge 3 ] && { STORAGE_ON=1; BACKEND_ON=1; }
[ "$DONE" -ge 4 ] && { MT1="e2-standard-2"; MT2="e2-standard-2"; }
[ "$DONE" -eq 4 ] && THIRD_ON=1
[ "$DONE" -ge 6 ] && { VPC_ON=1; NET_ON=1; }
[ "$DONE" -ge 7 ] && FW_ON=1

write_variables() {
  for f in variables.tf modules/instances/variables.tf modules/storage/variables.tf; do
    cat > "$f" <<EOF
variable "region" {
  description = "Region for resources"
  type        = string
  default     = "$REGION"
}

variable "zone" {
  description = "Zone for resources"
  type        = string
  default     = "$ZONE"
}

variable "project_id" {
  description = "Google Cloud Project ID"
  type        = string
  default     = "$PROJECT_ID"
}
EOF
  done
}

write_main() {
  {
    echo 'terraform {'
    if [ "$BACKEND_ON" = 1 ]; then cat <<EOF
  backend "gcs" {
    bucket = "$BUCKET"
    prefix = "terraform/state"
  }

EOF
    fi
    cat <<'EOF'
  required_providers {
    google = {
      source  = "hashicorp/google"
      version = "~> 6.0"
    }
  }
}

provider "google" {
  project = var.project_id
  region  = var.region
  zone    = var.zone
}

module "instances" {
  source = "./modules/instances"
}
EOF
    if [ "$STORAGE_ON" = 1 ]; then cat <<'EOF'

module "storage" {
  source = "./modules/storage"
}
EOF
    fi
    if [ "$VPC_ON" = 1 ]; then cat <<EOF

module "vpc" {
  source  = "terraform-google-modules/network/google"
  version = "10.0.0"

  project_id   = var.project_id
  network_name = "$VPC"
  routing_mode = "GLOBAL"

  subnets = [
    {
      subnet_name   = "subnet-01"
      subnet_ip     = "10.10.10.0/24"
      subnet_region = var.region
    },
    {
      subnet_name   = "subnet-02"
      subnet_ip     = "10.10.20.0/24"
      subnet_region = var.region
    }
  ]
}
EOF
    fi
    if [ "$FW_ON" = 1 ]; then cat <<'EOF'

resource "google_compute_firewall" "tf-firewall" {
  name    = "tf-firewall"
  network = module.vpc.network_self_link

  allow {
    protocol = "tcp"
    ports    = ["80"]
  }

  source_ranges = ["0.0.0.0/0"]
}
EOF
    fi
  } > main.tf
}

instance_block() {   # instance_block NAME MACHINE_TYPE SUBNET
  local net="default" subnet_line=""
  if [ "$NET_ON" = 1 ] && [ -n "$3" ]; then
    net="$VPC"
    subnet_line="    subnetwork = \"$3\""
  fi
  cat <<EOF
resource "google_compute_instance" "$1" {
  name         = "$1"
  machine_type = "$2"
  zone         = var.zone

  boot_disk {
    initialize_params {
      image = "$IMAGE"
    }
  }

  network_interface {
    network = "$net"
$subnet_line
  }

  metadata_startup_script = <<-EOT
        #!/bin/bash
    EOT
  allow_stopping_for_update = true
}

EOF
}

write_instances() {
  {
    instance_block tf-instance-1 "$MT1" subnet-01
    instance_block tf-instance-2 "$MT2" subnet-02
    if [ "$THIRD_ON" = 1 ]; then instance_block "$THIRD_VM" "e2-standard-2" ""; fi
  } > modules/instances/instances.tf
  cat > modules/instances/outputs.tf <<'EOF'
output "instance_1_id" {
  value = google_compute_instance.tf-instance-1.instance_id
}

output "instance_2_id" {
  value = google_compute_instance.tf-instance-2.instance_id
}
EOF
}

write_storage() {
  cat > modules/storage/storage.tf <<EOF
resource "google_storage_bucket" "storage-bucket" {
  name                        = "$BUCKET"
  location                    = "US"
  force_destroy               = true
  uniform_bucket_level_access = true
}
EOF
  cat > modules/storage/outputs.tf <<'EOF'
output "bucket_name" {
  value = google_storage_bucket.storage-bucket.name
}
EOF
}

tf_init()  { terraform init -input=false "$@" || fail "terraform init failed"; }
tf_apply() { terraform apply -auto-approve -input=false || fail "terraform apply failed"; }

write_variables; write_main; write_instances; write_storage
[ "$DONE" -gt 0 ] && echo && echo ">>> Resuming: tasks 1-$DONE are already done."

# ---------- TASK 1: configuration files + init ----------
if [ "$DONE" -lt 1 ]; then
  echo; echo "=== TASK 1: create the configuration files and initialize Terraform ==="
  tf_init
  mark 1
fi

# ---------- TASK 2: import the existing instances ----------
if [ "$DONE" -lt 2 ]; then
  echo; echo "=== TASK 2: import tf-instance-1 and tf-instance-2 ==="
  tf_init
  for n in tf-instance-1 tf-instance-2; do
    if terraform state list 2>/dev/null | grep -qx "module.instances.google_compute_instance.$n"; then
      echo "$n is already imported"
    else
      terraform import -input=false "module.instances.google_compute_instance.$n" \
        "projects/$PROJECT_ID/zones/$ZONE/instances/$n" || fail "could not import $n"
    fi
  done
  tf_apply
  mark 2
  pause "Import infrastructure (20)"
fi

# ---------- TASK 3: bucket + remote backend ----------
if [ "$DONE" -lt 3 ]; then
  echo; echo "=== TASK 3: create bucket $BUCKET and move the state into it ==="
  STORAGE_ON=1; write_main
  tf_init
  tf_apply
  BACKEND_ON=1; write_main
  terraform init -input=false -migrate-state -force-copy || fail "could not move the state to gs://$BUCKET"
  mark 3
  pause "Configure a remote backend (10)"
fi

# ---------- TASK 4: e2-standard-2 + third instance ----------
if [ "$DONE" -lt 4 ]; then
  echo; echo "=== TASK 4: switch to e2-standard-2 and add $THIRD_VM ==="
  MT1="e2-standard-2"; MT2="e2-standard-2"; THIRD_ON=1; write_instances
  tf_init
  tf_apply
  mark 4
  echo "NOTE: this checkpoint must be checked NOW - Task 5 deletes $THIRD_VM."
  pause "Modify and update infrastructure (20)" force
fi

# ---------- TASK 5: destroy the third instance ----------
if [ "$DONE" -lt 5 ]; then
  echo; echo "=== TASK 5: remove $THIRD_VM from the configuration ==="
  THIRD_ON=0; write_instances
  tf_init
  tf_apply
  mark 5
  pause "Destroy resources (10)"
fi

# ---------- TASK 6: network module from the Registry ----------
if [ "$DONE" -lt 6 ]; then
  echo; echo "=== TASK 6: create VPC $VPC with subnet-01 and subnet-02 ==="
  VPC_ON=1; write_main
  tf_init
  tf_apply
  echo "--- connecting tf-instance-1 to subnet-01 and tf-instance-2 to subnet-02"
  NET_ON=1; write_instances
  tf_apply
  mark 6
  pause "Use a Module from the Registry (20)"
fi

# ---------- TASK 7: firewall ----------
if [ "$DONE" -lt 7 ]; then
  echo; echo "=== TASK 7: create tf-firewall (tcp:80 from 0.0.0.0/0) ==="
  FW_ON=1; write_main
  tf_init
  tf_apply
  mark 7
  pause "Configure a firewall (20)"
fi

# ---------- SELF-CHECK ----------
echo
echo "=================== SELF-CHECK ==================="
ok()  { echo "  PASS  $1"; }
bad() { echo "  FAIL  $1"; }
STATE=$(terraform state list 2>/dev/null)
for n in tf-instance-1 tf-instance-2; do
  echo "$STATE" | grep -qx "module.instances.google_compute_instance.$n" \
    && ok "Task 2: $n is in the Terraform state" || bad "Task 2: $n is not in the Terraform state"
done
gcloud storage ls "gs://$BUCKET/terraform/state/default.tfstate" >/dev/null 2>&1 </dev/null \
  && ok "Task 3: state is stored at gs://$BUCKET/terraform/state" || bad "Task 3: no state file in gs://$BUCKET/terraform/state"
for n in tf-instance-1 tf-instance-2; do
  mt=$(gcloud compute instances describe "$n" --zone="$ZONE" --format="value(machineType.basename())" 2>/dev/null </dev/null)
  [ "$mt" = "e2-standard-2" ] && ok "Task 4: $n is e2-standard-2" || bad "Task 4: $n is '$mt', not e2-standard-2"
done
gcloud compute instances describe "$THIRD_VM" --zone="$ZONE" >/dev/null 2>&1 </dev/null \
  && bad "Task 5: $THIRD_VM still exists" || ok "Task 5: $THIRD_VM is destroyed"
mode=$(gcloud compute networks describe "$VPC" --format="value(routingConfig.routingMode)" 2>/dev/null </dev/null)
[ "$mode" = "GLOBAL" ] && ok "Task 6: VPC $VPC exists with GLOBAL routing" || bad "Task 6: VPC $VPC missing or not GLOBAL"
for s in subnet-01:10.10.10.0/24 subnet-02:10.10.20.0/24; do
  r=$(gcloud compute networks subnets describe "${s%%:*}" --region="$REGION" --format="value(ipCidrRange)" 2>/dev/null </dev/null)
  [ "$r" = "${s#*:}" ] && ok "Task 6: ${s%%:*} is ${s#*:}" || bad "Task 6: ${s%%:*} missing or wrong range ('$r')"
done
for p in tf-instance-1:subnet-01 tf-instance-2:subnet-02; do
  sn=$(gcloud compute instances describe "${p%%:*}" --zone="$ZONE" --format="value(networkInterfaces[0].subnetwork.basename())" 2>/dev/null </dev/null)
  [ "$sn" = "${p#*:}" ] && ok "Task 6: ${p%%:*} is on ${p#*:}" || bad "Task 6: ${p%%:*} is on '$sn', not ${p#*:}"
done
FW=$(gcloud compute firewall-rules describe tf-firewall --format=json 2>/dev/null </dev/null)
fwnet=$(gcloud compute firewall-rules describe tf-firewall --format="value(network.basename())" 2>/dev/null </dev/null)
if [ -n "$FW" ] && [ "$fwnet" = "$VPC" ] && echo "$FW" | grep -q '"80"' && echo "$FW" | grep -q '0.0.0.0/0'; then
  ok "Task 7: tf-firewall allows tcp:80 from 0.0.0.0/0 on $VPC"
else
  bad "Task 7: tf-firewall missing or not configured as required"
fi
echo
echo ">>> Done. Click 'Check my progress' for any checkpoint that is not green yet."
echo "    (Task 4's checkpoint can only pass while $THIRD_VM exists - if you missed it, it cannot be re-checked now.)"
LABEOF
```



bash terraform_lab.sh
