# Original Page
https://github.com/ksimir/grpc-go-api
<br/>
このページはksimirさんの許可をもらって日本語化してものです。
# Gaming Microservice Architecture (gRPC/Golang)
Sample Gaming Microservice gRPC APIsはgolangで作成されており、GCP (Google Cloud Platform)上でCloud Spannerをストレージレイヤーとして使ってます。このサンプルはGCPのGKE (Google Kubernetes Engine)でgRPC API Serverをホストしてますし、Cloud Endpointsを使ってセキュアにAPIとGCLB (L7 LB - ingress in k8s)でサービスしてます。
<img src="./gaming-microservice-arch.png">

GCPを初めての方はこのリンクを先に確認してください。[link](https://cloud.google.com/gcp/getting-started/)
<br/>
下記のコマンドは[Cloud Shell](https://cloud.google.com/shell/)または[Cloud SDK](https://cloud.google.com/sdk/)をインストールして実行する必要があります。
<br/>

## git clone
```
git clone https://github.com/minsoo-jun/grpc-go-api.git
cd grpc-go-apicd grpc-go-api
```

## project ID の設定:
```
export PROJECT_ID=$(gcloud config list project --format "value(core.project)")
```

## Cloud Spannerのinstance名とdatabaes名を設定:
Cloud Spannerのinstanceとdatabaseの作成方法についてはこのリンクを参考してください。[link](https://cloud.google.com/spanner/docs/quickstart-console)
<br/>
playerapi.Dockerfileとinventoryapi.Dockerfileの"ENV"部分を自分の環境の値を入れてください。
```
ENV PROJECTID="<< GCP Project ID >>"
ENV INSTANCE="<< Cloud Spanner Instance ID >>"
ENV DATABASE="<< Cloud Spanner Database Name >>"

```

## Docker image作成: 
指定したProject ID, Spanner Instance ID, Spanner Database でDocker Imageをビルド
```
docker build -t asia.gcr.io/${PROJECT_ID}/player-api:v1 . -f playerapi.Dockerfile

docker build -t asia.gcr.io/${PROJECT_ID}/inventory-api:v1 . -f inventoryapi.Dockerfile
```

## 作成したDocker imageを GCR (Google Container Repository)へ登録 
[gcloud docker とバージョン 18.03 以上の Docker クライアント](https://cloud.google.com/container-registry/docs/support/deprecation-notices#gcloud-docker)
```
gcloud docker -- push asia.gcr.io/${PROJECT_ID}/player-api:v1
gcloud docker -- push asia.gcr.io/${PROJECT_ID}/inventory-api:v1
```

## 下記のコマンドでイメージが正しく上がったかの確認ができます: 
```
gcloud container images list-tags asia.gcr.io/${PROJECT_ID}/player-api
gcloud container images list-tags asia.gcr.io/${PROJECT_ID}/inventory-api
```

## Web Appを GKEへDeploy (先deploymentを作成した後にserviceを作成する)
DeploymentするKubernetesクラスタが必要です。
```text
gcloud container clusters create grpc-game --zone asia-northeast1-c
```
下記のconifgファイルの`PROJECT_ID`, `INSTANCE`, `DATABASE`を自身のGCP Project IDやCloud SpannerのInstance ID、Database nameで置換してください :
- deployments/k8s/playerapi-deployment.yaml
- deployments/k8s/inventoryapi-deployment.yaml

```
kubectl create -f deployments/k8s/playerapi-deployment.yaml
kubectl create -f deployments/k8s/inventoryapi-deployment.yaml
kubectl create -f deployments/k8s/playerapi-service.yaml
kubectl create -f deployments/k8s/inventoryapi-service.yaml
```

## 作成されたDeploymentsとServicesを確認
```
kubectl get deployments
kubectl get svc
```

## Test your gRPC API using the client app located in cmd/player-client-grpc folder
下記のPackageのgo get必要です。<br/>
```text
go get github.com/google/uuid
go get golang.org/x/net/context
go get golang.org/x/oauth2/google
go get google.golang.org/grpc
go get google.golang.org/grpc/credentials
go get google.golang.org/grpc/metadata

```
下記のPackageは自分のgithub repositoryで置換するかksimirさんのRepositoryを指定してください。<br/>
自分のRepostoryに変更した場合にはmain.goのimport部分も同じく変更してください。
```text
go get github.com/minsoo-jun/grpc-go-api/pkg/api/v1/inventory
go get github.com/minsoo-jun/grpc-go-api/pkg/api/v1/player
```
テストプログラムを実行
```
$ cd cmd/client-grpc/
$ export EXTERNAL_IP=$(kubectl get service player-service --output jsonpath="{.status.loadBalancer.ingress[0].ip}")
$ go run main.go --grpc-address=${EXTERNAL_IP} --grpc-port=8080
```




## Secure your gRPC API using Cloud Endpoints
To first deploy Cloud Endpoints config without authentication, use api_config.yaml config file.

Create a proto descriptor file
```
$ protoc --include_imports --include_source_info --descriptor_set_out deployments/endpoints/player-service/player.pb api/proto/v1/player.proto
$ protoc --include_imports --include_source_info --descriptor_set_out deployments/endpoints/inventory-service/inventory.pb api/proto/v1/inventory.proto
```

Replace `PROJECT_ID` with your own GCP Project ID in the following config files:
- deployments/endpoints/player-service/api_config.yaml
- deployments/endpoints/inventory-service/api_config.yaml
- deployments/k8s/playerapi-endpoints-deployment.yaml
- deployments/k8s/inventoryapi-endpoints-deployment.yaml

Deploy the Endpoints configuration
```
$ cd deployments/endpoints
$ gcloud endpoints services deploy player-service/player.pb player-service/api_config.yaml
$ gcloud endpoints services deploy inventory-service/inventory.pb inventory-service/api_config.yaml
```

Then delete and redeploy your gRPC pods and service using the Cloud Endpoints ESP sidecar container
```
$ kubectl delete deployment player-deployment
$ kubectl delete deployment inventory-deployment
$ kubectl delete svc player-service
$ kubectl delete svc inventory-service
$ kubectl create -f ~/grpc-go-api/deployments/k8s/playerapi-endpoints-deployment.yaml
$ kubectl create -f ~/grpc-go-api/deployments/k8s/playerapi-endpoints-service.yaml
$ kubectl create -f ~/grpc-go-api/deployments/k8s/inventoryapi-endpoints-deployment.yaml
$ kubectl create -f ~/grpc-go-api/deployments/k8s/inventoryapi-endpoints-service.yaml
```

If the deployment is successful, you can access the GCP Console and start seeing metrics from the Cloud Endpoints portal.

## Configuring a quota for the gRPC API
Redeploy the Cloud Endpoints config predefined for quotas (api_config_quota.yaml), Replace `PROJECT_ID` with your own GCP Project ID
```
$ cd ~/grpc-go-api/deployments/endpoints
$ gcloud endpoints services deploy player-service/player.pb player-service/api_config_quota.yaml
```

Follow the instructions [here](https://cloud.google.com/docs/authentication/api-keys#creating_an_api_key) to create a new API key.

Test your gRPC API quota using the created API key
Replace `API_KEY` in the following command

```
$ cd ~/grpc-go-api/cmd/client-grpc/
$ EXTERNAL_IP=$(kubectl get service player-service --output jsonpath="{.status.loadBalancer.ingress[0].ip}")
$ go run main.go \
    --grpc-address=${EXTERNAL_IP} \
    --grpc-port=8080 \
    --api-key=API_KEY
```

## Configuring service account authentication for the gRPC API
Redeploy the Cloud Endpoints config predefined for service account authentication (api_config_auth.yaml)
```
$ cd deployments/endpoints
$ gcloud endpoints services deploy player-service/player.pb player-service/api_config_auth.yaml
```

Follow the instructions [here](https://cloud.google.com/endpoints/docs/grpc/service-account-authentication#creating_the_consumer_service_account_and_key) to create a new service account.

Test your gRPC API using a service account
Replace `PROJECT_ID` and `SERVICE_ACCOUNT` in the following command

```
$ cd ~/grpc-go-api/cmd/client-grpc/
$ EXTERNAL_IP=$(kubectl get service player-service --output jsonpath="{.status.loadBalancer.ingress[0].ip}")
$ go run main.go \
    --grpc-address=${EXTERNAL_IP} \
    --grpc-port=8080 \
    --keyfile=SERVICE_ACCOUNT_KEY.json \
    --audience=player.endpoints.PROJECT_ID.cloud.goog
```

<h1> Need to set Using a HTTP LB (ingress) to expose both gRPC services using managed SSL certificate

## HTTP LB (ingress)を使ってgRPC sericeをexposeさせる（managed SSL certificate)。 

Step 1 - global static IP addressを作成
```
$ gcloud compute addresses create ingress-ip --global
```

Step 2 - Create a managed SSL certificate (replace `API.DOMAIN.COM` by your own fqdn).
```
$ gcloud beta compute ssl-certificates create game-cert --domain API.DOMAIN.COM

$ gcloud beta compute ssl-certificates create game-cert --domains=api.msjun.net --global
Created [https://www.googleapis.com/compute/beta/projects/minsoojunprj/global/sslCertificates/game-cert].
NAME       TYPE     CREATION_TIMESTAMP             EXPIRE_TIME  MANAGED_STATUS
game-cert  MANAGED  2019-11-19T20:18:04.239-08:00               PROVISIONING
    api.msjun.net: PROVISIONING

```
> NOTE: the DNS record for your domain must reference the IP address you created at the previous step.
Otherwise you might get the error `FAILED_NOT_VISIBLE` for your managed SSL.
In my example, `API.DOMAIN.COM` DNS A record would point to the static IP generated during Step 1.

The following steps (3 & 4) are needed to enable SSL on ESP on Kubernetes.
For more details, please check https://cloud.google.com/endpoints/docs/grpc/enabling-ssl#ssl_keys_and_certificates

Step 3 - Any self-signed certificates seems to be ok so you can create the SSL key and certificate using OpenSSL
```
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
        -keyout ./nginx.key -out ./nginx.crt
```

Step 4 - Create a Kubernetes secret with your SSL key and certificate
```
$ kubectl create secret generic nginx-ssl \
     --from-file=./nginx.crt --from-file=./nginx.key
```
> FYI this k8s secret is used in the `playerapi-endpoints-ingress-deployment.yaml` and `inventoryapi-endpoints-ingress-deployment.yaml`

Step 5 - Clean up the previously deployed deployments and services
```
$ kubectl delete svc player-service
$ kubectl delete svc inventory-service
$ kubectl delete deployment inventory-deployment
$ kubectl delete deployment player-deployment
```

Step 6 - Finally redeploy the k8s deployments and services (updated to work with the HTTP LB) and deploy the HTTP LB to expose both gRPC APIs.

Replace `PROJECT_ID`, `INSTANCE`, `DATABASE` with your own GCP Project ID as well as your Cloud Spanner instance/db in the following config files:
- deployments/k8s/playerapi-endpoints-deployment.yaml
- deployments/k8s/inventoryapi-endpoints-deployment.yaml
```
$ kubectl create -f deployments/k8s/playerapi-endpoints-ingress-deployment.yaml
$ kubectl create -f deployments/k8s/playerapi-endpoints-ingress-service.yaml
$ kubectl create -f deployments/k8s/inventoryapi-endpoints-ingress-deployment.yaml
$ kubectl create -f deployments/k8s/inventoryapi-endpoints-ingress-service.yaml
$ kubectl create -f deployments/k8s/gameapi-ingress.yaml
```

Step 7 - Test your both gRPC APIs exposed by the HTTP LB (the gRPC path is used to route to the correct backend).
Replace `API.DOMAIN.COM` and `API_KEY` in the following command by the API key previously created
```
$ cd cmd/client-grpc
$ go run main.go \
    --grpc-address=API.DOMAIN.COM \
    --grpc-port=443 \
    --api-key=API_KEY
```
> Note that the test client (in `cmd/client-grpc`) only dials to the HTTP LB IP address once, the same gRPC connection is used to create a client for each gRPC API


