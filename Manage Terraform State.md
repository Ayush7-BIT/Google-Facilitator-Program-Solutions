```/text
cat > state_lab.sh <<'LABEOF'
#!/bin/bash
# Manage Terraform State (GSP752) - universal solution
# Run in Cloud Shell:  bash state_lab.sh
# Safe to re-run: it remembers your answers and resumes from the last finished part.

SCRIPT_PATH=$(readlink -f "$0")

PROJECT_ID=$(gcloud config get-value project 2>/dev/null)
if [ -z "$PROJECT_ID" ] || [ "$PROJECT_ID" = "(unset)" ]; then
  echo "ERROR: no project set. Run: gcloud config set project <Project ID from the lab panel>"
  exit 1
fi
export GOOGLE_CLOUD_PROJECT="$PROJECT_ID"
echo "Project: $PROJECT_ID"

WORKDIR="$HOME/tf-state-lab"
mkdir -p "$WORKDIR"
VARS_FILE="$WORKDIR/.lab_vars"
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

# ---------- detect the region ----------
meta() { gcloud compute project-info describe --quiet --format="value(commonInstanceMetadata.items[$1])" 2>/dev/null </dev/null; }
DET_REGION=$(gcloud config get-value compute/region 2>/dev/null)
if [ -z "$DET_REGION" ] || [ "$DET_REGION" = "(unset)" ]; then DET_REGION=$(meta google-compute-default-region); fi
if [ -z "$DET_REGION" ]; then DET_REGION="us-central1"; fi

echo
echo "=== Copy each value from the lab text (yellow), or press ENTER to accept the default ==="
ask "Region      -> Task 1, step 4 (provider block, 'region = ...')      " "${REGION:-$DET_REGION}" REGION
ask "Do Task 2 too (Docker import - not graded, optional)? (y/n)          " "${DO_TASK2:-y}" DO_TASK2
{ for v in REGION DO_TASK2; do printf '%s=%q\n' "$v" "${!v}"; done; } > "$VARS_FILE"
BUCKET="$PROJECT_ID"   # the lab names the bucket after the Project ID
echo "Bucket name: $BUCKET  (the lab uses your Project ID as the bucket name)"

cd "$WORKDIR" || exit 1

write_main() {   # write_main local|gcs  [force_destroy 0|1]
  {
    if [ "$1" = "gcs" ]; then cat <<EOF
terraform {
  backend "gcs" {
    bucket = "$BUCKET"
    prefix = "terraform/state"
  }
}
EOF
    else cat <<'EOF'
terraform {
  backend "local" {
    path = "terraform/state/terraform.tfstate"
  }
}
EOF
    fi
    cat <<EOF

provider "google" {
  project = "$PROJECT_ID"
  region  = "$REGION"
}

resource "google_storage_bucket" "test-bucket-for-state" {
  name                        = "$BUCKET"
  location                    = "US"
  uniform_bucket_level_access = true
EOF
    [ "${2:-0}" = 1 ] && echo "  force_destroy               = true"
    echo "}"
  } > main.tf
}

# ---------- TASK 1: backends (the graded part) ----------
if [ "$DONE" -lt 1 ]; then
  echo; echo "=== TASK 1a: local backend - create bucket $BUCKET ==="
  write_main local
  terraform init -input=false || fail "terraform init failed"
  terraform apply -auto-approve -input=false || fail "terraform apply failed"

  echo; echo "=== TASK 1b: move the state to the Cloud Storage backend (gs://$BUCKET/terraform/state) ==="
  write_main gcs
  terraform init -input=false -migrate-state -force-copy || fail "could not move the state to gs://$BUCKET"
  gcloud storage ls "gs://$BUCKET/terraform/state/default.tfstate" </dev/null >/dev/null 2>&1 \
    && echo "OK: gs://$BUCKET/terraform/state/default.tfstate exists" \
    || fail "state file not found in gs://$BUCKET/terraform/state"

  echo; echo "=== TASK 1c: add label key=value to the bucket and refresh the state ==="
  gcloud storage buckets update "gs://$BUCKET" --update-labels=key=value </dev/null || fail "could not add the label"
  terraform refresh -input=false || fail "terraform refresh failed"
  terraform show -no-color | grep -A 3 'labels' | head -n 8
  mark 1
fi

if [ "$DONE" -lt 2 ]; then
  echo
  echo ">>> Lab page: click 'Check my progress' for: Work with backends (100)"
  read -r -p ">>> Press ENTER here only AFTER it turns green - the next step deletes the bucket " _
  mark 2
fi

# ---------- TASK 1 clean up: back to local backend, destroy ----------
if [ "$DONE" -lt 3 ]; then
  echo; echo "=== TASK 1d: clean up - move state back to local, add force_destroy, destroy the bucket ==="
  write_main local 1
  terraform init -input=false -migrate-state -force-copy || fail "could not move the state back to local"
  terraform apply -auto-approve -input=false || fail "terraform apply failed"
  terraform destroy -auto-approve -input=false || fail "terraform destroy failed"
  mark 3
fi

# ---------- TASK 2: import a Docker container (optional, not graded) ----------
if [ "$DONE" -lt 4 ] && [ "$DO_TASK2" = "y" ]; then
  echo; echo "=== TASK 2a: start the hashicorp-learn NGINX container ==="
  docker rm -f hashicorp-learn >/dev/null 2>&1
  docker run --name hashicorp-learn --detach --publish 8080:80 nginx:latest || fail "docker run failed"
  docker ps --filter "name=hashicorp-learn"

  echo; echo "=== TASK 2b: clone learn-terraform-import and initialize ==="
  cd "$HOME" || exit 1
  rm -rf learn-terraform-import
  git clone https://github.com/hashicorp/learn-terraform-import.git || fail "git clone failed"
  cd learn-terraform-import || exit 1
  sed -i 's/version = "~> 3.0.2"/version = ">= 3.5"/' terraform.tf
  sed -i 's/^\([[:space:]]*\)host[[:space:]]*=/\1# host =/' main.tf     # comment out the host argument
  echo 'resource "docker_container" "web" {}' > docker.tf
  terraform init -upgrade -input=false || fail "terraform init failed"

  echo; echo "=== TASK 2c: import the container ==="
  terraform import -input=false docker_container.web "$(docker inspect -f '{{.ID}}' hashicorp-learn)" || fail "import failed"
  IMAGE_ID=$(docker inspect -f '{{.Image}}' hashicorp-learn)

  echo; echo "=== TASK 2d: write the minimal configuration and apply ==="
  cat > docker.tf <<EOF
resource "docker_container" "web" {
  image = "$IMAGE_ID"
  name  = "hashicorp-learn"

  ports {
    external = 8080
    internal = 80
    ip       = "0.0.0.0"
    protocol = "tcp"
  }
}
EOF
  terraform apply -auto-approve -input=false || fail "terraform apply failed"

  echo; echo "=== TASK 2e: manage the image as a resource ==="
  docker image inspect "$IMAGE_ID" -f '{{.RepoTags}}'
  cat >> docker.tf <<'EOF'

resource "docker_image" "nginx" {
  name = "nginx:latest"
}
EOF
  terraform apply -auto-approve -input=false || fail "terraform apply failed"
  sed -i "s|image = \"$IMAGE_ID\"|image = docker_image.nginx.image_id|" docker.tf
  terraform apply -auto-approve -input=false || fail "terraform apply failed"

  echo; echo "=== TASK 2f: change the external port to 8081 ==="
  sed -i 's/external = 8080/external = 8081/' docker.tf
  terraform apply -auto-approve -input=false || fail "terraform apply failed"
  docker ps --filter "name=hashicorp-learn"

  echo; echo "=== TASK 2g: destroy ==="
  terraform destroy -auto-approve -input=false || fail "terraform destroy failed"
  docker ps --filter "name=hashicorp-learn"
  mark 4
fi

echo
echo ">>> Done. The only checkpoint is 'Work with backends' (100) - make sure it is green on the lab page."
LABEOF
```




bash state_lab.sh


