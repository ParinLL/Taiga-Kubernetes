# Taiga-Kubernetes

專案修改自
[docker-taiga](https://github.com/docker-taiga)/**[taiga](https://github.com/docker-taiga/taiga)**



此專案原本是docker-compose的方式進行部署，轉換成yaml的作法是透過

[kubernetes](https://github.com/kubernetes)/**[kompose](https://github.com/kubernetes/kompose)**  進行轉換

原本的docker-compose.yaml如下

```yaml
version: '3'

services:
  back:
    image: dockertaiga/back:6.0.1
    container_name: taiga-back
    restart: unless-stopped
    depends_on:
      - db
      - events
    networks:
      - default
    volumes:
      - ./data/media:/taiga-media
      - ./conf/back:/taiga-conf
    env_file:
      - variables.env

  front:
    image: dockertaiga/front:6.0.1
    container_name: taiga-front
    restart: unless-stopped
    networks:
      - default
    volumes:
      - ./conf/front:/taiga-conf
    env_file:
      - variables.env
      
  events:
    image: dockertaiga/events:6.0.0
    container_name: taiga-events
    restart: unless-stopped
    depends_on:
      - rabbit
    networks:
      - default
    env_file:
      - variables.env

  db:
    image: postgres:12-alpine
    container_name: taiga-db
    restart: unless-stopped
    networks:
      - default
    env_file:
      - variables.env
    volumes:
      - ./data/db:/var/lib/postgresql/data

  rabbit:
    image: dockertaiga/rabbit
    container_name: taiga-rabbit
    restart: unless-stopped
    networks:
      - default
    env_file:
      - variables.env

  proxy:
    image: dockertaiga/proxy
    container_name: taiga-proxy
    restart: unless-stopped
    depends_on:
      - back
      - front
      - events
    networks:
      - default
    ports:
      - 80:80
      - 443:443
    volumes:
      #- ./cert:/taiga-cert
      - ./conf/proxy:/taiga-conf
    env_file:
      - variables.env

networks:
  default:
```



裡面的container使用**variables.env**這個檔案去給env

variables.env:

```
TAIGA_HOST=taiga.lan
TAIGA_SCHEME=http
TAIGA_PORT=80
TAIGA_BACK_HOST=back
TAIGA_FRONT_HOST=front
EVENTS_HOST=events
TAIGA_SECRET=secret

ENABLE_SSL=no
# CERT_NAME=fullchain.pem
# CERT_KEY=privkey.pem

POSTGRES_HOST=db
POSTGRES_DB=taiga
POSTGRES_USER=postgres
POSTGRES_PASSWORD=password

RABBIT_HOST=rabbit
RABBIT_USER=taiga
RABBIT_PASSWORD=password
RABBIT_VHOST=taiga
```

由於要移植到k8s上，我們計畫以ingress+cert-manager來做https轉發，所以這邊不帶憑證

轉換完成後
如下

```bash
-> # kompose convert -f ../../docker-compose.yml                                         
WARN Restart policy 'unless-stopped' in service rabbit is not supported, convert it to 'always'
WARN Restart policy 'unless-stopped' in service proxy is not supported, convert it to 'always'
WARN Restart policy 'unless-stopped' in service back is not supported, convert it to 'always'
WARN Restart policy 'unless-stopped' in service front is not supported, convert it to 'always'
WARN Restart policy 'unless-stopped' in service events is not supported, convert it to 'always'
WARN Restart policy 'unless-stopped' in service db is not supported, convert it to 'always'
WARN Volume mount on the host "/root/taiga/data/media" isn't supported - ignoring path on the host
WARN Volume mount on the host "/root/taiga/conf/back" isn't supported - ignoring path on the host
INFO Network default is detected at Source, shall be converted to equivalent NetworkPolicy at Destination
WARN Volume mount on the host "/root/taiga/data/db" isn't supported - ignoring path on the host
INFO Network default is detected at Source, shall be converted to equivalent NetworkPolicy at Destination
INFO Network default is detected at Source, shall be converted to equivalent NetworkPolicy at Destination
WARN Volume mount on the host "/root/taiga/conf/front" isn't supported - ignoring path on the host
INFO Network default is detected at Source, shall be converted to equivalent NetworkPolicy at Destination
WARN Volume mount on the host "/root/taiga/conf/proxy" isn't supported - ignoring path on the host
INFO Network default is detected at Source, shall be converted to equivalent NetworkPolicy at Destination
INFO Network default is detected at Source, shall be converted to equivalent NetworkPolicy at Destination
INFO Kubernetes file "proxy-service.yaml" created
INFO Kubernetes file "back-deployment.yaml" created
INFO Kubernetes file "variables-env-configmap.yaml" created
INFO Kubernetes file "back-claim0-persistentvolumeclaim.yaml" created
INFO Kubernetes file "back-claim1-persistentvolumeclaim.yaml" created
INFO Kubernetes file "default-networkpolicy.yaml" created
INFO Kubernetes file "db-deployment.yaml" created
INFO Kubernetes file "db-claim0-persistentvolumeclaim.yaml" created
INFO Kubernetes file "events-deployment.yaml" created
INFO Kubernetes file "front-deployment.yaml" created
INFO Kubernetes file "front-claim0-persistentvolumeclaim.yaml" created
INFO Kubernetes file "proxy-deployment.yaml" created
INFO Kubernetes file "proxy-claim0-persistentvolumeclaim.yaml" created
INFO Kubernetes file "rabbit-deployment.yaml" created
```

結構如下

![image-20210318112517524](C:/Users/6656/Documents/github/dock8s/Taiga/images/image-20210318112517524.png)

其中做比較多的修改為service的部分，因為docker compose在同一個network下的container可以互通，所以不用指定expose port，所以參照各container原始Dockerfile後，指派原本的各項container為獨立的service物件，而proxy以LoadBalancer type做為入口，這邊就省略了ingress+https的部分

另外還有把

```yaml
        - env:
            - name: 
              valueFrom:
                configMapKeyRef:
                	key:
                	name:
```

改成

```yaml
        - envFrom:
          - configMapRef:
              name: variables-env
```



然而中間遇到container initial階段出錯

發現是back這個container的

[start.sh](https://github.com/docker-taiga/back/blob/master/start.sh)

這個腳本內的sed會報錯**sed: bad option in substitution expression** 並且不能生效

```
 sed -e 's/$TAIGA_HOST/'$TAIGA_HOST'/' \
```

後來發現不能以/作為分隔，可以用~或@等符號取代

不過在docker內執行就沒問題，k8s就會出事?確實是值得探討的地方

後來檢查另外幾個有start.sh的image，並修改相對應script

另外原本的image沒有帶入SMTP相關env，在這裡把變數加入後端的config.py並重新打包

yaml路徑已經放在本人的Github上

請務必修改**variables-env-configmap.yaml**才能正確部署

### [Taiga-Kubernetes](https://github.com/ParinLL/Taiga-Kubernetes)