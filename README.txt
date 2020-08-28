기초 - nginx 서비스가 올라가 있는 상태
# nginx 생성
kubectl create deployment nginx --image=nginx
# 생성된 nginx 보기
kubectl get deployments.apps
# 생성된 nginx 라벨 보기
kubectl get po --show-labels
# scale replicas 1개로 지정
kubectl scale deployment --replicas=2 nginx


1. 서비스(Service)
서비스는 지정된 IP로 생성이 가능하고, 여러 Pod를 묶어서 로드 밸런싱이 가능하며, 고유한 DNS 이름을 가질 수 있다.

2. 동기
Pod의 경우 지정되는 IP가 랜덤하고 리스타트 때마다 변경되기 때문에 고정된 endpoint로 호출이 어렵다.
또한 여러 Pod에 같은 어플리케이션을 운용할 경우 이 Pod간의 로드밸런싱을 지원해줘야 하는데, Service가 이러한 역할을 한다.

3. 서비스 타입(Service Type)
Cluster IP (default)
Load Balancer
Node IP(nodeport)
External name

4. 타입별 구성방법
서비스는 다음과 같이 구성이 가능하며, 라벨 셀렉터 (label selector)를 이용하여, 관리하고자 하는 Pod들을 정의할 수 있다.
# pod의 LABELS 정보
[root@tylee-k8s-01 ~]# kubectl get po --show-labels
NAME                    READY   STATUS    RESTARTS   AGE    LABELS
nginx-f89759699-ndpd9   1/1     Running   0          9m5s   app=nginx,pod-template-hash=f89759699
nginx-f89759699-xm5bq   1/1     Running   0          9s     app=nginx,pod-template-hash=f89759699

# deployment 의 LABELS 정보
[root@tylee-k8s-01 ~]# kubectl get deployments.apps --show-labels
NAME    READY   UP-TO-DATE   AVAILABLE   AGE     LABELS
nginx   2/2     2            2           9m27s   app=nginx

Cluster IP
디폴트 설정으로, 서비스에 클러스터 IP (내부 IP)를 할당한다. 쿠버네티스 클러스터 내에서는 이 서비스에 접근이 가능하지만, 클러스터 외부에서는 외부 IP를 할당 받지 못했기 때문에 접근이 불가능하다.
# Cluster IP
[root@tylee-k8s-01 ~]# vim svc.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-svc
spec:
  selector:
    app: nginx
  ports:
    - port: 80
      protocol: TCP
      targetPort: 80

[root@tylee-k8s-01 ~]# kubectl create -f svc.yaml
service/nginx-svc-1 created

[root@tylee-k8s-01 ~]# kubectl get svc -o wide nginx-svc
NAME          TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE   SELECTOR
nginx-svc   ClusterIP   10.109.241.161   <none>        80/TCP    15s   app=nginx

[root@tylee-k8s-01 ~]# curl 10.109.241.161

로드밸런싱 default 알고리즘은 Pod간에 랜덤으로 부하를 분산하도록 한다.
만약 특정 클라이언트가 특정 Pod로 지속적으로 연결이 되게 하려면 Session Affinity를 사용하면 되는데, 서비스의 spec 부분에 sessionAffinity: ClientIP로 주면 된다.
ex)
apiVersion: v1
kind: Service 
spec:
  sessionAffinity: ClientIP
  ..

웹에서 HTTP Session을 사용하는 경우와 같이 각 서버에 각 클라이언트의 상태정보가 저장되어 있는 경우에 유용하게 사용할 수 있다.

Load Balancer
보통 클라우드 벤더에서 제공하는 설정 방식으로, 외부 IP를 가지고 있는 로드밸런서를 할당한다. 외부 IP를 가지고 있기 때문에 클러스터 외부에서 접근이 가능하다.
테스트를 위해서는 외부 로드밸런서가 필요하므로 추후 테스트 예정.

NodePort (30000-32767)
클러스터 IP로만 접근이 가능한 것이 아니라, 모든 노드의(master, worker) IP와 포트를 통해서도 접근이 가능하게 된다.
예를 들어 아래와 같이 nginx-nodeport-svc 이라는 서비스를 NodePort 타입으로 선언을 하고, NodePort를 30001으로 설정하면, 아래 설정에 따라 클러스터 IP의 80포트로도 접근이 가능하지만, 모든 노드의 30001 포트로도 서비스를 접근할 수 있다.

[root@tylee-k8s-01 ~]# vim nodeport-svc.yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-nodeport-svc
spec:
  selector:
    app: nginx
  type: NodePort
  ports:
    - name: http
      port: 80
      protocol: TCP
      targetPort: 80
      nodePort: 30001

[root@tylee-k8s-01 ~]# kubectl create -f nodeport-svc.yaml

[root@tylee-k8s-01 ~]# kubectl get svc -o wide nginx-nodeport-svc
NAME                   TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE   SELECTOR
nginx-nodeport-svc   NodePort   10.107.223.203   <none>        80:30001/TCP   90s   app=nginx

# 브라우저에서 192.168.0.101:30001 또는 192.168.0.102:30001 호출

ExternalName
ExternalName은 외부 서비스를 쿠버네티스 내부에서 호출하고자 할때 사용할 수 있다.
쿠버네티스 클러스터 내의 Pod들은 클러스터 IP를 가지고 있기 때문에 클러스터 IP 대역 밖의 서비스를 호출하고자 하면, NAT 설정등 복잡한 설정이 필요하다.
특히 AWS나 GCP와 같은 클라우드 환경을 사용할 경우 데이터베이스나 클라우드에서 제공되는 매니지드 서비스(RDS, CloudSQL)등을 사용하고자 할 경우에는 쿠버네티스 클러스터 밖이기 때문에 호출이 어려운 경우가 있는데, 이를 쉽게 해결할 수 있는 방법이 ExternalName 타입이다.
아래와 같이 서비스를 ExternalName 타입으로 설정하고, 주소를 DNS로 k8s.service.com으로 설정해주면 nginx-externalname-svc 는 들어오는 모든 요청을 k8s.service.com 으로 포워딩 해준다. (일종의 프록시와 같은 역할)

# 테스트 시 web서버 필요
# DNS를 이용하는 방식
[root@tylee-k8s-01 ~]# vim /etc/hosts
10.65.40.20 k8s.service.com

[root@tylee-k8s-01 ~]# vim dns-externalname-svc.yaml
apiVersion: v1
kind: Service
metadata:
  name: dns-externalname-svc
  namespace: default
spec:
  type: ExternalName
  externalName: k8s.service.com

[root@tylee-k8s-01 ~]# kubectl create -f dns-external-svc.yaml

[root@tylee-k8s-01 service]# kubectl get svc dns-external-svc
NAME                 TYPE           CLUSTER-IP   EXTERNAL-IP     PORT(S)   AGE
dns-external-svc   ExternalName   <none>       k8s.service.com   <none>    101s

[root@tylee-k8s-01 service]# curl k8s.service.com
k8s ExternalName test page

# DNS가 아닌 직접 IP를 이용하는 방식
[root@tylee-k8s-01 ~]# vim externalname-svc-1.yaml
apiVersion: v1
kind: Service
metadata:
  name: externalname-svc-httpd
spec:
  ports:
    - port: 80

[root@tylee-k8s-01 ~]# kubectl create -f externalname-svc-1.yaml

[root@tylee-k8s-01 ~]# kubectl get svc externalname-svc-httpd
NAME                     TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
externalname-svc-httpd   ClusterIP   10.108.247.71   <none>        80/TCP    82s

# EndPoint 지정, 이때 서비스명과 서비스 EndPoints의 이름이 동일해야 한다. externalname-svc-httpd로 같은 서비스명을 사용하였고 이 서비스는, 10.65.40.20:80 서비스를 가르키도록 되어 있다.
[root@tylee-k8s-01 ~]# vim externalname-svc-2.yaml
apiVersion: v1
kind: Endpoints
metadata:
  name: externalname-svc-httpd
subsets:
  - addresses:
    - ip: 10.65.40.20
    ports:
    - port: 80

[root@tylee-k8s-01 ~]# kubectl create -f externalname-svc-2.yaml

# 생성된 서비스 확인
[root@tylee-k8s-01 ~]# kubectl get svc -o wide externalname-svc-httpd
NAME                     TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE    SELECTOR
externalname-svc-httpd   ClusterIP   10.108.247.71   <none>        80/TCP    8m8s   <none>

# 쿠버네티스 내부 클러스터의 Pod에서 curl 명령을 이용해서 호출해보면 다음과 같이 외부 서비스를 호출할 수 있음을 확인할 수 있다.
[root@tylee-k8s-01 ~]# curl 10.108.247.71
k8s ExternalName test page

Headless Service
서비스는 접근을 위해서 Cluster IP 또는 External IP를 지정받는다.
즉 서비스를 통해서 제공되는 기능들에 대한 엔드포인트를 쿠버네티스 서비스를 통해서 통제하는 개념인데, 마이크로 서비스 아키텍쳐에서는 기능 컴포넌트에 대한 엔드포인트(IP주소)를 찾는 기능을 서비스 디스커버리 (Service Discovery) 라고 하고, 서비스의 위치를 등록해놓는 서비스 디스커버리 솔루션을 제공한다. etcd나 hashcorp의 consul과 같은 솔루션이 대표적인 사례인데, 이 경우 쿠버네티스 서비스를 통해서 마이크로 서비스 컴포넌트를 관리하는 것이 아니라 서비스 디스커버리 솔루션을 이용하기 때문에, 서비스에 대한 IP 주소가 필요없다.
이런 시나리오를 지원하기 위한 쿠버네티스의 서비스를 헤드리스서비스 (Headless service) 라고 하는데, 이러한 헤드리스 서비스는 Cluster IP등의 주소를 가지지 않는다. 단 DNS이름을 가지게 되는데, 이 DNS 이름을 lookup 해보면, 서비스(로드밸런서)의 IP를 리턴하지 않고, 이 서비스에 연결된 Pod들의 IP 주소들을 리턴하게 된다.

DNS 간단한 테스트 - nginx pod 이용
# headless service 생성
[root@tylee-k8s-01 service]# vim headless-svc.yaml
apiVersion: v1
kind: Service
metadata:
  name: headless-svc
spec:
  clusterIP: None
  selector:
    app: nginx
  ports:
    - name: nginx
      port: 80
      protocol: TCP
      targetPort: 80

[root@tylee-k8s-01 service]# kubectl create -f headless-svc.yaml

# Cluster-IP None 상태 확인
[root@tylee-k8s-01 service]# kubectl get svc headless-svc
NAME             TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
headless-svc-1   ClusterIP   None         <none>        80/TCP    5m6s


[root@tylee-k8s-01 service]# kubectl get pod
NAME                    READY   STATUS    RESTARTS   AGE
nginx-f89759699-ndpd9   1/1     Running   0          171m
nginx-f89759699-xm5bq   1/1     Running   0          162m

# Get a shell to the running container
[root@tylee-k8s-01 ~]# kubectl exec --stdin --tty nginx-f89759699-ndpd9 -- /bin/bash

apt-get update
apt-get install -y dnsutils

[root@tylee-k8s-01 service]# kubectl exec --stdin --tty nginx-f89759699-ndpd9 -- /bin/bash
root@nginx-f89759699-ndpd9:/# nslookup headless-svc
Server:         10.96.0.10
Address:        10.96.0.10#53

Name:   headless-svc.default.svc.cluster.local
Address: 10.244.1.7
Name:   headless-svc.default.svc.cluster.local
Address: 10.244.1.8

root@nginx-f89759699-ndpd9:/# curl headless-svc.default.svc.cluster.local

# External IP
[root@tylee-k8s-01 service_back]# vim externalip-svc.yaml
apiVersion: v1
kind: Service
metadata:
  name: externalip-svc
spec:
  selector:
    app: nginx
  ports:
    - name: nginx
      port: 80
      protocol: TCP
      targetPort: 80
  externalIPs:
  - 192.168.0.101

[root@tylee-k8s-01 service_back]# kubectl create -f externalip-svc.yaml

[root@tylee-k8s-01 service_back]# kubectl get svc externalip-svc

브라우저에서 external ip 로 호출
