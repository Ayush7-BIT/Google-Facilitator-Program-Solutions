```/text
cat > terraform_modules_lab.sh <<'LABEOF'
#!/bin/bash
# Interact with Terraform Modules (GSP751) - universal solution

die() { echo; echo "STOP: $1"; exit 1; }
ask() { local r; read -e -r -p "$1 [$2]: " r; printf -v "$3" '%s' "${r:-$2}"; }

PROJECT_ID=$(gcloud config get-value project 2>/dev/null)
if [ -z "$PROJECT_ID" ] || [ "$PROJECT_ID" = "(unset)" ]; then
  die "no project set. Run: gcloud config set project <Project ID from the lab panel>"
fi
export GOOGLE_CLOUD_PROJECT="$PROJECT_ID"
echo "Project: $PROJECT_ID"

# ---------- detect the region ----------
meta() { gcloud compute project-info describe --quiet --format="value(commonInstanceMetadata.items[$1])" 2>/dev/null </dev/null; }
DET_REGION=$(gcloud config get-value compute/region 2>/dev/null)
if [ -z "$DET_REGION" ] || [ "$DET_REGION" = "(unset)" ]; then DET_REGION=$(meta google-compute-default-region); fi
if [ -z "$DET_REGION" ]; then
  DET_ZONE=$(meta google-compute-default-zone)
  [ -n "$DET_ZONE" ] && DET_REGION="${DET_ZONE%-*}"
fi
if [ -z "$DET_REGION" ]; then
  echo "Could not detect the region - type the one from the lab text."
  DET_REGION="us-west1"
fi

echo
echo "=== Press ENTER to accept each default (check them against the lab text) ==="
ask "Region (the 'subnet_region' / 'location' value)   " "$DET_REGION" REGION
ask "Network name (Task 1)                              " "example-vpc" NETNAME
ask "Bucket name (Task 2 - lab uses the Project ID)     " "$PROJECT_ID" BUCKET
ask "Run terraform destroy after the checks, like the lab? (y/n)" "y" DESTROY
ask "Pause so you can click 'Check my progress'? (y/n)  " "y" PAUSE

pause() {
  if [ "$PAUSE" = "y" ]; then
    read -r -p ">>> Lab page: click 'Check my progress' for: $1 -- then press ENTER here to continue " _
  fi
}

# The Compute Engine API is normally enabled in this lab; students often cannot enable APIs.
gcloud services list --enabled --filter="config.name=compute.googleapis.com" --format="value(config.name)" 2>/dev/null | grep -q . \
  || gcloud services enable compute.googleapis.com --quiet >/dev/null 2>&1 || true

# ---------- install Terraform ----------
echo
echo "=== Prerequisite: install Terraform ==="
find_tf() {   # prints a REAL terraform (Cloud Shell has a placeholder that only prints instructions)
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
  echo "Terraform is not installed yet. Installing it the way the lab does..."
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

# =====================================================================
# TASK 1: use a module from the Registry
# =====================================================================
echo
echo "=== TASK 1: use the Terraform Registry network module ==="
cd "$HOME" || exit 1
if [ -d terraform-google-network ]; then
  echo "A previous terraform-google-network folder exists - cleaning it up first"
  (cd terraform-google-network/examples/simple_project 2>/dev/null && "$TF" destroy -auto-approve -input=false >/dev/null 2>&1)
  rm -rf terraform-google-network
fi
git clone https://github.com/terraform-google-modules/terraform-google-network || die "git clone failed"
cd terraform-google-network || exit 1
git checkout tags/v6.0.1 -b v6.0.1 || die "could not check out v6.0.1"
cd examples/simple_project || die "examples/simple_project not found"

cat > variables.tf <<EOF
variable "project_id" {
  description = "The project ID to host the network in"
  default     = "${PROJECT_ID}"
}

variable "network_name" {
  description = "The name of the network to be created"
  default     = "${NETNAME}"
}
EOF

cat > main.tf <<EOF
module "test-vpc-module" {
  source       = "terraform-google-modules/network/google"
  version      = "~> 6.0"
  project_id   = var.project_id
  network_name = var.network_name
  mtu          = 1460

  subnets = [
    {
      subnet_name   = "subnet-01"
      subnet_ip     = "10.10.10.0/24"
      subnet_region = "${REGION}"
    },
    {
      subnet_name           = "subnet-02"
      subnet_ip             = "10.10.20.0/24"
      subnet_region         = "${REGION}"
      subnet_private_access = "true"
      subnet_flow_logs      = "true"
    },
    {
      subnet_name               = "subnet-03"
      subnet_ip                 = "10.10.30.0/24"
      subnet_region             = "${REGION}"
      subnet_flow_logs          = "true"
      subnet_flow_logs_interval = "INTERVAL_10_MIN"
      subnet_flow_logs_sampling = 0.7
      subnet_flow_logs_metadata = "INCLUDE_ALL_METADATA"
      subnet_flow_logs_filter   = "false"
    }
  ]
}
EOF
echo "--- outputs.tf (from the repository) ---"; head -n 8 outputs.tf

"$TF" init -input=false || die "terraform init failed in simple_project"
applied=0
for i in 1 2 3; do
  if "$TF" apply -auto-approve -input=false; then applied=1; break; fi
  echo "   apply failed (try $i/3) - waiting 20s"; sleep 20
done
[ "$applied" = 1 ] || die "terraform apply (VPC) did not finish - send me the error above"
VPC_OK=0
gcloud compute networks describe "$NETNAME" --project="$PROJECT_ID" >/dev/null 2>&1 && VPC_OK=1
[ "$VPC_OK" = 1 ] && echo "PASS  VPC network $NETNAME exists" || echo "CHECK VPC network $NETNAME not visible yet"
pause "Provision infrastructure (50) - click it now while the VPC exists"

echo
echo "--- Clean up the VPC (as the lab does) ---"
"$TF" destroy -auto-approve -input=false
cd "$HOME" && rm -rf terraform-google-network
pause "Provision infrastructure (50) - the lab clicks it here, after the cleanup (click again if it was not green)"

# =====================================================================
# TASK 2: build a module
# =====================================================================
echo
echo "=== TASK 2: build the gcs-static-website-bucket module ==="
ROOT="$HOME/terraform-modules-lab"
MOD="$ROOT/modules/gcs-static-website-bucket"
mkdir -p "$MOD" && cd "$ROOT" || die "could not create $ROOT"

cat > "$MOD/README.md" <<'EOF'
# GCS static website bucket

This module provisions Cloud Storage buckets configured for static website hosting.
EOF

cat > "$MOD/LICENSE" <<'EOF'
Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
EOF

cat > "$MOD/website.tf" <<'EOF'
resource "google_storage_bucket" "bucket" {
  name                        = var.name
  project                     = var.project_id
  location                    = var.location
  storage_class               = var.storage_class
  labels                      = var.labels
  force_destroy               = var.force_destroy
  uniform_bucket_level_access = true

  versioning {
    enabled = var.versioning
  }

  dynamic "retention_policy" {
    for_each = var.retention_policy == null ? [] : [var.retention_policy]
    content {
      is_locked        = var.retention_policy.is_locked
      retention_period = var.retention_policy.retention_period
    }
  }

  dynamic "encryption" {
    for_each = var.encryption == null ? [] : [var.encryption]
    content {
      default_kms_key_name = var.encryption.default_kms_key_name
    }
  }

  dynamic "lifecycle_rule" {
    for_each = var.lifecycle_rules
    content {
      action {
        type          = lifecycle_rule.value.action.type
        storage_class = lookup(lifecycle_rule.value.action, "storage_class", null)
      }
      condition {
        age                   = lookup(lifecycle_rule.value.condition, "age", null)
        created_before        = lookup(lifecycle_rule.value.condition, "created_before", null)
        with_state            = lookup(lifecycle_rule.value.condition, "with_state", null)
        matches_storage_class = lookup(lifecycle_rule.value.condition, "matches_storage_class", null)
        num_newer_versions    = lookup(lifecycle_rule.value.condition, "num_newer_versions", null)
      }
    }
  }
}
EOF

cat > "$MOD/variables.tf" <<'EOF'
variable "name" {
  description = "The name of the bucket."
  type        = string
}

variable "project_id" {
  description = "The ID of the project to create the bucket in."
  type        = string
}

variable "location" {
  description = "The location of the bucket."
  type        = string
}

variable "storage_class" {
  description = "The Storage Class of the new bucket."
  type        = string
  default     = null
}

variable "labels" {
  description = "A set of key/value label pairs to assign to the bucket."
  type        = map(string)
  default     = null
}

variable "bucket_policy_only" {
  description = "Enables Bucket Policy Only access to a bucket."
  type        = bool
  default     = true
}

variable "versioning" {
  description = "While set to true, versioning is fully enabled for this bucket."
  type        = bool
  default     = true
}

variable "force_destroy" {
  description = "When deleting a bucket, this boolean option will delete all contained objects. If false, Terraform will fail to delete buckets which contain objects."
  type        = bool
  default     = true
}

variable "iam_members" {
  description = "The list of IAM members to grant permissions on the bucket."
  type = list(object({
    role   = string
    member = string
  }))
  default = []
}

variable "retention_policy" {
  description = "Configuration of the bucket's data retention policy for how long objects in the bucket should be retained."
  type = object({
    is_locked        = bool
    retention_period = number
  })
  default = null
}

variable "encryption" {
  description = "A Cloud KMS key that will be used to encrypt objects inserted into this bucket"
  type = object({
    default_kms_key_name = string
  })
  default = null
}

variable "lifecycle_rules" {
  description = "The bucket's Lifecycle Rules configuration."
  type = list(object({
    # Object with keys:
    # - type - The type of the action of this Lifecycle Rule. Supported values: Delete and SetStorageClass.
    # - storage_class - (Required if action type is SetStorageClass) The target Storage Class of objects affected by this Lifecycle Rule.
    action = any

    # Object with keys:
    # - age - (Optional) Minimum age of an object in days to satisfy this condition.
    # - created_before - (Optional) Creation date of an object in RFC 3339 (e.g. 2017-06-13) to satisfy this condition.
    # - with_state - (Optional) Match to live and/or archived objects. Supported values include: "LIVE", "ARCHIVED", "ANY".
    # - matches_storage_class - (Optional) Storage Class of objects to satisfy this condition.
    # - num_newer_versions - (Optional) Relevant only for versioned objects. The number of newer versions of an object to satisfy this condition.
    condition = any
  }))
  default = []
}
EOF

cat > "$MOD/outputs.tf" <<'EOF'
output "bucket" {
  description = "The created storage bucket"
  value       = google_storage_bucket.bucket
}
EOF

cat > main.tf <<EOF
module "gcs-static-website-bucket" {
  source = "./modules/gcs-static-website-bucket"

  name       = var.name
  project_id = var.project_id
  location   = "${REGION}"

  lifecycle_rules = [{
    action = {
      type = "Delete"
    }
    condition = {
      age        = 365
      with_state = "ANY"
    }
  }]
}
EOF

cat > outputs.tf <<'EOF'
output "bucket-name" {
  description = "Bucket names."
  value       = "module.gcs-static-website-bucket.bucket"
}
EOF

cat > variables.tf <<EOF
variable "project_id" {
  description = "The ID of the project in which to provision resources."
  type        = string
  default     = "${PROJECT_ID}"
}

variable "name" {
  description = "Name of the buckets to create."
  type        = string
  default     = "${BUCKET}"
}
EOF

echo "--- module structure ---"
find . -maxdepth 3 -not -path './.terraform*' -type f | sort

"$TF" init -input=false || die "terraform init failed in $ROOT"
applied=0
for i in 1 2 3; do
  if "$TF" apply -auto-approve -input=false; then applied=1; break; fi
  echo "   apply failed (try $i/3) - waiting 20s"; sleep 20
done
[ "$applied" = 1 ] || die "terraform apply (bucket) did not finish - send me the error above"

# ---------- upload the files ----------
echo
echo "=== Upload files to the bucket ==="
cd "$ROOT" || exit 1
BASE="https://raw.githubusercontent.com/hashicorp/learn-terraform-modules/master/modules/aws-s3-static-website-bucket/www"
curl -fsSL "$BASE/index.html" -o index.html || rm -f index.html
curl -fsSL "$BASE/error.html" -o error.html || rm -f error.html
[ -s index.html ] || cat > index.html <<'EOF'
<html lang="en">
  <head>
    <meta charset="utf-8">
    <title>Nothing to see here</title>
  </head>
  <body>
    <h1>Nothing to see here</h1>
  </body>
</html>
EOF
[ -s error.html ] || cat > error.html <<'EOF'
<html lang="en">
  <head>
    <meta charset="utf-8">
    <title>Error</title>
  </head>
  <body>
    <h1>Error: page not found</h1>
  </body>
</html>
EOF
ls -l *.html
gcloud storage cp ./*.html "gs://${BUCKET}" || die "could not upload the files to gs://${BUCKET}"
echo "Website file: https://storage.cloud.google.com/${BUCKET}/index.html"

# ---------- SELF-CHECK (before destroying anything) ----------
echo
echo "=================== SELF-CHECK ==================="
ok()  { echo "  PASS  $1"; }
bad() { echo "  FAIL  $1"; }
[ "$VPC_OK" = 1 ] && ok "Task 1: the VPC $NETNAME was created from the Registry module (and cleaned up afterwards)" || echo "  CHECK Task 1: the VPC was not confirmed"
BINFO=$(gcloud storage buckets describe "gs://${BUCKET}" --format=json 2>/dev/null)
[ -n "$BINFO" ] && ok "Task 2: bucket gs://${BUCKET} exists" || bad "Task 2: bucket missing"
echo "$BINFO" | grep -qi "\"location\": \"${REGION}\"" && ok "Task 2: bucket location is $REGION" || echo "  CHECK Task 2: bucket location differs from $REGION"
gcloud storage ls "gs://${BUCKET}/index.html" >/dev/null 2>&1 && ok "Task 2: index.html is in the bucket" || bad "Task 2: index.html missing"
gcloud storage ls "gs://${BUCKET}/error.html" >/dev/null 2>&1 && ok "Task 2: error.html is in the bucket" || bad "Task 2: error.html missing"
[ -f "$MOD/website.tf" ] && [ -f "$MOD/variables.tf" ] && [ -f "$MOD/outputs.tf" ] && ok "Task 2: the module has website.tf, variables.tf and outputs.tf" || bad "Task 2: module files missing"

pause "Upload files to the bucket (50) - click it NOW, before the bucket is destroyed"

# ---------- clean up ----------
if [ "$DESTROY" = "y" ]; then
  echo
  echo "=== Clean up the website and infrastructure (terraform destroy) ==="
  cd "$ROOT" && "$TF" destroy -auto-approve -input=false
else
  echo "Skipping terraform destroy. Run it later with:  cd $ROOT && $TF destroy -auto-approve"
fi
echo
echo ">>> Done. If a checkpoint is not green, click 'Check my progress' again, or send me the output."
LABEOF
```

bash terraform_modules_lab.sh
