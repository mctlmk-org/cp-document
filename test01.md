# 설치

테스트 여부
| No | Cluster1 | Cluster2 | 확인여부 |
| :--: | :--: | :--: | :--: |
|1|NHN|NCP|O| 
|2|NHN|NHN|O| 
|3|NHN|KT|X| 

## <span id='1'>1. 클러스터 준비
| No | Provider | API Server URL | ContextName |Node|
| :--: | :--: | :--: | :--: | :--: |
|1|NHN|https://133.186.152.215:6443|ctx-1|2Core 4G 1개|
|2|NCP|https://175.45.214.201:6443 |ctx-2|2Core 4G 1개|

> PodSecurity  설정 Off (/etc/kubernetes/admission-controls/podsecurity.yaml)


## <span id='2'>2. MetalLB 설치
> 양쪽 클러스터 모두 설치

### <span id='2.1'>2.1. ARP 설정 변경
MetalLB는 ARP프로토콜(OSI Layer 2)을 이용하기 때문에 `kube-proxy` 에서 `strictARP` 설정을 `true` 로 변경해 줘야 한다. 
- 명령어
```bash
kubectl get configmap kube-proxy -n kube-system -o yaml | sed -e "s/strictARP: false/strictARP: true/" | kubectl apply -f - -n kube-system
```
- 실행결과
```bash
Warning: resource configmaps/kube-proxy is missing the kubectl.kubernetes.io/last-applied-configuration annotation which is required by kubectl apply. kubectl apply should only be used on resources created declaratively by either kubectl create --save-config or kubectl apply. The missing annotation will be patched automatically.
configmap/kube-proxy configured
```

### <span id='2.2'>2.2. MetalLB 설치
- 명령어
```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.7/config/manifests/metallb-native.yaml
```

- 실행결과
```bash
namespace/metallb-system created
customresourcedefinition.apiextensions.k8s.io/addresspools.metallb.io created
customresourcedefinition.apiextensions.k8s.io/bfdprofiles.metallb.io created
customresourcedefinition.apiextensions.k8s.io/bgpadvertisements.metallb.io created
customresourcedefinition.apiextensions.k8s.io/bgppeers.metallb.io created
customresourcedefinition.apiextensions.k8s.io/communities.metallb.io created
customresourcedefinition.apiextensions.k8s.io/ipaddresspools.metallb.io created
customresourcedefinition.apiextensions.k8s.io/l2advertisements.metallb.io created
serviceaccount/controller created
serviceaccount/speaker created
role.rbac.authorization.k8s.io/controller created
role.rbac.authorization.k8s.io/pod-lister created
clusterrole.rbac.authorization.k8s.io/metallb-system:controller created
clusterrole.rbac.authorization.k8s.io/metallb-system:speaker created
rolebinding.rbac.authorization.k8s.io/controller created
rolebinding.rbac.authorization.k8s.io/pod-lister created
clusterrolebinding.rbac.authorization.k8s.io/metallb-system:controller created
clusterrolebinding.rbac.authorization.k8s.io/metallb-system:speaker created
secret/webhook-server-cert created
service/webhook-service created
deployment.apps/controller created
daemonset.apps/speaker created
validatingwebhookconfiguration.admissionregistration.k8s.io/metallb-webhook-configuration created
```

### <span id='2.3'>2.3. LoadBalancer 대역 설정
istioGateway에서 `External IP`로 사용할 LoadBalancer 대역을 설정한다. 

- metallb-config.yaml
```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: default
  namespace: metallb-system
spec:
  addresses:
    - 192.168.56.100-192.168.56.200
  autoAssign: true
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: default
  namespace: metallb-system
spec:
  ipAddressPools:
    - default
```

```bash
kubectl create -f metallb-config.yaml
```

> webhook 에러발생될 경우
>> `metallb-webhook-configuration` webhook을 삭제 후 재생성한다.   
>> kubectl delete validatingwebhookconfigurations metallb-webhook-configuration


## <span id='3'>3. Istio 멀티 클러스터 설치

### <span id='3.1'>3.1. Deployment 파일 다운로드
Istio 멀티 클러스터 설치를 위해 컨테이너 플랫폼 포털 Deployment 파일을 다운로드 받아 아래 경로로 위치시킨다.<br>

+ 컨테이너 플랫폼 포털 Deployment 파일 다운로드 :
   [cp-portal-deployment-v1.5.0.tar.gz](https://nextcloud.k-paas.org/index.php/s/SSo9H3qjLsFn3ob/download)

```bash
# Deployment 파일 다운로드 경로 생성
$ mkdir -p ~/workspace/container-platform
$ cd ~/workspace/container-platform

# Deployment 파일 다운로드 및 파일 경로 확인
$ wget --content-disposition https://nextcloud.k-paas.org/index.php/s/SSo9H3qjLsFn3ob/download

$ ls ~/workspace/container-platform
  cp-portal-deployment-v1.5.0.tar.gz

# Deployment 파일 압축 해제
$ tar -xvf cp-portal-deployment-v1.5.0.tar.gz
```

<br>

### <span id='3.2'>3.2. 도구 설치
Istio 멀티 클러스터 설치에 필요한 커맨드 라인 등 도구 설치 스크립트를 실행한다.
```bash
$ cd ~/workspace/container-platform/cp-portal-deployment/istio_mc
$ chmod +x install_tools.sh
$ ./install_tools.sh
```

<br>

### <span id='3.3'>3.3. 멀티 클러스터 접근 구성
Istio 멀티 클러스터를 설치할 클러스터 Cluster1, Cluster2에 접근할 수 있도록 컨텍스트 구성이 필요하다. <br>
:loudspeaker: Cluster1, Cluster2 두 kubeconfig 파일 내 cluster, context, user 명이 중복되지 않는지 확인한다.
```bash
# .kube 디렉터리 생성
$ mkdir -p ${HOME}/.kube

# Cluster1, Cluster2 kubeconfig 파일 위치
$ ls ${HOME}/.kube
cluster1-config  cluster2-config

# kubeconfig 파일 경로 설정
$ export KUBECONFIG="${HOME}/.kube/cluster1-config:${HOME}/.kube/cluster2-config"
```
- 컨텍스트 목록 조회 정상 확인
```bash
$ kubectl config get-contexts
CURRENT   NAME    CLUSTER    AUTHINFO         NAMESPACE
*         ctx-1   cluster1   cluster1-admin
          ctx-2   cluster2   cluster2-admin
```
- 클러스터 접근 정상 확인
```bash
# Cluster1 노드 조회 
$ kubectl get nodes --context=ctx-1
NAME                                  STATUS   ROLES    AGE     VERSION
k8s-cluster-1-default-worker-node-0   Ready    <none>   5h44m   v1.27.3

# Cluster2 노드 조회 
$ kubectl get nodes --context=ctx-2
NAME                                  STATUS   ROLES    AGE     VERSION
k8s-cluster-2-default-worker-node-0   Ready    <none>   5h43m   v1.27.3
```

<br>

### <span id='3.4'>3.4. Istio 멀티 클러스터 설치 변수 정의
Istio 멀티 클러스터를 설치하기 전 변수 값 정의가 필요하다. 설정에 필요한 정보를 확인하여 변수를 설정한다. 

```bash
$ cd ~/workspace/container-platform/cp-portal-deployment/istio_mc
$ vi istio-vars-mc.sh
```
```bash                                                     
# COMMON VARIABLE (Please change the value of the variables below.)
CLUSTER1_CONFIG[CTX]="{cluster1 context name}"    # Cluster1 Context Name
CLUSTER2_CONFIG[CTX]="{cluster2 context name}"    # Cluster2 Context Name
```

<b>CLUSTER1_CONFIG[CTX]</b><br>
클러스터 Cluster1의 컨텍스트 명 입력

<b>CLUSTER2_CONFIG[CTX]</b><br>
클러스터 Cluster2의 컨텍스트 명 입력

<br>

- 예시
```bash
# 컨텍스트 목록 조회
$ kubectl config get-contexts
CURRENT   NAME          CLUSTER    AUTHINFO         NAMESPACE
*         ctx-1 (입력)  cluster1   cluster1-admin
          ctx-2 (입력)  cluster2   cluster2-admin

# 컨텍스트 명 입력
CLUSTER1_CONFIG[CTX]="ctx-1"
CLUSTER2_CONFIG[CTX]="ctx-2"
```

<br>

### <span id='3.5'>3.5. Istio 멀티 클러스터 설치 스크립트 실행
Istio 멀티 클러스터를 설치를 위한 스크립트를 실행한다.
```bash
$ chmod +x deploy-istio-mc.sh
$ ./deploy-istio-mc.sh
```
```bash
...
[Install Istio in cluster1]...
Your certificate has been saved in certs/cluster-1/ca-cert.pem.
Your private key has been saved in certs/cluster-1/ca-key.pem.
namespace/istio-system created
secret/cacerts created
✔ Istio core installed
✔ Istiod installed
✔ Installation complete
Made this installation the default for injection and validation.
✔ Ingress gateways installed
✔ Installation complete
gateway.networking.istio.io/cross-network-gateway created
[Install Istio in cluster2]...
Your certificate has been saved in certs/cluster-2/ca-cert.pem.
Your private key has been saved in certs/cluster-2/ca-key.pem.
namespace/istio-system created
secret/cacerts created
✔ Istio core installed
✔ Istiod installed
✔ Installation complete
Made this installation the default for injection and validation.
✔ Ingress gateways installed
✔ Installation complete
gateway.networking.istio.io/cross-network-gateway created
secret/istio-remote-secret-cluster-1 created
secret/istio-remote-secret-cluster-2 created

--------------------------------------------------------------
[cluster1 (ctx-1)] $ istioctl remote-clusters
--------------------------------------------------------------
NAME          SECRET                                         STATUS      ISTIOD
cluster-2     istio-system/istio-remote-secret-cluster-2     syncing     istiod-79b559cf5f-hmvjc

--------------------------------------------------------------
[cluster2 (ctx-2)] $ istioctl remote-clusters
--------------------------------------------------------------
NAME          SECRET                                         STATUS     ISTIOD
cluster-1     istio-system/istio-remote-secret-cluster-1     synced     istiod-5df6b4d4d8-chp5w
```

```bash
$ kubectl get svc -n istio-system --context=ctx-1
NAME                   TYPE           CLUSTER-IP      EXTERNAL-IP    PORT(S)                                                                                                      AGE
istio-ingressgateway   LoadBalancer   10.233.16.69    192.168.0.71   15021:30841/TCP,80:32693/TCP,443:31547/TCP,31400:32365/TCP,15443:31599/TCP,15012:30856/TCP,15017:30501/TCP   2m23s
istiod                 ClusterIP      10.233.31.192   <none>         15010/TCP,15012/TCP,443/TCP,15014/TCP                                                                        2m41s
$ kubectl get svc -n istio-system --context=ctx-2
NAME                   TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)                                                                                                      AGE
istio-ingressgateway   LoadBalancer   10.233.18.241   172.20.0.15   15021:30911/TCP,80:30085/TCP,443:32223/TCP,31400:31205/TCP,15443:32114/TCP,15012:32424/TCP,15017:31039/TCP   116s
istiod                 ClusterIP      10.233.9.234    <none>        15010/TCP,15012/TCP,443/TCP,15014/TCP                                                                        2m13s

```
<br>

### <span id='3.6'>3.6. (참조) Istio 멀티 클러스터 삭제
설치된 Istio 멀티 클러스터 구성의 삭제를 원하는 경우 아래 스크립트를 실행한다.<br>
:loudspeaker: (주의) 해당 스크립트 실행 시, **Istio 멀티클러스터 구성이 모두 제거**되므로 주의가 필요하다.<br>

```bash
$ cd ~/workspace/container-platform/cp-portal-deployment/istio_mc
$ chmod +x uninstall-istio-mc.sh
$ ./uninstall-istio-mc.sh
```
``` 
(결과 업데이트)
```

<br>

## <span id='4'>4. 샘플 어플리케이션 배포 

## <span id='4.1'>4.1. ctx-2(naver)에 PREROUTING 생성
> 라우팅 설정은 각 Node(Master,Worker)에 모두 설정 필요 / Pod가 실행되는 쪽에서 Routing이 걸려서 처리됨.  

- iptables 라우팅 설정  
-d :: 상대편 Cluster의 Gateway Loadbalancer External IP  
--to-destination :: 상대편 Cluster의 Gateway Loadbalancer External IP에 설정한 공인IP
```
$ sudo iptables -t nat -I PREROUTING -p tcp -d 192.168.0.120 -j DNAT --to-destination 133.186.212.116
```
- 라우팅 설정 확인
```
$ sudo iptables -nL PREROUTING -t nat --line-numbers
```
- 라우팅 삭제(처리용)
```
sudo iptables -D PREROUTING {해당Number} -t nat
```
## <span id='4.2'>4.2. 샘플 배포
```sh
kubectl --context=ctx-1 create ns sample
kubectl --context=ctx-2 create ns sample
kubectl --context=ctx-1 label namespace sample istio-injection=enabled --overwrite
kubectl --context=ctx-2 label namespace sample istio-injection=enabled --overwrite
```

### 테스트용 Yaml 생성
아래 두개 Yaml 파일은 istio를 다운받으면 동일하게 존재하는 Yaml이다. 

#### helloworld.yaml
```yaml
apiVersion: v1
kind: Service
metadata:
  name: helloworld
  labels:
    app: helloworld
    service: helloworld
spec:
  ports:
  - port: 5000
    name: http
  selector:
    app: helloworld
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: helloworld-v1
  labels:
    app: helloworld
    version: v1
spec:
  replicas: 1
  selector:
    matchLabels:
      app: helloworld
      version: v1
  template:
    metadata:
      labels:
        app: helloworld
        version: v1
    spec:
      containers:
      - name: helloworld
        image: docker.io/istio/examples-helloworld-v1
        resources:
          requests:
            cpu: "100m"
        imagePullPolicy: IfNotPresent #Always
        ports:
        - containerPort: 5000
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: helloworld-v2
  labels:
    app: helloworld
    version: v2
spec:
  replicas: 1
  selector:
    matchLabels:
      app: helloworld
      version: v2
  template:
    metadata:
      labels:
        app: helloworld
        version: v2
    spec:
      containers:
      - name: helloworld
        image: docker.io/istio/examples-helloworld-v2
        resources:
          requests:
            cpu: "100m"
        imagePullPolicy: IfNotPresent #Always
        ports:
        - containerPort: 5000
        
```

#### sleep.yaml
```yaml
apiVersion: v1
kind: Service
metadata:
  name: helloworld
  labels:
    app: helloworld
    service: helloworld
spec:
  ports:
  - port: 5000
    name: http
  selector:
    app: helloworld
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: sleep
---
apiVersion: v1
kind: Service
metadata:
  name: sleep
  labels:
    app: sleep
spec:
  ports:
  - port: 80
    name: http
  selector:
    app: sleep
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sleep
spec:
  replicas: 1
  selector:
    matchLabels:
      app: sleep
  template:
    metadata:
      labels:
        app: sleep
        sidecar.istio.io/inject: "true"
    spec:
      serviceAccountName: sleep
      containers:
      - name: sleep
        image: curlimages/curl
        command: ["/bin/sleep", "3650d"]
        imagePullPolicy: IfNotPresent
```


#### ctx-1에 배포
```sh
$ kubectl --context=ctx-1 -n sample apply -f helloworld.yaml  

$ kubectl --context=ctx-1 -n sample get pods
NAME                             READY   STATUS    RESTARTS   AGE
helloworld-v1-78b9f5c87f-x7tb9   2/2     Running   0          67m
helloworld-v2-54dddc5567-rg2tr   2/2     Running   0          67m

$ kubectl --context=ctx-1 -n sample get service
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
helloworld   ClusterIP   10.233.18.239   <none>        5000/TCP   67m
```

#### ctx-2에 배포
```sh
$ kubectl --context=ctx-2 -n sample apply -f sleep.yaml

$ kubectl --context=ctx-2 -n sample get pods
NAME                     READY   STATUS    RESTARTS   AGE
sleep-5cc8999566-g77ws   2/2     Running   0          97m

$ kubectl --context=ctx-2 -n sample get service
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
helloworld   ClusterIP   10.233.10.184   <none>        5000/TCP   3h49m
sleep        ClusterIP   10.233.29.143   <none>        80/TCP     3h49m
```


#### curl로 확인
ctx-2번에서 실행중인 sleep pod에 접속하여 helloworld.sample:5000으로 curl을 실행하면 정상적으로 리턴되는 것을 볼 수 있다. 
```sh
ubuntu@cp-cluster-a-1:~/tmp$ kubectl --context=ctx-2 -n sample exec -it sleep-5cc8999566-g77ws sh
kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl exec [POD] -- [COMMAND] instead.
~ $ curl helloworld.sample:5000/hello
Hello version: v1, instance: helloworld-v1-775c565884-mqz7c
~ $ curl helloworld.sample:5000/hello
Hello version: v2, instance: helloworld-v2-7bb54f8948-rqx2p
~ $ curl helloworld.sample:5000/hello
Hello version: v1, instance: helloworld-v1-775c565884-mqz7c
~ $ curl helloworld.sample:5000/hello
Hello version: v1, instance: helloworld-v1-775c565884-mqz7c

```

<br>

-------
| Cluster1 | Cluster2 | 확인여부 |
| :--: | :--: | :--: |
|NHN|KT|O| 
```sh
sudo iptables -t nat -I PREROUTING -p tcp -d 192.168.0.240 -j DNAT --to-destination 133.186.135.191
sudo iptables -t nat -I PREROUTING -p tcp -d 192.168.0.240 --dport 15443 -j DNAT --to-destination 133.186.135.191
sudo iptables -t nat -I PREROUTING -p tcp -d 192.168.0.240 --dport 15443 -j DNAT --to-destination 133.186.135.191:15443
```


