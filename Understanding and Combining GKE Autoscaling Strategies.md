```/text
set +H
cat > gke_autoscale_lab.sh <<'LABEOF'
#!/bin/bash
# Understanding and Combining GKE Autoscaling Strategies (GSP768)
# Run: bash gke_autoscale_lab.sh  (1 question, then runs by itself ~25 min)
# It runs in the background: if Cloud Shell disconnects, reopen and run
#   tail -f ~/gsp768.log      to keep watching.
LOG=~/gsp768.log; ENVF=~/.gsp768.env; C=scaling-demo
ask() { local r; read -e -r -p "$2 [$3]: " r; printf -v "$1" '%s' "${r:-$3}"; }

if [ "$1" != "--bg" ]; then
  P=$(gcloud config get-value project 2>/dev/null)
  [ -z "$P" ] && { echo "run: gcloud config set project <Project ID>"; exit 1; }
  DZ=$(gcloud container clusters list --filter="name=$C" \
    --format="value(location)" 2>/dev/null | head -1)
  [ -z "$DZ" ] && DZ=$(gcloud compute project-info describe \
    --format="value(commonInstanceMetadata.items.google-compute-default-zone)" \
    2>/dev/null)
  OLDP=$(grep -s '^P=' $ENVF | cut -d= -f2)
  if [ "$OLDP" = "$P" ]; then
    ZONE=$(grep '^ZONE=' $ENVF | cut -d= -f2)
    echo "=== Resuming this lab (ZONE=$ZONE) - finished tasks are skipped ==="
  else
    echo "=== YELLOW value (gcloud config set compute/zone ...) ==="
    ask ZONE "ZONE" "${DZ:-us-central1-a}"
  fi
  echo "P=$P" > $ENVF; echo "ZONE=$ZONE" >> $ENVF
  if [ -f ~/.gsp768.pid ] && kill -0 "$(cat ~/.gsp768.pid)" 2>/dev/null; then
    echo "Already running - showing the log (Ctrl+C only stops watching)"
  else
    nohup bash "$0" --bg > $LOG 2>&1 &
    echo $! > ~/.gsp768.pid
    echo "Started in background. Ctrl+C only stops watching."
  fi
  sleep 1; tail -n +1 -f $LOG --pid="$(cat ~/.gsp768.pid)"
  exit 0
fi

# ===================== background part =====================
source $ENVF
gcloud config set project "$P" -q 2>/dev/null
gcloud config set compute/zone "$ZONE" -q 2>/dev/null
cd ~
D=~/.gsp768_done/$P; mkdir -p $D       # one marker file per finished task
step() { if [ -f $D/$1 ]; then echo; echo "=== $2: already done - skipped ===";
  return 1; fi; echo; echo "=== $2 ==="; }
done_() { touch $D/$1; }
retry() {  # retry <cmd...>  (cluster updates fail while another runs)
  for i in $(seq 1 15); do "$@" && return 0
    echo "   (busy, retry $i in 30s)"; sleep 30; done
  echo "STOP: '$*' failed - send Claude a screenshot"; exit 1
}

if step t0 "Task: Provision testing environment"; then
if gcloud container clusters describe $C --zone "$ZONE" >/dev/null 2>&1; then
  echo "   cluster $C already exists"
else
  retry gcloud container clusters create $C --zone "$ZONE" \
    --num-nodes=3 --enable-vertical-pod-autoscaling
fi
retry gcloud container clusters get-credentials $C --zone "$ZONE"
cat << EOF > php-apache.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: php-apache
spec:
  selector:
    matchLabels:
      run: php-apache
  replicas: 3
  template:
    metadata:
      labels:
        run: php-apache
    spec:
      containers:
      - name: php-apache
        image: k8s.gcr.io/hpa-example
        ports:
        - containerPort: 80
        resources:
          limits:
            cpu: 500m
          requests:
            cpu: 200m
---
apiVersion: v1
kind: Service
metadata:
  name: php-apache
  labels:
    run: php-apache
spec:
  ports:
  - port: 80
  selector:
    run: php-apache
EOF
kubectl apply -f php-apache.yaml
kubectl rollout status deployment php-apache --timeout=180s
echo ">>> DONE - click 'Check my progress' (Provision testing environment)"
done_ t0; fi
# kubectl needs the cluster login on every (re)start, even when t0 is skipped
retry gcloud container clusters get-credentials $C --zone "$ZONE" >/dev/null

if step t1 "Task 1: Horizontal Pod Autoscaling"; then
kubectl get hpa php-apache >/dev/null 2>&1 || \
  kubectl autoscale deployment php-apache --cpu-percent=50 --min=1 --max=10
kubectl get hpa
echo ">>> DONE - click 'Check my progress' (Scale pods with HPA)"
done_ t1; fi

if step t2 "Task 2: Vertical Pod Autoscaling"; then
gcloud container clusters describe $C --zone "$ZONE" \
  | grep ^verticalPodAutoscaling -A 1
kubectl get deployment hello-server >/dev/null 2>&1 || \
  kubectl create deployment hello-server \
    --image=gcr.io/google-samples/hello-app:1.0
kubectl set resources deployment hello-server --requests=cpu=450m
cat << EOF > hello-vpa.yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: hello-server-vpa
spec:
  targetRef:
    apiVersion: "apps/v1"
    kind: Deployment
    name: hello-server
  updatePolicy:
    updateMode: "Off"
EOF
kubectl apply -f hello-vpa.yaml
sleep 20
sed -i 's/Off/Auto/g' hello-vpa.yaml
kubectl apply -f hello-vpa.yaml
kubectl scale deployment hello-server --replicas=2
echo "   waiting up to 4 min for VPA to resize hello-server..."
for i in $(seq 1 16); do
  R=$(kubectl get pods -l app=hello-server \
    -o jsonpath='{.items[*].spec.containers[0].resources.requests.cpu}')
  echo "   cpu requests now: $R"
  echo "$R" | grep -q 450m || break
  sleep 15
done
if echo "$R" | grep -q 450m; then
  echo "   still 450m - setting 25m by hand (as the lab says)"
  kubectl set resources deployment hello-server --requests=cpu=25m
fi
kubectl get vpa hello-server-vpa
echo ">>> DONE - click 'Check my progress' (Scale size of pods with VPA)"
done_ t2; fi

if step t5 "Task 5: Cluster autoscaler"; then
retry gcloud beta container clusters update $C --zone "$ZONE" -q \
  --enable-autoscaling --min-nodes 1 --max-nodes 5
retry gcloud beta container clusters update $C --zone "$ZONE" -q \
  --autoscaling-profile optimize-utilization
pdb() { kubectl get pdb "$1" -n kube-system >/dev/null 2>&1 || \
  kubectl create poddisruptionbudget "$1" --namespace=kube-system \
    --selector "$2" --max-unavailable 1; }
pdb kube-dns-pdb k8s-app=kube-dns
pdb prometheus-pdb k8s-app=prometheus-to-sd
pdb kube-proxy-pdb component=kube-proxy
pdb metrics-agent-pdb k8s-app=gke-metrics-agent
pdb metrics-server-pdb k8s-app=metrics-server
pdb fluentd-pdb k8s-app=fluentd-gke
pdb backend-pdb k8s-app=glbc
pdb kube-dns-autoscaler-pdb k8s-app=kube-dns-autoscaler
pdb stackdriver-pdb app=stackdriver-metadata-agent
pdb event-pdb k8s-app=event-exporter
kubectl get nodes
echo ">>> DONE - click 'Check my progress' (Cluster autoscaler)"
done_ t5; fi

if step t6 "Task 6: Node Auto Provisioning"; then
retry gcloud container clusters update $C --zone "$ZONE" -q \
  --enable-autoprovisioning --min-cpu 1 --min-memory 2 \
  --max-cpu 45 --max-memory 160
echo ">>> DONE - click 'Check my progress' (Node Auto Provisioning)"
done_ t6; fi

if step t7 "Task 7: larger demand (load generator)"; then
kubectl delete pod load-generator --ignore-not-found >/dev/null
kubectl run load-generator --image=busybox --restart=Never -- \
  /bin/sh -c "while sleep 0.01; do wget -q -O- http://php-apache; done"
echo "   waiting for php-apache to reach 7 replicas (max 12 min)..."
for i in $(seq 1 48); do
  N=$(kubectl get deployment php-apache -o jsonpath='{.status.replicas}')
  kubectl get hpa php-apache --no-headers | sed 's/^/   /'
  [ "${N:-0}" -ge 7 ] && break
  sleep 15
done
kubectl get deployment php-apache
kubectl delete pod load-generator --ignore-not-found
echo "   load generator stopped"
done_ t7; fi

if step t8 "Task 8: Optimize larger loads (pause pod)"; then
cat << EOF > pause-pod.yaml
---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: overprovisioning
value: -1
globalDefault: false
description: "Priority class used by overprovisioning."
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: overprovisioning
  namespace: kube-system
spec:
  replicas: 1
  selector:
    matchLabels:
      run: overprovisioning
  template:
    metadata:
      labels:
        run: overprovisioning
    spec:
      priorityClassName: overprovisioning
      containers:
      - name: reserve-resources
        image: k8s.gcr.io/pause
        resources:
          requests:
            cpu: 1
            memory: 4Gi
EOF
kubectl apply -f pause-pod.yaml
sleep 60
kubectl get pods -n kube-system -l run=overprovisioning
kubectl get nodes
echo ">>> DONE - click 'Check my progress' (Optimize larger loads)"
done_ t8; fi
echo ">>> All done."
LABEOF
echo "SCRIPT SAVED OK - now run: bash gke_autoscale_lab.sh"
```

bash gke_autoscale_lab.sh
