```/text
set +H
cat > gsp091_lab.sh <<'LABEOF'
#!/bin/bash
# Creating and Alerting on Logs-based Metrics (GSP091)
# Run: bash gsp091_lab.sh   (3 questions, ~15 min, no ENTER after that)
# Safe to run again after a disconnect: finished tasks are skipped.
ask() { local r; read -e -r -p "$2 [$3]: " r; printf -v "$1" '%s' "${r:-$3}"; }
die() { echo; echo "STOP: $1 - send Claude a screenshot"; exit 1; }
P=$(gcloud config get-value project 2>/dev/null)
[ -z "$P" ] && die "run: gcloud config set project <Project ID>"
ENVF=~/.gsp091.env; D=~/.gsp091_done/$P; mkdir -p $D
step() { if [ -f $D/$1 ]; then echo; echo "=== $2: already done ==="; return 1
  fi; echo; echo "=== $2 ==="; }
done_() { touch $D/$1; }

if [ "$(grep -s '^P=' $ENVF | cut -d= -f2)" = "$P" ]; then
  source $ENVF; echo "=== Resuming (ZONE=$ZONE REGION=$REGION) ==="
else
  MD="value(commonInstanceMetadata.items.google-compute-default"
  DZ=$(gcloud compute project-info describe --format="$MD-zone)" 2>/dev/null)
  DR=$(gcloud compute project-info describe --format="$MD-region)" 2>/dev/null)
  ACC=$(gcloud config get-value account 2>/dev/null)
  echo "=== YELLOW values - press Enter to accept the suggestion ==="
  ask ZONE "ZONE  (Task 1: 'gcloud config set compute/zone ...')" \
    "${DZ:-us-central1-a}"
  ask REGION "REGION  (Task 3: '--location=...' of docker-repo)" "${DR:-${ZONE%-*}}"
  ask EMAIL "Email for alerts  (Task 2: any email; student username ok)" \
    "$ACC"
  printf 'P=%s\nZONE=%s\nREGION=%s\nEMAIL=%s\n' "$P" "$ZONE" "$REGION" \
    "$EMAIL" > $ENVF
fi
export PROJECT_ID=$P
gcloud config set compute/zone "$ZONE" -q 2>/dev/null
gcloud services enable container.googleapis.com artifactregistry.googleapis.com \
  monitoring.googleapis.com logging.googleapis.com -q >/dev/null 2>&1
IMG=$REGION-docker.pkg.dev/$P/docker-repo/flask-telemetry:v1
cd ~

# ---------- email notification channel (used by Task 2 and Task 6) ----------
CH=$(gcloud beta monitoring channels list --format="value(name)" \
  --filter="labels.email_address=$EMAIL" 2>/dev/null | head -1)
[ -z "$CH" ] && CH=$(gcloud beta monitoring channels create --type=email \
  --display-name="lab-email" --channel-labels=email_address=$EMAIL \
  --format="value(name)" -q)
[ -z "$CH" ] && die "could not create the email notification channel"

pol() { # name json-file
  gcloud alpha monitoring policies list --format="value(displayName)" \
    | grep -qx "$1" && { echo "   policy '$1' already exists"; return; }
  gcloud alpha monitoring policies create --policy-from-file="$2" -q \
    >/dev/null && echo "   policy '$1' created" || die "policy '$1' failed"; }

if step t1 "Task 1: GKE cluster gmp-cluster"; then
gcloud container clusters describe gmp-cluster --zone $ZONE >/dev/null 2>&1 || \
  gcloud container clusters create gmp-cluster --num-nodes=1 --zone $ZONE \
  || die "cluster create failed"
echo ">>> Task 1 DONE - click 'Check my progress' (Deploy the GKE cluster)"
done_ t1; fi

if step t2 "Task 2: log-based alert 'stopped vm'"; then
F2='resource.type=\"gce_instance\"'
F2="$F2"' protoPayload.methodName=\"v1.compute.instances.stop\"'
cat > stopped_vm.json <<EOF
{
  "displayName": "stopped vm",
  "combiner": "OR",
  "conditions": [ {
    "displayName": "Log match condition",
    "conditionMatchedLog": {
      "filter": "$F2"
    } } ],
  "alertStrategy": { "notificationRateLimit": { "period": "300s" },
                     "autoClose": "3600s" },
  "notificationChannels": [ "$CH" ]
}
EOF
pol "stopped vm" stopped_vm.json
echo ">>> Task 2 DONE - click 'Check my progress' (Create the Log-based alert)"
done_ t2; fi

if step t3 "Task 3: docker-repo + push flask-telemetry:v1"; then
gcloud artifacts repositories describe docker-repo --location=$REGION \
  >/dev/null 2>&1 || gcloud artifacts repositories create docker-repo \
  --repository-format=docker --location=$REGION \
  --description="Docker repository" --project=$P || die "repo failed"
gcloud auth configure-docker $REGION-docker.pkg.dev -q >/dev/null 2>&1
[ -f flask_telemetry.tar ] || { wget -q \
  https://storage.googleapis.com/spls/gsp1024/flask_telemetry.zip \
  && unzip -oq flask_telemetry.zip; }
docker load -i flask_telemetry.tar || die "docker load failed"
docker tag \
  gcr.io/ops-demo-330920/flask_telemetry:61a2a7aabc7077ef474eb24f4b69faeab47deed9 \
  $IMG || die "docker tag failed"
docker push $IMG || die "docker push failed"
echo ">>> Task 3 DONE - click 'Check my progress' (Create a Docker repository)"
done_ t3; fi

if step t4 "Task 4: deploy the metrics app to gmp-test"; then
gcloud container clusters get-credentials gmp-cluster --zone $ZONE \
  || die "get-credentials failed"
kubectl get ns gmp-test >/dev/null 2>&1 || kubectl create ns gmp-test
[ -d gmp_prom_setup ] || { wget -q \
  https://storage.googleapis.com/spls/gsp1024/gmp_prom_setup.zip \
  && unzip -oq gmp_prom_setup.zip; }
cd ~/gmp_prom_setup || die "gmp_prom_setup missing"
sed -i "s|<ARTIFACT REGISTRY IMAGE NAME>|$IMG|" flask_deployment.yaml
grep -n "image:" flask_deployment.yaml
kubectl -n gmp-test apply -f flask_deployment.yaml
kubectl -n gmp-test apply -f flask_service.yaml
echo "   waiting for the service external IP..."
for i in $(seq 1 40); do
  IP=$(kubectl get services -n gmp-test \
    -o jsonpath='{.items[*].status.loadBalancer.ingress[0].ip}')
  [ -n "$IP" ] && curl -s -m 5 -o /dev/null http://$IP/metrics && break
  sleep 10
done
[ -z "$IP" ] && die "no external IP for the service"
curl -s http://$IP/metrics | head -5
cd ~
echo ">>> Task 4 DONE - click 'Check my progress' (Deploy the application)"
done_ t4; fi

if step t5 "Task 5: log-based metric hello-app-error"; then
F='severity=ERROR resource.labels.container_name="hello-app"'
F="$F"' textPayload: "ERROR: 404 Error page not found"'
gcloud logging metrics describe hello-app-error >/dev/null 2>&1 || \
  gcloud logging metrics create hello-app-error \
  --description="hello-app 404 errors" --log-filter="$F" \
  || die "metric failed"
echo ">>> Task 5 DONE - click 'Check my progress' (Create the log-based metric)"
done_ t5; fi

if step t6 "Task 6: metrics-based alert 'log based metric alert'"; then
F6='metric.type=\"logging.googleapis.com/user/hello-app-error\"'
F6="$F6"' AND resource.type=\"k8s_container\"'
cat > metric_alert.json <<EOF
{
  "displayName": "log based metric alert",
  "combiner": "OR",
  "conditions": [ {
    "displayName": "hello-app-error above 0",
    "conditionThreshold": {
      "filter": "$F6",
      "aggregations": [ { "alignmentPeriod": "120s",
                          "perSeriesAligner": "ALIGN_SUM" } ],
      "comparison": "COMPARISON_GT",
      "thresholdValue": 0,
      "duration": "0s",
      "trigger": { "count": 1 }
    } } ],
  "notificationChannels": [ "$CH" ]
}
EOF
pol "log based metric alert" metric_alert.json
echo ">>> Task 6 DONE - click 'Check my progress' (Create the metrics-based alert)"
done_ t6; fi

if step t7 "Task 7: generate 404 errors (~3 min)"; then
gcloud container clusters get-credentials gmp-cluster --zone $ZONE >/dev/null 2>&1
IP=$(kubectl get services -n gmp-test \
  -o jsonpath='{.items[*].status.loadBalancer.ingress[0].ip}')
[ -z "$IP" ] && die "service IP not found"
timeout 180 bash -c -- "while true; do curl -s -o /dev/null http://$IP/error; \
  sleep \$((RANDOM % 4)); done"
echo "   errors sent; waiting 60s for logs to arrive"; sleep 60
echo ">>> Task 7 DONE - click 'Check my progress' (Generate some errors)"
echo "    (if grey, wait 1-2 min and click again)"
done_ t7; fi
echo; echo ">>> All tasks finished."
LABEOF
echo "SCRIPT SAVED OK - now run: bash gsp091_lab.sh"
```

bash gsp091_lab.sh
