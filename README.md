# circe-kubernetes
Deploy [circe-angular](https://github.com/Kevin-Vu/circe-angular) and [circe-spring-boot](https://github.com/Kevin-Vu/circe-spring-boot) with GKE.  
All of these commands are performed in the Google Cloud Console which already has Maven, Docker and Git installed.

## Prerequisites
- Kubernetes cluster on GKE

## Deploy Postgres on Kubernetes
1. Create a Configmap for postgres
```
$ kubectl create -f postgres/postgres-configmap.yml
```

2. Create a PersistentVolumeClaim so the data get persisted even if the database is shut down
```
$ kubectl create -f postgres/postgres-pv-claim.yml
```

3. Create a Deployment for postgres
```
$ kubectl create -f postgres/postgres-database.yml
$ kubectl get pods # to see the created pods for postgres
```

4. Create a Service to internally expose postgres
```
$ kubectl create -f postgres/postgres-service.yml
$ kubectl get svc # to see the postgres service
```

Note : all the postgres yml files can be merged into one single file

## Deploy Spring Boot on Kubernetes
1. Package the Spring-Boot project
```
$ git clone https://github.com/Kevin-Vu/circe-spring-boot.git
$ mv -f circe-spring-boot/src/main/resources/{application-prod,application}.properties
$ mvn -DskipTests -f circe-spring-boot/pom.xml package
```

2. Enable Google Container Registry to store the docker image you will create
```
$ gcloud services enable containerregistry.googleapis.com
```

3. Create the docker image and push it
```
$ mvn -DskipTests com.google.cloud.tools:jib-maven-plugin:build -Dimage=gcr.io/$GOOGLE_CLOUD_PROJECT/circe-spring:v1
```
The image is available in your Google Cloud Platform, in the **Container Registry** section.

4. Create a Configmap with the hostname of postgres
```
$ kubectl create configmap hostname-config --from-literal=postgres_host=$(kubectl get svc postgres -o jsonpath="{.spec.clusterIP}")
```

5. Create the Deployment for spring-boot  
Replace `<SOMETHING>` in `deployments/circe-spring-boot-deployment.yml` by the value of `$GOOGLE_CLOUD_PROJECT`.
```
$ kubectl create -f deployments/circe-spring-boot-deployment.yml
$ kubectl get pods # to see the created pods for postgres
$ kubectl logs -f <FULLNAME OF THE SPRING POD> # to see if the spring starts well
```
Note : in this deployment yml we specify only one replica, you can set it (now or later) to more to scale your application

6. Create a Service to internally expose spring-boot
```
$ kubectl create -f services/circe-spring-boot-service.yml
```

## Deploy Angular on Kubernetes
1. Create the docker image for angular and push it
```
$ git clone https://github.com/Kevin-Vu/circe-angular.git
$ cd circe-angular && docker build -t circe-angular . && cd .. 
$ docker tag circe-angular gcr.io/$GOOGLE_CLOUD_PROJECT/circe-angular:v1
$ docker push gcr.io/$GOOGLE_CLOUD_PROJECT/circe-angular:v1
```
Note : there is a more elegant way to perform the build using -f...

2. Create the Deployment for angular  
Replace `<SOMETHING>` in `deployments/circe-angular-deployment.yml` by the value of `$GOOGLE_CLOUD_PROJECT`.
```
$ kubectl create -f deployments/circe-angular-deployment.yml
```

3. Create the Service to internally expose angular
```
$ kubectl create -f services/circe-angular-service.yml
```

## Get a domain name with an IP address
1. Buy a domain name at [Google Domain](https://domains.google.com/m/registrar/)

2. Get a static ip address
```
$ gcloud compute addresses create circe-ip --global
$ gcloud compute addresses describe circe-ip --global # Note the ip address
```

3. Follow the [Google Cloud tutorial](https://cloud.google.com/dns/docs/quickstart?hl=en) to register your domain with an IP address   
After finishing it, go to sleep because it takes about one day to two for the IP to get fixed to the domain name

4. Test that your domain has your static ip with a ping

## Expose your app to the world with HTTPS
1. If you have Kubernetes 1.15 or above you should be able to perform this command 
```
$ kubectl get managedcertificates # it should return something like 'No resource found'
```
**ONLY** If not, install CRD
```
$ git clone https://github.com/GoogleCloudPlatform/gke-managed-certs.git
$ kubectl apply -f gke-managed-certs/deploy/managedcertificates-crd.yaml
$ kubectl apply -f gke-managed-certs/deploy/managed-certificate-controller.yaml
$ kubectl get managedcertificates # it should return something like 'No ressource found'
```

2. Generate a certificate for your domains  
Add all your domains name in `ingress/circe-gke-cert.yml`
```
$ kubectl create -f ingress/circe-gke-cert.yml
```
Note : we use the GKE to generate a certificate, there are other ways to do that with Let's encrypt and Nginx.  
GKE is by far the easiest way.

3. Create the Ingress
```
$ kubectl create -f ingress/circe-ingress.yml
```

4. Open a second console  
- On the first one
```
$ watch kubectl describe managedcertificate circe-gke-cert
```
- On the second one
```
$ watch kubectl describe ing circe-ingress
```

Wait until in the first console **PROVISIONING** become **ACTIVE** and in the second console **UNKNOWN** become **HEALTHY**

5. Access your application with your favorite browser using your domaine name.  
Here is the final result with the domain **circe.live**
<img src="desktop.png" width="750">
