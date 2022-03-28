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
## Kaynak Kod 1- AKS Kurulumu
#### Geliştirici ekip uygulamalar halihazırda container yapısı kullanıyor. Süreçleri hızlandırmak, yönetmek ve esnek bir şekilde büyüyüp küçülmesini sağlamak için AKS kullanılması isteniyor. Bu bölümde;
##### Resource Group oluşturulacak
##### Cluster networkü şekillendirilecek
##### AKS clusterı oluşturulacak
##### AKS clusterına bağlanılacak
##### Namespace oluşturulacak
### Resource Group Oluşturulması
#### 1- Windows terminal aracılığıyla Cloud Shell'e bağlanmak
#### 2- 

