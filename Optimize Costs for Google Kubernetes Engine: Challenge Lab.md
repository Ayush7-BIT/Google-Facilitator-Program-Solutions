```/text
cat > gke_cost_lab.sh <<'LABEOF'
#!/bin/bash
# Universal solver: Optimize Costs for Google Kubernetes Engine: Challenge Lab (GSP343)
# Run in Cloud Shell with:  bash gke_cost_lab.sh      (safe to re-run)

ask() { local r; read -e -r -p "$2 [$3]: " r; printf -v "$1" '%s' "${r:-$3}"; }
ask_req() {   # required value, no stale default
  local r=""
  while [ -z "$r" ]; do
    read -e -r -p "$2${3:+ [$3]}: " r; r="${r:-$3}"
    [ -z "$r" ] && echo "   (required - copy it from the yellow text)"
  done
  printf -v "$1" '%s' "$r"
}
die() { echo; echo "STOP: $1"; exit 1; }

NS=dev
FRONTEND_IMAGE="gcr.io/qwiklabs-resources/onlineboutique-frontend:v2.1"

PROJECT_ID=$(gcloud config get-value project 2>/dev/null)
if [ -z "$PROJECT_ID" ] || [ "$PROJECT_ID" = "(unset)" ]; then
  die "no project set. Run: gcloud config set project <Project ID from the lab panel>"
fi
echo "Project: $PROJECT_ID"

DET_ZONE=$(gcloud config get-value compute/zone 2>/dev/null)
if [ -z "$DET_ZONE" ] || [ "$DET_ZONE" = "(unset)" ]; then
  DET_ZONE=$(gcloud compute project-info describe --quiet \
    --format="value(commonInstanceMetadata.items[google-compute-default-zone])" 2>/dev/null </dev/null)
fi
[ "$DET_ZONE" = "(unset)" ] && DET_ZONE=""
DET_CLUSTER=$(gcloud container clusters list --format="value(name)" 2>/dev/null | head -1)

echo
echo "=== Enter the YELLOW values from the lab text (press Enter to keep the [default]) ==="
ask_req CLUSTER "Cluster name (Task 1: 'create a cluster ... named ...')"                 "$DET_CLUSTER"
ask_req ZONE    "Zone (Task 1: 'in the ... zone')"                                        "$DET_ZONE"
ask_req POOL    "New node pool name (Task 2: 'Create a new node pool named ...')"          ""
ask_req HPA_MAX "Frontend HPA max replicas (Task 4: 'maximum of ... replicas')"           ""
ask     LOAD    "Run the optional load test at the end (Task 4, not scored)? (y/n)"       "n"
ask     PAUSE   "Pause so you can click 'Check my progress'? (y/n)"                       "y"
gcloud config set compute/zone "$ZONE" >/dev/null 2>&1

pause() {
  if [ "$PAUSE" = "y" ]; then
    read -r -p ">>> Lab page: click 'Check my progress' for: $1 -- then press ENTER here to continue " _
  fi
}

gcloud services enable container.googleapis.com 2>/dev/null || true

# ================= Task 1 =================
echo
echo "=== Task 1: zonal cluster $CLUSTER in $ZONE (e2-standard-2, 2 nodes, rapid) ==="
if gcloud container clusters describe "$CLUSTER" --zone="$ZONE" >/dev/null 2>&1; then
  echo "   cluster already exists"
else
  echo "   creating the cluster (takes about 5-8 minutes)..."
  gcloud container clusters create "$CLUSTER" --zone="$ZONE" \
    --machine-type=e2-standard-2 --num-nodes=2 --release-channel=rapid --quiet \
    || die "could not create the cluster (error above)"
fi
gcloud container clusters get-credentials "$CLUSTER" --zone="$ZONE" || die "could not get cluster credentials"

kubectl create namespace dev  --dry-run=client -o yaml | kubectl apply -f -
kubectl create namespace prod --dry-run=client -o yaml | kubectl apply -f -

cd ~ || exit 1
[ -d microservices-demo ] || git clone https://github.com/GoogleCloudPlatform/microservices-demo.git
cd microservices-demo || die "git clone failed"
kubectl apply -f ./release/kubernetes-manifests.yaml --namespace "$NS" || die "could not deploy the app"

echo "   waiting for all deployments in $NS to be ready (a few minutes)..."
kubectl wait --for=condition=available deployment --all -n "$NS" --timeout=600s || echo "   (some pods still starting - continuing)"
echo "   waiting for the frontend-external IP..."
FE_IP=""
for i in $(seq 1 40); do
  FE_IP=$(kubectl get svc frontend-external -n "$NS" -o jsonpath='{.status.loadBalancer.ingress[0].ip}' 2>/dev/null)
  [ -n "$FE_IP" ] && break; sleep 10
done
echo "   store: http://$FE_IP"
for i in $(seq 1 30); do
  curl -s -m 5 -o /dev/null -w '%{http_code}' "http://$FE_IP" 2>/dev/null | grep -q '^200' && { echo "   store is up (HTTP 200)"; break; }
  sleep 10
done
pause "Create the cluster and deploy the app (40 points)"

# ================= Task 2 =================
echo
echo "=== Task 2: node pool $POOL (e2-custom-2-3584, 2 nodes), then remove default-pool ==="
if gcloud container node-pools describe "$POOL" --cluster="$CLUSTER" --zone="$ZONE" >/dev/null 2>&1; then
  echo "   node pool already exists"
else
  gcloud container node-pools create "$POOL" --cluster="$CLUSTER" --zone="$ZONE" \
    --machine-type=e2-custom-2-3584 --num-nodes=2 --quiet || die "could not create the node pool"
fi

if gcloud container node-pools describe default-pool --cluster="$CLUSTER" --zone="$ZONE" >/dev/null 2>&1; then
  echo "   cordoning default-pool nodes..."
  for node in $(kubectl get nodes -l cloud.google.com/gke-nodepool=default-pool -o=name); do
    kubectl cordon "$node"
  done
  echo "   draining default-pool nodes (moves the pods to $POOL)..."
  for node in $(kubectl get nodes -l cloud.google.com/gke-nodepool=default-pool -o=name); do
    kubectl drain "$node" --ignore-daemonsets --delete-emptydir-data --force --grace-period=10 --timeout=300s \
      || echo "   (drain of $node reported a problem - continuing)"
  done
  kubectl wait --for=condition=available deployment --all -n "$NS" --timeout=300s || true
  echo "   deleting default-pool..."
  gcloud container node-pools delete default-pool --cluster="$CLUSTER" --zone="$ZONE" --quiet \
    || die "could not delete default-pool"
else
  echo "   default-pool already deleted"
fi
kubectl get nodes
pause "Migrate to an optimized node pool (20 points)"

# ================= Task 3 =================
echo
echo "=== Task 3: PDB onlineboutique-frontend-pdb + frontend image v2.1 (imagePullPolicy Always) ==="
kubectl get pdb onlineboutique-frontend-pdb -n "$NS" >/dev/null 2>&1 \
  || kubectl create poddisruptionbudget onlineboutique-frontend-pdb -n "$NS" \
       --selector app=frontend --min-available 1 || die "could not create the PDB"
kubectl patch deployment frontend -n "$NS" --type=json -p "[
  {\"op\":\"replace\",\"path\":\"/spec/template/spec/containers/0/image\",\"value\":\"$FRONTEND_IMAGE\"},
  {\"op\":\"replace\",\"path\":\"/spec/template/spec/containers/0/imagePullPolicy\",\"value\":\"Always\"}
]" || die "could not update the frontend deployment"
kubectl rollout status deployment/frontend -n "$NS" --timeout=300s || true
kubectl get deployment frontend -n "$NS" -o jsonpath='{.spec.template.spec.containers[0].image}{"  "}{.spec.template.spec.containers[0].imagePullPolicy}{"\n"}'
pause "Apply a frontend update (20 points)"

# ================= Task 4 =================
echo
echo "=== Task 4: frontend HPA (50% CPU, 1-$HPA_MAX), cluster autoscaler (1-6), recommendationservice HPA (50%, 1-5) ==="
kubectl get hpa frontend -n "$NS" >/dev/null 2>&1 && kubectl delete hpa frontend -n "$NS" >/dev/null
kubectl autoscale deployment frontend -n "$NS" --cpu-percent=50 --min=1 --max="$HPA_MAX" || die "could not create the frontend HPA"
gcloud container clusters update "$CLUSTER" --zone="$ZONE" --node-pool="$POOL" \
  --enable-autoscaling --min-nodes=1 --max-nodes=6 --quiet || die "could not enable cluster autoscaling"
kubectl get hpa recommendationservice -n "$NS" >/dev/null 2>&1 && kubectl delete hpa recommendationservice -n "$NS" >/dev/null
kubectl autoscale deployment recommendationservice -n "$NS" --cpu-percent=50 --min=1 --max=5 || echo "   (could not create the recommendationservice HPA)"
kubectl get hpa -n "$NS"
pause "Autoscale from estimated traffic (20 points)"

if [ "$LOAD" = "y" ]; then
  echo
  echo "=== Optional load test (3 minutes, not scored) ==="
  LG=$(kubectl get pod -n "$NS" | grep loadgenerator | cut -f1 -d ' ')
  timeout 180 kubectl exec "$LG" -n "$NS" -- sh -c "export USERS=8000; locust --host=\"http://$FE_IP\" --headless -u \"8000\" 2>&1" | tail -n 20
  kubectl get hpa -n "$NS"
fi

# ================= Self-check =================
echo
echo "=== Self-check ==="
check() { if eval "$1"; then echo "  PASS  $2"; else echo "  FAIL  $2"; fi; }
check '[ "$(gcloud container clusters describe $CLUSTER --zone=$ZONE --format="value(releaseChannel.channel)")" = "RAPID" ]' "Task 1: cluster $CLUSTER on the rapid channel"
check 'kubectl get ns dev prod >/dev/null 2>&1' "Task 1: namespaces dev and prod"
check 'kubectl get svc frontend-external -n dev >/dev/null 2>&1' "Task 1: app deployed in dev"
check '[ "$(gcloud container node-pools describe $POOL --cluster=$CLUSTER --zone=$ZONE --format="value(config.machineType)")" = "e2-custom-2-3584" ]' "Task 2: $POOL uses e2-custom-2-3584"
check '! gcloud container node-pools describe default-pool --cluster=$CLUSTER --zone=$ZONE >/dev/null 2>&1' "Task 2: default-pool deleted"
check 'kubectl get pdb onlineboutique-frontend-pdb -n dev >/dev/null 2>&1' "Task 3: PDB onlineboutique-frontend-pdb"
check 'kubectl get deploy frontend -n dev -o jsonpath="{.spec.template.spec.containers[0].image}" | grep -q "v2.1"' "Task 3: frontend image v2.1"
check 'kubectl get hpa frontend -n dev >/dev/null 2>&1' "Task 4: frontend HPA"
check '[ "$(gcloud container node-pools describe $POOL --cluster=$CLUSTER --zone=$ZONE --format="value(autoscaling.maxNodeCount)")" = "6" ]' "Task 4: cluster autoscaler max 6 nodes"
check 'kubectl get hpa recommendationservice -n dev >/dev/null 2>&1' "Task 4: recommendationservice HPA"
echo
echo ">>> Done. 4 checkpoints: cluster+app (40), node pool (20), frontend update (20), autoscaling (20)."
LABEOF
```




bash gke_cost_lab.sh
