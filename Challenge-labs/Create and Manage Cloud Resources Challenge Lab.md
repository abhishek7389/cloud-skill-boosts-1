########### Defining some variables given by Cloud Skill Boosts ###################
```
export VM_NAME=<vm_name_given_in_the_lab_instructions>
```
```
export PORT=<port_given_in_the_lab_instructions>
```
```
export FIREWALL_RULE_NAME=<firewall_rule_name_given_in_the_lab_instructions>
```

########### Task 1 ###################

```
gcloud compute instances create $VM_NAME \
--zone=us-east1-b \
--machine-type=f1-micro \
--image=projects/debian-cloud/global/images/debian-10-buster-v20220406 
```

########### Task 2 ###################
```
gcloud container clusters create nucleus-backend \
          --num-nodes 1 \
          --network nucleus-vpc \
          --region us-east1
gcloud container clusters get-credentials nucleus-backend \
          --region us-east1

kubectl create deployment hello-server \
          --image=gcr.io/google-samples/hello-app:2.0

kubectl expose deployment hello-server \
          --type=LoadBalancer \
          --port $PORT
```


########### Task 3 ###################
```
cat << EOF > startup.sh
#! /bin/bash
apt-get update
apt-get install -y nginx
service nginx start
sed -i -- 's/nginx/Google Cloud Platform - '"\$HOSTNAME"'/' /var/www/html/index.nginx-debian.html
EOF

```
```
gcloud compute instance-templates create web-server-template \
       --metadata-from-file startup-script=startup.sh \
       --network nucleus-vpc \
       --machine-type g1-small \
       --region us-east1
```
```
gcloud compute target-pools create nginx-pool
```
```
gcloud compute instance-groups managed create web-server-group \
       --base-instance-name web-server \
       --size 2 \
       --template web-server-template \
       --region us-east1

gcloud compute firewall-rules create $FIREWALL_RULE_NAME \
       --allow tcp:80 \
       --network nucleus-vpc
       
gcloud compute http-health-checks create http-basic-check
gcloud compute instance-groups managed \
       set-named-ports web-server-group \
       --named-ports http:80 \
       --region us-east1

gcloud compute backend-services create web-server-backend \
       --protocol HTTP \
       --http-health-checks http-basic-check \
       --global

gcloud compute backend-services add-backend web-server-backend \
       --instance-group web-server-group \
       --instance-group-region us-east1 \
       --global

gcloud compute url-maps create web-server-map \
       --default-service web-server-backend

gcloud compute target-http-proxies create http-lb-proxy \
       --url-map web-server-map

gcloud compute forwarding-rules create permit-tcp-rule-261 \
     --global \
     --target-http-proxy http-lb-proxy \
     --ports 80
```
```
gcloud compute forwarding-rules list
```
