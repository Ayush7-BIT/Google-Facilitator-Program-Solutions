```text
cat > lab3.sh <<'LABEOF'
#!/bin/bash
# Universal solver: Build a Website on Google Cloud - Challenge Lab
# Run in Cloud Shell with:  bash lab3.sh

REPO_URL="https://github.com/googlecodelabs/monolith-to-microservices.git"
VERSION="1.0.0"
MACHINE="e2-medium"
NODES=3

ask() { local r; read -e -r -p "$2 [$3]: " r; printf -v "$1" '%s' "${r:-$3}"; }
die() { echo; echo "STOP: $1"; exit 1; }

wait_ip() {
  local ip=""
  echo "   waiting for the external IP of $1 ..." >&2
  for i in $(seq 1 40); do
    ip=$(kubectl get service "$1" -o jsonpath='{.status.loadBalancer.ingress[0].ip}' 2>/dev/null)
    [ -n "$ip" ] && break
    sleep 10
  done
  echo "$ip"
}

deploy() {   # deploy <name> <application port>
  kubectl get deployment "$1" >/dev/null 2>&1 || kubectl create deployment "$1" --image="$REPO/$1:$VERSION"
  kubectl get service "$1" >/dev/null 2>&1 || kubectl expose deployment "$1" --type=LoadBalancer --port 80 --target-port "$2"
}

PROJECT=$(gcloud config get-value project 2>/dev/null)
[ -z "$PROJECT" ] && die "no project set. Run: gcloud config set project <Project ID from the lab panel>"
export GOOGLE_CLOUD_PROJECT="$PROJECT"
echo "Project: $PROJECT"

echo
echo "=== Enter the YELLOW values from the lab (press Enter to keep the default) ==="
ask ZONE      "Zone (Task 2: 'deploy all of your resources in the ... zone')" "us-west1-b"
ask CLUSTER   "Cluster name (Task 2)"                        "fancy-prod-523"
ask MONOLITH  "Monolith image / container name (Task 1)"     "fancy-monolith-609"
ask ORDERS    "Orders image / container name (Task 3)"       "fancy-orders-196"
ask PRODUCTS  "Products image / container name (Task 3)"     "fancy-products-378"
ask FRONTEND  "Frontend image / container name (Task 6)"     "fancy-frontend-191"
REPO="gcr.io/${PROJECT}"

echo
echo "=== Enabling APIs and starting the cluster (it builds in the background) ==="
gcloud services enable container.googleapis.com cloudbuild.googleapis.com artifactregistry.googleapis.com compute.googleapis.com
gcloud artifacts repositories create gcr.io --repository-format=docker --location=us --project="$PROJECT" >/dev/null 2>&1 || true
gcloud config set compute/zone "$ZONE" >/dev/null 2>&1
gcloud container clusters create "$CLUSTER" --zone "$ZONE" --num-nodes "$NODES" --machine-type "$MACHINE" --async 2>&1 | tail -3

echo
echo "=== Task 1: monolith code and container ==="
cd ~
[ -d monolith-to-microservices ] || git clone "$REPO_URL"
cd ~/monolith-to-microservices || die "git clone failed"
if [ ! -d react-app/node_modules ]; then ./setup.sh || echo "(setup.sh reported errors - continuing)"; fi
export NVM_DIR="${NVM_DIR:-$HOME/.nvm}"
for f in "$NVM_DIR/nvm.sh" /usr/local/nvm/nvm.sh; do
  if [ -s "$f" ]; then . "$f"; break; fi
done
if type nvm >/dev/null 2>&1; then nvm install 22 && nvm use 22; else echo "(nvm not found - using the default node)"; fi
node -v
cd ~/monolith-to-microservices/monolith
gcloud builds submit --tag "$REPO/$MONOLITH:$VERSION" . || die "monolith build failed"

echo
echo "=== Task 2: cluster and monolith deployment ==="
for i in $(seq 1 60); do
  ST=$(gcloud container clusters describe "$CLUSTER" --zone "$ZONE" --format='value(status)' 2>/dev/null)
  [ "$ST" = "RUNNING" ] && break
  echo "   waiting for the cluster (status: ${ST:-creating}) ..."; sleep 15
done
[ "$ST" = "RUNNING" ] || die "cluster $CLUSTER is not RUNNING"
gcloud container clusters get-credentials "$CLUSTER" --zone "$ZONE"
deploy "$MONOLITH" 8080

echo
echo "=== Task 3: containerize Orders and Products ==="
for pair in "orders:$ORDERS" "products:$PRODUCTS"; do
  dir="${pair%%:*}"; name="${pair#*:}"
  cd ~/monolith-to-microservices/microservices/src/$dir || die "folder $dir not found"
  gcloud builds submit --tag "$REPO/$name:$VERSION" . || die "$dir build failed"
done

echo
echo "=== Task 4: deploy Orders (8081) and Products (8082) ==="
deploy "$ORDERS" 8081
deploy "$PRODUCTS" 8082
ORDERS_IP=$(wait_ip "$ORDERS")
PRODUCTS_IP=$(wait_ip "$PRODUCTS")
if [ -z "$ORDERS_IP" ] || [ -z "$PRODUCTS_IP" ]; then die "load balancer IPs are not ready; run 'bash lab3.sh' again in a minute"; fi
echo "Orders IP: $ORDERS_IP   Products IP: $PRODUCTS_IP"

echo
echo "=== Task 5: point the frontend at the new services ==="
cd ~/monolith-to-microservices/react-app
cat > .env <<EOF
REACT_APP_ORDERS_URL=http://${ORDERS_IP}/api/orders
REACT_APP_PRODUCTS_URL=http://${PRODUCTS_IP}/api/products
EOF
cat .env
npm install --no-audit --no-fund --silent
npm run build || die "npm run build failed"
if [ -d build ]; then rm -rf ../microservices/src/frontend/public && cp -r build ../microservices/src/frontend/public; fi

echo
echo "=== Task 6: containerize the Frontend ==="
cd ~/monolith-to-microservices/microservices/src/frontend
gcloud builds submit --tag "$REPO/$FRONTEND:$VERSION" . || die "frontend build failed"

echo
echo "=== Task 7: deploy the Frontend (8080 -> 80) ==="
deploy "$FRONTEND" 8080
FRONTEND_IP=$(wait_ip "$FRONTEND")

echo
echo "=== RESULT ==="
kubectl get deployments
kubectl get services
echo
echo "Orders   : http://$ORDERS_IP/api/orders   -> HTTP $(curl -s -o /dev/null -w '%{http_code}' http://$ORDERS_IP/api/orders)"
echo "Products : http://$PRODUCTS_IP/api/products -> HTTP $(curl -s -o /dev/null -w '%{http_code}' http://$PRODUCTS_IP/api/products)"
echo "Frontend : http://$FRONTEND_IP   -> HTTP $(curl -s -o /dev/null -w '%{http_code}' http://$FRONTEND_IP)"
echo
echo ">>> Done. On the lab page, click 'Check my progress' on every task."
LABEOF
```


bash lab3.sh

