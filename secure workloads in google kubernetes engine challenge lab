TASK 1--------------------------

gcloud container clusters create security-demo-cluster596 \
   --zone us-central1-c \
   --machine-type n1-standard-4 \
   --num-nodes 2 \
   --enable-network-policy
  
TASK 2--------------------------

gcloud sql instances create wordpress-db-273 --region us-central1

gcloud iam service-accounts create sa-wordpress-157

gcloud projects add-iam-policy-binding $DEVSHELL_PROJECT_ID \
   --member="serviceAccount:sa-wordpress-157@$DEVSHELL_PROJECT_ID.iam.gserviceaccount.com" \
   --role="roles/cloudsql.client"

gcloud iam service-accounts keys create key.json --iam-account=sa-wordpress-157@$DEVSHELL_PROJECT_ID.iam.gserviceaccount.com

kubectl create secret generic cloudsql-instance-credentials --from-file key.json

kubectl create secret generic cloudsql-db-credentials \
   --from-literal username=wordpress \
   --from-literal password=''
   
   
TASK 3--------------------------------

helm version
helm repo add stable https://charts.helm.sh/stable
helm repo update

helm install nginx-ingress stable/nginx-ingress --set rbac.create=true

kubectl get service nginx-ingress-controller

. add_ip.sh

kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v0.16.0/cert-manager.yaml

kubectl create clusterrolebinding cluster-admin-binding \
   --clusterrole=cluster-admin \
   --user=$(gcloud config get-value core/account)
   
   
task 4 -----------------------------------------

apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-nginx-access-to-internet
spec:
  podSelector:
    matchLabels:
      app: nginx-ingress
  policyTypes:
  - Ingress
  ingress:
  - {}
  
  
  kubectl apply -f network-policy.yaml
  
  
  TASK 5--------------------
  
  Name - us-central1-c.security-demo-cluster596
  
  TASK 6---------------------
  
  kubectl apply -f psp-restrictive.yaml
  kubectl apply -f psp-role.yaml
  kubectl apply -f psp-use.yaml
