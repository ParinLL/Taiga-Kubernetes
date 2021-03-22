# Taiga-Kubernetes

## 使用方式

請clone此git

```bash
git clone https://github.com/ParinLL/Taiga-Kubernetes
```
並提供兩種部署方式
kubernetes
docker-compose

預設帳號為**admin** ， 密碼為**123123**

### Kubernetes

進入kubernetes資料夾

```bash
cd Taiga-Kubernetes/kubernetes
```

修改[variables-env-configmap.yaml](https://github.com/ParinLL/Taiga-Kubernetes/blob/main/kubernetes/variables-env-configmap.yaml) 中的變數，否則不能成功部署

修改完成後，請部署當前kubernetes資料夾中的所有yaml

```bash
kubectl apply -f . ##或指定namespace (-n $namespace)
```

這樣就完成taiga服務的部署
但此時還無法從外部連入

請使用Ingress 的方式暴露proxy這個service

請勿修改**ENABLE_SSL** 的值

當**TAIGA_SCHEME=http** 時，TAIGA_PORT=80，必須允許在ingress可以使用http://$TAIGA_HOST:80連入

當**TAIGA_SCHEME=https** 時，TAIGA_PORT=443，且必須在ingress配置憑證，連入位置為https://$TAIGA_HOST:443



### docker-compose

進入docker-compose資料夾

```bash
cd Taiga-Kubernetes/docker-compose
```

修改[variables.env](https://github.com/ParinLL/Taiga-Kubernetes/blob/main/docker-compose/variables.env) 中的變數，否則不能成功部署

當**TAIGA_SCHEME=http** 時，TAIGA_PORT=80，ENABLE_SSL=no，proxy container會expose 80 port提供服務

當**TAIGA_SCHEME=https** 時，TAIGA_PORT=443，ENABLE_SSL=yes，並建立**certs** 資料夾，放入crt與key

取消註解CERT_NAME，CERT_KEY，並填入crt與key的名稱

修改[docker-compose.yaml](https://github.com/ParinLL/Taiga-Kubernetes/blob/main/docker-compose/docker-compose.yaml) ，並修改service proxy下的值，取消註解**\- 443:443** 與**\- ./cert:/taiga-cert** 

修改完成之後，使用以下指令開始部署

```bash
docker-compose up -d
```

