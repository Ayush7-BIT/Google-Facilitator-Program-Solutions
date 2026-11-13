```/text
set +H
cat > secfund_lab.sh <<'LABEOF'
#!/bin/bash
# Implement Cloud Security Fundamentals on Google Cloud: Challenge Lab (GSP342)
# Run: bash secfund_lab.sh   (3 questions, then runs by itself, ~10 min)
die() { echo; echo "STOP: $1"; exit 1; }
ask() { local r; read -e -r -p "$2 [$3]: " r; printf -v "$1" '%s' "${r:-$3}"; }
P=$(gcloud config get-value project 2>/dev/null)
[ -z "$P" ] && die "run: gcloud config set project <Project ID>"
JH=orca-jumphost
ZONE=$(gcloud compute instances list --filter="name=$JH" --format="value(zone.basename())")
[ -n "$ZONE" ] || die "$JH not found - wait for the lab setup to finish"
DR=$(gcloud iam roles list --project="$P" --format="value(name.basename())" | head -1)
DS=$(gcloud iam service-accounts list --format="value(email)" \
  | grep -i orca | head -1 | cut -d@ -f1)
echo "Project $P | jumphost zone $ZONE"
echo; echo "=== Enter the YELLOW values from the lab text (Enter keeps the [default]) ==="
ask ROLE "Custom security role ID (Task 1)" "$DR"
ask SA   "Service account name, part before @ (Task 2)" "$DS"
CL=""; while [ -z "$CL" ]; do read -e -r -p "Cluster name (Task 4, yellow): " CL; done
SAE="$SA@$P.iam.gserviceaccount.com"

echo; echo "=== Task 1: custom role $ROLE ==="
gcloud iam roles describe "$ROLE" --project="$P" --format="value(includedPermissions)" \
  || die "role $ROLE not found (check the name)"
echo ">>> Task 1 DONE - click 'Check my progress'"

echo; echo "=== Task 2: service account $SAE ==="
gcloud iam service-accounts describe "$SAE" --format="value(email)" \
  || die "service account $SAE not found (check the name)"
echo ">>> Task 2 DONE - click 'Check my progress'"

echo; echo "=== Task 3: bind 4 roles to the service account ==="
for r in roles/monitoring.viewer roles/monitoring.metricWriter \
         roles/logging.logWriter "projects/$P/roles/$ROLE"; do
  gcloud projects add-iam-policy-binding "$P" --member="serviceAccount:$SAE" \
    --role="$r" --condition=None --quiet >/dev/null && echo "   bound $r"
done
echo ">>> Task 3 DONE - click 'Check my progress'"

echo; echo "=== Task 4: private cluster $CL in $ZONE (5-8 min) ==="
JIP=$(gcloud compute instances describe $JH --zone=$ZONE \
  --format="get(networkInterfaces[0].networkIP)")
echo "   jumphost internal IP: $JIP"
gcloud container clusters describe "$CL" --zone="$ZONE" >/dev/null 2>&1 || \
gcloud container clusters create "$CL" --zone="$ZONE" --num-nodes=1 \
  --disk-type=pd-standard --disk-size=50 \
  --network=orca-build-vpc --subnetwork=orca-build-subnet \
  --service-account="$SAE" \
  --enable-master-authorized-networks --master-authorized-networks="$JIP/32" \
  --enable-ip-alias --enable-private-nodes --enable-private-endpoint \
  --master-ipv4-cidr=172.16.0.0/28 --quiet || die "cluster creation failed"
echo ">>> Task 4 DONE - click 'Check my progress'"

echo; echo "=== Task 5: deploy hello-server from $JH ==="
cat > ~/jh.sh <<EOF
sudo apt-get update -qq
sudo apt-get install -y -qq kubectl google-cloud-cli-gke-gcloud-auth-plugin
export USE_GKE_GCLOUD_AUTH_PLUGIN=True
grep -q USE_GKE ~/.bashrc || echo "export USE_GKE_GCLOUD_AUTH_PLUGIN=True" >> ~/.bashrc
gcloud container clusters get-credentials $CL --internal-ip --project=$P --zone $ZONE
kubectl get deployment hello-server >/dev/null 2>&1 || \
  kubectl create deployment hello-server --image=gcr.io/google-samples/hello-app:1.0
kubectl get service hello-server >/dev/null 2>&1 || \
  kubectl expose deployment hello-server --type=LoadBalancer --port 8080
kubectl get deployments,pods
EOF
ok=0
for i in 1 2 3; do
  gcloud compute ssh $JH --zone=$ZONE --quiet --command "bash -s" < ~/jh.sh && { ok=1; break; }
  gcloud compute ssh $JH --zone=$ZONE --quiet --tunnel-through-iap \
    --command "bash -s" < ~/jh.sh && { ok=1; break; }
  echo "   ssh failed ($i/3) - retry in 20s"; sleep 20
done
[ $ok = 1 ] || die "could not reach $JH - use its SSH button in the console and run the lines in ~/jh.sh"
echo ">>> Task 5 DONE - click 'Check my progress'"
echo; echo ">>> All tasks finished."
LABEOF
echo "SCRIPT SAVED OK - now run: bash secfund_lab.sh"
```


bash secfund_lab.sh
