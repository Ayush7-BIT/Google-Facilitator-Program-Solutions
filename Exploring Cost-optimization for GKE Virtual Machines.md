```/text
set +H
cat > gke_vm_lab.sh <<'LABEOF'
#!/bin/bash
# Exploring Cost-optimization for GKE Virtual Machines (GSP767)
# Run: bash gke_vm_lab.sh   (no questions, no ENTER presses, ~20 min)
die() { echo; echo "STOP: $1"; exit 1; }
P=$(gcloud config get-value project 2>/dev/null)
[ -z "$P" ] && die "run: gcloud config set project <Project ID>"
CL=hello-demo-cluster
ZONE=$(gcloud container clusters list --filter="name=$CL" --format="value(location)")
[ -n "$ZONE" ] || die "$CL not found yet - wait 2-5 min for the lab setup, then re-run"
REGION="${ZONE%-*}"
echo "Project $P | cluster $CL in $ZONE | region $REGION"
gcloud config set compute/zone "$ZONE" >/dev/null 2>&1

# ============ TASK 2: scale up + node pool migration ============
echo; echo "=== Task 2: scale hello-server to 2 ==="
gcloud container clusters get-credentials $CL --zone "$ZONE" || die "credentials"
kubectl scale deployment hello-server --replicas=2
echo ">>> 'Scale Up Hello App' DONE - click Check my progress"

if gcloud container node-pools describe my-node-pool --cluster $CL \
     --zone "$ZONE" >/dev/null 2>&1; then
  echo; echo "=== resize my-node-pool to 4 nodes ==="
  gcloud container clusters resize $CL --node-pool my-node-pool \
    --num-nodes 4 --zone "$ZONE" --quiet
fi
echo; echo "=== create larger-pool (e2-standard-2) ==="
gcloud container node-pools describe larger-pool --cluster=$CL \
  --zone="$ZONE" >/dev/null 2>&1 || \
gcloud container node-pools create larger-pool --cluster=$CL \
  --machine-type=e2-standard-2 --num-nodes=1 --zone="$ZONE" --quiet \
  || die "node pool failed"
echo ">>> 'Create node pool' DONE - click Check my progress"

if gcloud container node-pools describe my-node-pool --cluster $CL \
     --zone "$ZONE" >/dev/null 2>&1; then
  echo; echo "=== move pods off my-node-pool and delete it ==="
  for n in $(kubectl get nodes -l cloud.google.com/gke-nodepool=my-node-pool -o=name)
  do kubectl cordon "$n"; done
  for n in $(kubectl get nodes -l cloud.google.com/gke-nodepool=my-node-pool -o=name)
  do kubectl drain --force --ignore-daemonsets --delete-emptydir-data \
       --grace-period=10 --timeout=300s "$n"; done
  kubectl get pods -o=wide
  gcloud container node-pools delete my-node-pool --cluster $CL \
    --zone "$ZONE" --quiet
fi

# ============ TASK 3: regional cluster + pods ============
echo; echo "=== Task 3: regional cluster regional-demo in $REGION (5-10 min) ==="
gcloud container clusters describe regional-demo --region="$REGION" >/dev/null 2>&1 || \
  gcloud container clusters create regional-demo --region="$REGION" \
    --num-nodes=1 --quiet || die "regional cluster failed"
gcloud container clusters get-credentials regional-demo --region="$REGION"

cat > ~/pod-1.yaml <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: pod-1
  labels:
    security: demo
spec:
  containers:
  - name: container-1
    image: wbitt/network-multitool
EOF
cat > ~/pod-2.yaml <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: pod-2
spec:
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: security
            operator: In
            values:
            - demo
        topologyKey: "kubernetes.io/hostname"
  containers:
  - name: container-2
    image: us-docker.pkg.dev/google-samples/containers/gke/hello-app:1.0
EOF
kubectl apply -f ~/pod-1.yaml
kubectl get pod pod-2 >/dev/null 2>&1 || kubectl apply -f ~/pod-2.yaml
kubectl wait --for=condition=Ready pod/pod-1 pod/pod-2 --timeout=300s
kubectl get pod pod-1 pod-2 --output wide
echo ">>> 'Managing a Regional Cluster' DONE - click Check my progress"

# ============ flow logs + BigQuery sink (console steps of the lab) ============
echo; echo "=== VPC flow logs on the $REGION default subnet + sink to BigQuery ==="
gcloud services enable networkmanagement.googleapis.com >/dev/null 2>&1
# 1. dataset + sink FIRST, so no flow log is lost
bq --location=US show us_flow_logs >/dev/null 2>&1 || bq --location=US mk -d us_flow_logs
if ! gcloud logging sinks describe FlowLogsSample >/dev/null 2>&1; then
  gcloud logging sinks create FlowLogsSample \
    bigquery.googleapis.com/projects/$P/datasets/us_flow_logs \
    --log-filter='logName:"vpc_flows"' --quiet
fi
WI=$(gcloud logging sinks describe FlowLogsSample --format="value(writerIdentity)")
gcloud projects add-iam-policy-binding "$P" --member="$WI" \
  --role=roles/bigquery.dataEditor --condition=None --quiet >/dev/null 2>&1
# dataset-level WRITER for the sink as well
bq show --format=prettyjson $P:us_flow_logs > /tmp/ds.json
python3 -c '
import json, sys
d = json.load(open("/tmp/ds.json")); sa = sys.argv[1]
acc = d.setdefault("access", [])
if not any(a.get("userByEmail") == sa for a in acc):
    acc.append({"role": "WRITER", "userByEmail": sa})
json.dump(d, open("/tmp/ds.json", "w"))' "${WI#serviceAccount:}"
bq update --source /tmp/ds.json $P:us_flow_logs >/dev/null \
  && echo "   sink FlowLogsSample can write to us_flow_logs"
# 2. flow logs: every flow, 5 s aggregation, all metadata
gcloud compute networks subnets update default --region="$REGION" \
  --enable-flow-logs --logging-aggregation-interval=interval-5-sec \
  --logging-flow-sampling=1.0 --logging-metadata=include-all --quiet \
  || echo "   (flow logs: error above)"
echo "   waiting 60s for flow logging to start"; sleep 60

# 3. traffic: pod-1 -> pod-2 (different nodes) + node VM -> node VM
IP2=$(kubectl get pod pod-2 -o jsonpath='{.status.podIP}')
N1=$(kubectl get pod pod-1 -o jsonpath='{.spec.nodeName}')
N2=$(kubectl get pod pod-2 -o jsonpath='{.spec.nodeName}')
NIP2=$(kubectl get node "$N2" \
  -o jsonpath='{.status.addresses[?(@.type=="InternalIP")].address}')
echo "   ping pod-2 ($IP2) from pod-1 - different nodes:"
kubectl exec pod-1 -- ping -c 10 "$IP2" | tail -2
kubectl delete pod zone-pinger --ignore-not-found >/dev/null 2>&1
kubectl run zone-pinger --image=busybox --restart=Never \
  --overrides="{\"spec\":{\"hostNetwork\":true,\"nodeName\":\"$N1\"}}" \
  -- sh -c "ping -i 0.2 -c 2400 $NIP2 & ping -i 0.2 -c 2400 $IP2; wait"
echo "   node $N1 -> node $N2 ($NIP2) traffic running for ~8 min"
kubectl exec pod-1 -- ping -i 0.2 -c 300 "$IP2" >/dev/null 2>&1 &

# 4. wait until flow logs are really in BigQuery
echo "   waiting for flow-log rows in BigQuery (up to 15 min)..."
ROWS=0; T=""
for i in $(seq 1 45); do
  sleep 20
  T=$(bq ls --max_results=50 us_flow_logs 2>/dev/null \
    | awk '/vpc_flows/{print $1; exit}')
  [ -n "$T" ] && ROWS=$(bq query --use_legacy_sql=false --format=csv -q \
    "SELECT COUNT(*) FROM \`$P.us_flow_logs.$T\`" 2>/dev/null | tail -1)
  echo "   $(date +%H:%M:%S) table: ${T:-not yet}  rows: ${ROWS:-0}"
  [ "${ROWS:-0}" -gt 0 ] 2>/dev/null && break
done
if [ "${ROWS:-0}" -gt 0 ] 2>/dev/null; then
  # the lab's own query - the checker looks for it in the BigQuery job history
  Q="SELECT jsonPayload.src_instance.zone AS src_zone,"
  Q="$Q jsonPayload.src_instance.vm_name AS src_vm,"
  Q="$Q jsonPayload.dest_instance.zone AS dest_zone,"
  Q="$Q jsonPayload.dest_instance.vm_name FROM \`$P.us_flow_logs.$T\`"
  bq query --use_legacy_sql=false --max_rows=10 "$Q"
else
  echo "   (no rows yet - continue; if the check stays grey, wait and retry)"
fi

# ============ move pod-2 next to pod-1 ============
echo; echo "=== Simulate Traffic: recreate pod-2 with podAffinity ==="
sed -i 's/podAntiAffinity/podAffinity/g' ~/pod-2.yaml
kubectl delete pod pod-2 --wait=true
kubectl create -f ~/pod-2.yaml
kubectl wait --for=condition=Ready pod/pod-2 --timeout=300s
kubectl get pod pod-1 pod-2 --output wide
IP2=$(kubectl get pod pod-2 -o jsonpath='{.status.podIP}')
echo "   ping pod-2 ($IP2) from pod-1 - same node now:"
kubectl exec pod-1 -- ping -c 10 "$IP2" | tail -2
echo ">>> 'Simulate Traffic' DONE - click Check my progress"
echo "    (if grey: wait 2-3 min and click again - flow logs keep arriving)"
echo; echo ">>> All tasks finished."
LABEOF
echo "SCRIPT SAVED OK - now run: bash gke_vm_lab.sh"
```

bash gke_vm_lab.sh

