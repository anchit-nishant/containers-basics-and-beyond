# Lab : Deploying the Containerized Application
In this Lab, we will deploy and test the application manually step by step. This will help to figure out any issues.

* **Clone Repo**
```
cd ~/environment

if [ ! -d ~/environment/containers-basics-and-beyond ]; then
  git clone https://github.com/vijay-khanna/containers-basics-and-beyond
fi

cd ~/environment/containers-basics-and-beyond/
```


* **Enable SSM Permission to the Container Nodes**
The Application Secrets will be stored in Parameter Store, which will be securely accessed by the Node Application.

>Open one of the Worker Instance in EC2 Console, note the "IAM role" assigned to instance, Open the Role in IAM.
It will look something like "eksctl-EKS-Cluster-Istio-vk-nodeg-NodeInstanceRole-abcd1234", based on EKS Cluster name selected earlier. </br>

> Attach the below policies (These are for Test purpose only. For Production, give more restrictive access)</br>
>#**AmazonEC2RoleforSSM**</br>
>#**AmazonSSMFullAccess**</br>


Create Application Token on the below websites, and store the Token in SSM Store</br>
>https://darksky.net/dev</br>
>https://account.mapbox.com/</br>

Test the aws cli commands from Cloud9 Console

```
#//Check if the Secrets Exist, as required, then this step can be skipped. 
aws ssm get-parameters --names "/Params/keys/DarkSkyAPISecret"
aws ssm get-parameters --names "/Params/keys/MapBoxAccessToken"

#//Skip the below steps if Parameters in this Code-Block found if above step are fine
read -p "Enter the DarkSkyAPISecret : " DarkSkyAPISecret ; echo "DarkSkyAPISecret :  "$DarkSkyAPISecret

aws ssm put-parameter --name "/Params/keys/DarkSkyAPISecret" --value $DarkSkyAPISecret --type String --overwrite

read -p "Enter the MapBoxAccessToken : " MapBoxAccessToken ; echo "MapBoxAccessToken :  "$MapBoxAccessToken

aws ssm put-parameter --name "/Params/keys/MapBoxAccessToken" --value $MapBoxAccessToken --type String --overwrite

#//To Verify the Successfull entry to SSM Store
aws ssm get-parameters --names "/Params/keys/DarkSkyAPISecret"
aws ssm get-parameters --names "/Params/keys/MapBoxAccessToken"

```
</br>
Optionally Test the above commands from the Worker Nodes. Use SSH Key created earlier.

</br>

* **Deploy the Service, so it is ready by the time Deployment finishes**
```
kubectl apply -f ~/environment/containers-basics-and-beyond/app-one/ver1/front-end/service-front-end.yaml
kubectl apply -f ~/environment/containers-basics-and-beyond/app-one/ver1/backend-pi-array/service-back-end-pi-array.yaml
kubectl apply -f ~/environment/containers-basics-and-beyond/app-one/ver1/backend-motm/service-back-end-motm.yaml
```
</br>

* **Update the Load Balancer for front end Service in client js file, and Optionally a Route53 Entry**
```
#//update the value of field : frontEndDNSURLandPort with the LB-URL or DNS A-Record. If it is a Route53 Entry, then create 'A' record and point to LB.

front_end_lb=$(kubectl get svc front-end-service | grep front-end-service | awk '{print $4}') ; echo $front_end_lb
sed -i "s|LBorDNSURL|$front_end_lb|g"  ~/environment/containers-basics-and-beyond/app-one/ver1/front-end/public/js/app-client-script.js


#//To Check frontEndDNSURLandPort value
#//Check if the Output of below two commands mentions the same LoadBalancer End Point

echo $front_end_lb
cat ~/environment/containers-basics-and-beyond/app-one/ver1/front-end/public/js/app-client-script.js | grep 'const frontEndDNSURLandPort' 


#//backend-service-edit
backend_end_motd__lb=$(kubectl get svc back-end-motm-service | grep back-end-motm-service | awk '{print $4}') 
echo $backend_end_motd__lb
sed -i "s|MOTMLBURL|$backend_end_motd__lb|g"  ~/environment/containers-basics-and-beyond/app-one/ver1/front-end/src/utils/forecast.js


#//To Check frontEndDNSURLandPort value
#//Check if the Output of below two commands mentions the same LoadBalancer End Point

echo $backend_end_motd__lb
cat ~/environment/containers-basics-and-beyond/app-one/ver1/front-end/src/utils/forecast.js | grep 'var urlMotm'

```

* **creating Container from Dockerfile, and saving to ECR Repo in own account**

>#**FrontEnd Service**</br>
```
cd ~/environment/containers-basics-and-beyond/app-one/ver1/front-end/      

#//Below command will Login to ECR Repo

aws ecr get-login --region us-east-1 --no-include-email  


#//Copy-Paste the Console output of above command output into the console to login. 
#//Starting with "docker login.... Ending with amazonaws.com". 
#//You must get "Login Succeeded" message to proceed.

frontEndRepoECR=$(echo $frontEndRepoECRURI | awk -F'/' '{print $2}')
echo $frontEndRepoECR

#---Copy-Paste the below "IF - FI " Code Block, to create a new ECR Repo if required.

if [ -z "$frontEndRepoECR" ] 
then 
  clear
  echo "\$frontEndRepoECR is empty, creating the ECR Repo" 
  # aws ecr delete-repository --repository-name $frontEndRepoECR --force
  frontEndRepoECRURI=$(aws ecr create-repository --repository-name ${EKS_CLUSTER_NAME,,}_front_end_app_one_ver1 | jq -r  '.repository.repositoryUri')
  echo "export frontEndRepoECRURI=${frontEndRepoECRURI}" >> ~/.bash_profile
  echo $frontEndRepoECRURI
else 
  clear
  echo "\$frontEndRepoECR is NOT empty, Will skip re-creating the ECR Repo" 
  echo $frontEndRepoECRURI
fi

#---

#//The below command will create Container image from DockerFile. 
#//This takes 5-7 minutes. 
#//Red text message for gpg key is normal, and are not errors.


docker build -t app-one-ver1-front-end:v1 .   


docker images  | grep app-one-ver1-front-end    
frontEndImageId=$(docker images app-one-ver1-front-end:v1 | grep app-one-ver1-front-end | awk '{print $3}') ; echo $frontEndImageId   
echo "export frontEndRepoECRURI=${frontEndRepoECRURI}" >> ~/.bash_profile
docker tag $frontEndImageId $frontEndRepoECRURI
docker push $frontEndRepoECRURI  
```
</br>

>#**Backend Pi-Array Service**</br>
```
cd ~/environment/containers-basics-and-beyond/app-one/ver1/backend-pi-array/       


echo $backEndPiArrayRepoECRURI  

#---Copy-Paste the below "IF - FI " Code Block, to create a new Backend Pi array ECR Repo if required.

if [ -z "$backEndPiArrayRepoECRURI" ] 
then 
  clear
  echo "\$backEndPiArrayRepoECRURI is empty, will create the ECR Repo" 
  # backEndPiArrayRepoECR=$(echo $backEndPiArrayRepoECRURI  | awk -F'/' '{print $2}') ; echo $backEndPiArrayRepoECR
  # aws ecr delete-repository --repository-name $backEndPiArrayRepoECR --force

  backEndPiArrayRepoECRURI=$(aws ecr create-repository --repository-name ${EKS_CLUSTER_NAME,,}_back_end_pi_array_app_one_ver1 | jq -r  '.repository.repositoryUri')
  echo "export backEndPiArrayRepoECRURI=${backEndPiArrayRepoECRURI}" >> ~/.bash_profile
  echo $backEndPiArrayRepoECRURI 
else 
  clear
  echo "\$backEndPiArrayRepoECRURI is NOT empty, Will skip re-creating the ECR Repo, "
  echo $backEndPiArrayRepoECRURI  
fi

#---



#//The below command will create Container image from DockerFile. 
#//This takes 5-7 minutes. 
#//Red text message for gpg key is normal, and not errors.

docker build -t app-one-ver1-back-end-pi-array:v1 .


docker images  | grep app-one-ver1-back-end-pi-array

backEndPiArrayImageId=$(docker images app-one-ver1-back-end-pi-array:v1 | grep app-one-ver1-back-end-pi-array    | awk '{print $3}') ; echo $backEndPiArrayImageId   
docker tag $backEndPiArrayImageId $backEndPiArrayRepoECRURI
docker push $backEndPiArrayRepoECRURI
```
</br>

</br>

>#**Backend Message of the Moment Service**</br>
```
cd ~/environment/containers-basics-and-beyond/app-one/ver1/backend-motm/   

echo $backEndmotmRepoECRURI

#---Copy-Paste the below "IF - FI " Code Block, to create a new Backend motm - Message of Moment ECR Repo if required.

if [ -z "$backEndmotmRepoECRURI" ] 
then 
  clear
  echo "\$backEndmotmRepoECRURI is empty, create the ECR Repo" 
  # backEndmotmRepoECR=$(echo $backEndmotmRepoECRURI  | awk -F'/' '{print $2}') ; echo $backEndmotmRepoECR
  # aws ecr delete-repository --repository-name $backEndmotmRepoECR --force
  backEndmotmRepoECRURI=$(aws ecr create-repository --repository-name ${EKS_CLUSTER_NAME,,}_back_end_motm_app_one_ver1 | jq -r  '.repository.repositoryUri')
  echo $backEndmotmRepoECRURI  
  echo "export backEndmotmRepoECRURI =${backEndmotmRepoECRURI}" >> ~/.bash_profile
  echo $backEndmotmRepoECRURI  
else 
  clear
  echo "\$backEndmotmRepoECRURI is NOT empty, Will skip re-creating the ECR Repo, "
  echo $backEndmotmRepoECRURI  
fi

#---



#//The below command will create Container image from DockerFile. 
#//This takes 5-7 minutes. 
#//Red text message for gpg key is normal, and not errors.



#// version-1 motm
cd ~/environment/containers-basics-and-beyond/app-one/ver1/backend-motm/v1/
docker build -t app-one-ver1-back-end-motm:v1 .


docker images  | grep app-one-ver1-back-end-motm | grep v1
backEndmotmImageId=$(docker images app-one-ver1-back-end-motm:v1 | grep app-one-ver1-back-end-motm | awk '{print $3}') ; echo $backEndmotmImageId   
docker tag $backEndmotmImageId $backEndmotmRepoECRURI:v1
docker push $backEndmotmRepoECRURI:v1



#// version-2 motm
cd ~/environment/containers-basics-and-beyond/app-one/ver1/backend-motm/v2/
docker build -t app-one-ver1-back-end-motm:v2 .


docker images  | grep app-one-ver1-back-end-motm | grep v2   
backEndmotmImageId=$(docker images app-one-ver1-back-end-motm:v2 | grep app-one-ver1-back-end-motm | grep v2  | awk '{print $3}') ; echo $backEndmotmImageId   
docker tag $backEndmotmImageId $backEndmotmRepoECRURI:v2
docker push $backEndmotmRepoECRURI:v2


#// version-3 motm
cd ~/environment/containers-basics-and-beyond/app-one/ver1/backend-motm/v3/
docker build -t app-one-ver1-back-end-motm:v3 .


docker images  | grep app-one-ver1-back-end-motm | grep v3   
backEndmotmImageId=$(docker images app-one-ver1-back-end-motm:v3 | grep app-one-ver1-back-end-motm | grep v3  | awk '{print $3}') ; echo $backEndmotmImageId   
docker tag $backEndmotmImageId $backEndmotmRepoECRURI:v3
docker push $backEndmotmRepoECRURI:v3



```
</br>


* **Updating deployment files with ECR Link to Container Images**
```
#//replacing the IMAGE_URL with appropriate ECR Repo Locations for Deployment.Yaml Files
cp ~/environment/containers-basics-and-beyond/app-one/ver1/front-end/deployment-front-end.yaml /tmp/deployment-front-end.yaml
sed -i "s|IMAGE_URL|$frontEndRepoECRURI|g" /tmp/deployment-front-end.yaml
cat /tmp/deployment-front-end.yaml


cp ~/environment/containers-basics-and-beyond/app-one/ver1/backend-pi-array/deployment-back-end-pi-array.yaml  /tmp/deployment-back-end-pi-array.yaml
sed -i "s|IMAGE_URL|$backEndPiArrayRepoECRURI|g" /tmp/deployment-back-end-pi-array.yaml
cat /tmp/deployment-back-end-pi-array.yaml

cp ~/environment/containers-basics-and-beyond/app-one/ver1/backend-motm/v1/deployment-back-end-motm-v1.yaml /tmp/deployment-back-end-motm-v1.yaml
sed -i "s|IMAGE_URL|$backEndmotmRepoECRURI|g" /tmp/deployment-back-end-motm-v1.yaml
cat /tmp/deployment-back-end-motm-v1.yaml


cp ~/environment/containers-basics-and-beyond/app-one/ver1/backend-motm/v2/deployment-back-end-motm-v2.yaml /tmp/deployment-back-end-motm-v2.yaml
sed -i "s|IMAGE_URL|$backEndmotmRepoECRURI|g" /tmp/deployment-back-end-motm-v2.yaml
cat /tmp/deployment-back-end-motm-v2.yaml

cp ~/environment/containers-basics-and-beyond/app-one/ver1/backend-motm/v3/deployment-back-end-motm-v3.yaml /tmp/deployment-back-end-motm-v3.yaml
sed -i "s|IMAGE_URL|$backEndmotmRepoECRURI|g" /tmp/deployment-back-end-motm-v3.yaml
cat /tmp/deployment-back-end-motm-v3.yaml


```
</br>

* **To Deploy Containers and Service using kubectl**

```
kubectl apply -f /tmp/deployment-back-end-pi-array.yaml
#//wait few seconds, after deploying backend. and then deploy front end..else there might be errors to fetch from backend.


kubectl apply -f /tmp/deployment-back-end-motm-v1.yaml
kubectl apply -f /tmp/deployment-back-end-motm-v2.yaml
kubectl apply -f /tmp/deployment-back-end-motm-v3.yaml

kubectl get svc,deploy,pods

kubectl apply -f /tmp/deployment-front-end.yaml


kubectl get svc,deploy,pods


```
</br>

:warning:  :fire:  :triangular_flag_on_post:  :hand:
</br>
* **To Delete all Deployments, Services Created in this Lab. This will retain all other elements**
```
kubectl delete -f /tmp/deployment-front-end.yaml              
kubectl delete -f /tmp/deployment-back-end-pi-array.yaml
kubectl delete -f /tmp/deployment-back-end-motm-v1.yaml
kubectl delete -f /tmp/deployment-back-end-motm-v2.yaml
kubectl delete -f /tmp/deployment-back-end-motm-v3.yaml

kubectl delete -f ~/environment/containers-basics-and-beyond/app-one/ver1/front-end/service-front-end.yaml
kubectl delete -f ~/environment/containers-basics-and-beyond/app-one/ver1/backend-motm/service-back-end-motm.yaml
kubectl delete -f ~/environment/containers-basics-and-beyond/app-one/ver1/backend-pi-array/service-back-end-pi-array.yaml

kubectl get svc,deploy,pods

```

</br>



* **check the frontEndDNSURLandPort URL in Browser Window, to check http access**
</br>

```
curl http://$front_end_lb
```
