# AKS-Workshop-TR
Metehan Küçükçakır
## Giriş ve Açıklama
### Fruit Smoothies isimli bir şirkette IT çalışanı olduğunuzu ve yazılımcıların ürünlerin puanlanabileceği bir yazılım geliştirdiğini düşünün. Geliştirilen uygulama birden fazla bileşen içeriyor ve bunların Azure Kubernetes Service üzerinde host edilmesi planlanıyor.
### Uygulama bir adet frontend, bir adet database ve bir adet RESTful API'dan oluşuyor.
## İçerik
### Bu modülde multi-container bir uygulamayı Azure Kubernetes Service(AKS)'e deploy edeceğiz. Sırasıyla uygulayacağımız adımlar aşağıdaki gibidir;
#### -AKS Cluster'ının ıluşturulması
#### -Pod'ların deploymentı için en uygun yolun seçilmesi
#### -Podların internal ve external olarak yayınlanması
#### -SSL/TLS sertifikalarının tanımlanması
#### -AKS üzerinde çalışan bir uygulamanın manuel ya da otomatik olarak scale edilmesi
## Ön Gereksinimler
#### Kubernetes ve konseptlerine aşınalık
#### Kaynakları deploy edebilmek için bir Azure subscriptionu
#### Azure Cloud Shell
## Altyapı Mimarisi
### Hedef: MongoDB, web ve api servislerimizi üzerinde çalıştırabileceğimiz bir AKS Cluster'ı oluşturmak.
#### (Mimari için ana dizindeki "mimari.png" incelenmeli)
##### 1- Kubernetes Cluster olarak AKS kullanılacak
##### 2- Container imajlarını depolamak için Azure Container Registry kullanılacak
##### 3- Uygulamaya ilişkin üç bileşenin deploy edilmesi
###### 3-1 Fruit Smoothies website'ı için gerekli database helm ile deploy edilecek
###### 3-2 Fruit Smoothies website'ının RESTFul API'ı deploy edilecek
###### 3-3 Fruit Smoothies website'ının frontendi deploy edilecek
##### 4- AKS Ingress'i deploy edilecek
##### 5- Cert-manager kullanılarak SSL/TLS konfigurasyonu yapılacak
##### 6- Autoscaler ve Horizontal pod autoscaler aktif edilecek
## Kaynak Kod
### ratings-api
#### https://github.com/MicrosoftDocs/mslearn-aks-workshop-ratings-api
### ratings-web
#### https://github.com/MicrosoftDocs/mslearn-aks-workshop-ratings-web
## 1- AKS Kurulumu
#### Geliştirici ekip uygulamalar halihazırda container yapısı kullanıyor. Süreçleri hızlandırmak, yönetmek ve esnek bir şekilde büyüyüp küçülmesini sağlamak için AKS kullanılması isteniyor. Bu bölümde;
##### Resource Group oluşturulacak
##### Cluster networkü şekillendirilecek
##### AKS clusterı oluşturulacak
##### AKS clusterına bağlanılacak
##### Namespace oluşturulacak
### Resource Group Oluşturulması
#### 1- Windows terminal aracılığıyla Cloud Shell'e bağlanmak
#### 2- Değişkenlerin tanımlanması
```
REGION_NAME=westeurope
```
```
RESOURCE_GROUP=appmo-demo-rg
```
```
SUBNET_NAME=aks-subnet
```
```
VNET_NAME=aks-vnet
```
#### 3- Resource Group'un oluşturulması
```
az group create --name $RESOURCE_GROUP --location $REGION_NAME
```
### Networkun oluşturulması
#### Kubenet Network nedir?
Kubenet, kubernetes'in default network modelidir. Kubenet network ile AKS nodeları Azure Virtual Network subnetinden bir IP adresi alırlar. Podlar AKS içinde oluşan farklı bir networkten ip alırlar ve AKS dışındaki kaynaklara erişirken NAT ile erişirler.
#### CNI Network nedir?
CNI, AKS clusterının doğrudan Azure Virtual Network'e bağlı olduğu, her podun doğrudan vnetten bir ip aldığı yapıdır. Kubenet'e göre daha gelişmiş, daha yönetilebilir bir network yapısıdır.

##### Virtual Network oluşturma
```
az network vnet create \
    --resource-group $RESOURCE_GROUP \
    --location $REGION_NAME \
    --name $VNET_NAME \
    --address-prefixes 10.0.0.0/8 \
    --subnet-name $SUBNET_NAME \
    --subnet-prefixes 10.240.0.0/16
 ```
 ##### Subnet ID'sini öğrenme
```SUBNET_ID=$(az network vnet subnet show \
    --resource-group $RESOURCE_GROUP \
    --vnet-name $VNET_NAME \
    --name $SUBNET_NAME \
    --query id -o tsv)
```
### AKS Clusterının oluşturulması
#### Kubernetes versiyon kontrolü
```
VERSION=$(az aks get-versions \
    --location $REGION_NAME \
    --query 'orchestrators[?!isPreview] | [-1].orchestratorVersion' \
    --output tsv)
```
#### Cluster isminin tanımlanması
```
AKS_CLUSTER_NAME=aksworkshop-$RANDOM
```
#### Cluster isim kontrolü
```
echo $AKS_CLUSTER_NAME
```
#### Clusterın oluşturulması
```
az aks create \
--resource-group $RESOURCE_GROUP \
--name $AKS_CLUSTER_NAME \
--vm-set-type VirtualMachineScaleSets \
--node-count 2 \
--load-balancer-sku standard \
--location $REGION_NAME \
--kubernetes-version $VERSION \
--network-plugin azure \
--vnet-subnet-id $SUBNET_ID \
--service-cidr 10.2.0.0/24 \
--dns-service-ip 10.2.0.10 \
--docker-bridge-address 172.17.0.1/16 \
--generate-ssh-keys
```
### Clustera bağlanma
#### Bağlantı bilgilerini tanımlama
```
az aks get-credentials \
    --resource-group $RESOURCE_GROUP \
    --name $AKS_CLUSTER_NAME
```
#### Bağlantı kontrolü
```
kubectl get nodes
```
### Namespace oluşturma
```
kubectl get namespace
```
```
kubectl create namespace ratingsapp
```
## 2- Container Registry Oluşturulması
#### ACR ismi oluşturma
```
ACR_NAME=acr$RANDOM
```
#### ACR oluşturma
```
az acr create \
    --resource-group $RESOURCE_GROUP \
    --location $REGION_NAME \
    --name $ACR_NAME \
    --sku Standard
```
#### Uygulamaların ACR ile build edilmesi
##### API
```
git clone https://github.com/MicrosoftDocs/mslearn-aks-workshop-ratings-api.git
```
```
cd mslearn-aks-workshop-ratings-api
```
```
az acr build \
    --resource-group $RESOURCE_GROUP \
    --registry $ACR_NAME \
    --image ratings-api:v1 .
```
##### WEB
```
cd ~
```
```
git clone https://github.com/MicrosoftDocs/mslearn-aks-workshop-ratings-web.git
```
```
cd mslearn-aks-workshop-ratings-web
```
```
az acr build \
    --resource-group $RESOURCE_GROUP \
    --registry $ACR_NAME \
    --image ratings-web:v1 .
```
##### İmajların kontrolü
```
az acr repository list \
    --name $ACR_NAME \
    --output table
```
#### AKS-ACR arasındaki erişimin tanımlanması
```
az aks update \
    --name $AKS_CLUSTER_NAME \
    --resource-group $RESOURCE_GROUP \
    --attach-acr $ACR_NAME
```
## MongoDB'nin Oluşturulması
### Deployment
```
helm repo add bitnami https://charts.bitnami.com/bitnami
```
```
helm search repo bitnami
```
```
helm install ratings bitnami/mongodb \
    --namespace ratingsapp \
    --set auth.username=<username>,auth.password=<password>,auth.database=ratingsdb
```
### Kubernetes secreti
```
kubectl create secret generic mongosecret \
    --namespace ratingsapp \
    --from-literal=MONGOCONNECTION="mongodb://<username>:<password>@ratings-mongodb.ratingsapp:27017/ratingsdb"
```
## API'ın Deploy Edilmesi
### Deployment yaml dosyasının oluşturulması
```
nano ratings-api-deployment.yaml
```  
[ratings-api-deployment.yaml](https://github.com/metehankucukcakir/AKS-Workshop-TR/blob/master/Yaml%20Files/ratings-api-deployment.yaml)
### Deployment
```kubectl apply \
    --namespace ratingsapp \
    -f ratings-api-deployment.yaml
```
### Kontrol
```
kubectl get pods \
    --namespace ratingsapp \
    -l app=ratings-api -w
```
### Service oluşturulması
```
nano ratings-api-service.yaml
```    
[ratings-api-service.yaml](https://github.com/metehankucukcakir/AKS-Workshop-TR/blob/master/Yaml%20Files/ratings-api-service.yaml)
### Deployment
```kubectl apply \
    --namespace ratingsapp \
    -f ratings-api-service.yaml
```
### Kontrol
```
kubectl get service ratings-api --namespace ratingsapp
```
```    
kubectl get endpoints ratings-api --namespace ratingsapp
```
## WEB'in Deploy Edilmesi
### Deployment yaml dosyasının oluşturulması
```
nano ratings-web-deployment.yaml
```  
[ratings-web-deployment.yaml](https://github.com/metehankucukcakir/AKS-Workshop-TR/blob/master/Yaml%20Files/ratings-web-deployment.yaml)
### Deployment
```
kubectl apply \
    --namespace ratingsapp \
    -f ratings-web-deployment.yaml
```
### Kontrol
```
kubectl get pods \
    --namespace ratingsapp \
    -l app=ratings-web-deployment -w
```
### Service oluşturulması
```
nano ratings-web-service.yaml
```    
[ratings-web-service.yaml](https://github.com/metehankucukcakir/AKS-Workshop-TR/blob/master/Yaml%20Files/ratings-web-service.yaml)
### Deployment
```
kubectl apply \
    --namespace ratingsapp \
    -f ratings-web-service.yaml
```
### Kontrol
```
kubectl get service ratings-web --namespace ratingsapp
```
```
kubectl get endpoints ratings-web --namespace ratingsapp
```
### Uygulamanın Test Edilmesi
ratings-web isimli servisin public IP'sine doğrudan gidilir.

## 4- Ingress Deploymentı
### Namespace Oluşturulması
```
kubectl create namespace ingress
```
### Helm client eklenmesi
```
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
```
```
helm repo update
```
### Nginx kurulumu
```
helm install nginx-ingress ingress-nginx/ingress-nginx \
    --namespace ingress \
    --set controller.replicaCount=2 \
    --set controller.nodeSelector."beta\.kubernetes\.io/os"=linux \
    --set defaultBackend.nodeSelector."beta\.kubernetes\.io/os"=linux
```
### Public IP adresi
```
kubectl get services --namespace ingress -w
```
### Web servisinin tekrar düzenlenmesi
```
nano ratings-web-service.yaml
```
```
kubectl delete service \
    --namespace ratingsapp \
    ratings-web
```
[ratings-web-service.yaml](https://github.com/metehankucukcakir/AKS-Workshop-TR/blob/master/Yaml%20Files/updated-ratings-web-service.yaml)
```
kubectl apply \
    --namespace ratingsapp \
    -f ratings-web-service.yaml
``` 
### Web servisi için ingress oluşturulması
```
nano ratings-web-ingress.yaml
```    
[ratings-web-ingress.yaml](https://github.com/metehankucukcakir/AKS-Workshop-TR/blob/master/Yaml%20Files/ratings-web-ingress.yaml)
``` 
kubectl apply \
    --namespace ratingsapp \
    -f ratings-web-ingress.yaml \
    --validate=false
```
### Ingressin test edilmesi
http://frontend.13.68.177.68.nip.io

## 5- TLS/SSL Konfigürasyonu
### Cert-manager oluşturulması
```
kubectl create namespace cert-manager
```
```
helm repo add jetstack https://charts.jetstack.io
```
```
helm repo update
```
```
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.7.2/cert-manager.crds.yaml
```
```
helm install cert-manager \
    --namespace cert-manager \
    --version v1.7.2 \
    jetstack/cert-manager
```
#### Lets Encrypt için ClusterIssuer oluşturulması
```
nano cluster-issuer.yaml
```    
[cluster-issuer.yaml](https://github.com/metehankucukcakir/AKS-Workshop-TR/blob/master/Yaml%20Files/cluster-issuer.yaml)
```
kubectl apply \
    --namespace ratingsapp \
    -f cluster-issuer.yaml
```
#### Web servisi için SSL/TLS'i aktif etme
```
nano ratings-web-ingress.yaml
```
[ratings-web-ingress.yaml](https://github.com/metehankucukcakir/AKS-Workshop-TR/blob/master/Yaml%20Files/updated-ratings-web-ingress.yaml)
```    
kubectl apply \
    --namespace ratingsapp \
    -f ratings-web-ingress.yaml
```
```
kubectl describe cert ratings-web-cert --namespace ratingsapp
```
### Test
https://frontend.13.68.177.68.nip.io

## 6- Scale
#### Horizontal pod autoscaler
Horizontal pod autoscaler (HPA) denetleyicisi, Kubernetes denetleyici yöneticisinin bir HorizontalPodAutoscaler tanımında belirtilen metriklere göre kaynak kullanımını sorgulamasına olanak tanıyan Kubernetes kontrol döngüsüdür. 
    
HPA, hesaplanan değere göre pod sayısını otomatik olarak yukarı veya aşağı ölçeklendirir.

### HPA oluşturma

```
nano ratings-api-hpa.yaml
```    
[ratings-api-hpa.yaml](https://github.com/metehankucukcakir/AKS-Workshop-TR/blob/master/Yaml%20Files/ratings-api-hpa.yaml)
  
### Yük testi
```
az container create \
    -g $RESOURCE_GROUP \
    -n loadtest \
    --cpu 4 \
    --memory 1 \
    --image azch/artillery \
    --restart-policy Never \
    --command-line "artillery quick -r 1000 -d 300 $LOADTEST_API_ENDPOINT"
```
```
kubectl get hpa \
  --namespace ratingsapp -w
```    
    
    
## Referanslar
Bu içerik, Microsoft Learn sayfası referans alınarak değiştirilmiş/Türkçeleştirilmiştir.
    
[Azure Kubernetes Service Workshop](https://docs.microsoft.com/en-us/learn/modules/aks-workshop/)
