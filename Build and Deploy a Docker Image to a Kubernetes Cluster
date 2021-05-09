Firstly clone this repository and unzip it - 

gsutil cp gs://sureskills-ql/challenge-labs/ch04-kubernetes-app-deployment/echo-web.tar.gz .

tar -xvf echo-web.tar.gz


2ND TASK ----------

gcloud builds submit --tag gcr.io/$DEVSHELL_PROJECT_ID/echo-app:v1 .


1ST TASK ----------

gcloud container clusters create echo-cluster --num-nodes 2 --zone us-central1-a --machine-type n1-standard-2


3RD TASK -----------

kubectl create deployment echo-web --image=gcr.io/qwiklabs-resources/echo-app:v1


4RTH TASK ----------


kubectl expose deployment echo-web --type=LoadBalancer --port=80 --target-port=8000



kubectl get svc
