### [Index](https://github.com/K-PaaS/Guide/blob/master/README.md) > [K-PaaS Sidecar Install](./README.md) > Sidecar

## Table of Contents

1. [문서 개요](#1)  
  1.1. [목적](#1.1)  
  1.2. [범위](#1.2)  
  1.3. [참고자료](#1.3)  

2. [K-PaaS Sidecar 설치](#2)  
  2.1. [Prerequisite](#2.1)  
  2.2. [실행파일 소개](#2.2)  
  2.3. [실행파일 다운로드](#2.3)  
  2.4. [variable 설정](#2.4)  
  2.5. [네임스페이스 생성 & 레지스트리 정보 입력](#2.5)  
  2.6. [Sidecar Dependency 배포](#2.6)  
  2.7. [Sidecar 설치](#2.7)  
  2.8. [Sidecar 로그인 및 테스트 앱 배포](#2.8)  
  　※ [(참고) Sidecar 삭제](#2.8.1)  
  　※ [(참고) Sidecar Dependency 삭제 (Sidecar 삭제 후)](#2.8.2)  
  2.9. [Sidecar User 생성](#2.9)  
  　2.9.1. [Sidecar User Account 생성](#2.9.1)  
  　2.9.2. [Sidecar Service Account 생성](#2.9.2)  
  　　※ [(참고) Container Platform Portal 계정을 사용하여 Sidecar 접속](#2.9.2.1)  
  　2.9.3. [Sidecar Admin 권한 부여](#2.9.3)  
  ※ [(참고) Container Platform Portal Harbor를 활용한 Sidecar 설치](#2.10)  

<br><br>
# <div id='1'> 1. 문서 개요
## <div id='1.1'> 1.1. 목적
본 문서는 K-PaaS Container-Platform 단독 배포 환경에서 K-PaaS Sidecar(이하 Sidecar)를 설치하기 위한 가이드를 제공하는 데 목적이 있다.

<br>

## <div id='1.2'> 1.2. 범위
본 문서는 [korifi v0.12.0](https://github.com/cloudfoundry/korifi/tree/v0.12.0), [sidecar-deployment v2.0.0-beta2](https://github.com/K-PaaS/sidecar-deployment/tree/v2.0.0-beta2), [cp-deployment v1.5.1.1](https://github.com/k-paas/cp-deployment/tree/v1.5.1.1)을 기준으로 작성하였다.    
본 문서는 K-PaaS Container-Platform 단독 배포(Kubespray)를 활용하여 Kubernetes Cluster를 구성 후 Sidecar 설치 기준으로 작성하였다.  
본 문서는 IaaS, Kubernetes에 대한 기본 이해도가 있다는 전제하에 가이드를 진행하였다.  

<br>


## <div id='1.3'> 1.3. 참고자료
K-PaaS 컨테이너 플랫폼 : [https://github.com/K-PaaS/container-platform](https://github.com/K-PaaS/container-platform)  
Kubespray : [https://kubespray.io](https://kubespray.io)  
Kubespray github : [https://github.com/kubernetes-sigs/kubespray](https://github.com/kubernetes-sigs/kubespray)  
korifi github : [https://github.com/cloudfoundry/korifi](https://github.com/cloudfoundry/korifi)  

<br>

# <div id='2'> 2. K-PaaS Sidecar 설치

## <div id='2.1'> 2.1. Prerequisite
- Default StorageClass 지정
- OCI 호환 이미지 레지스트리 제공 (e.g. [Docker Hub](https://hub.docker.com/), [Google container registry](https://cloud.google.com/container-registry),  [Azure container registry](https://hub.docker.com/), [Harbor](https://goharbor.io/), etc....)  
- LoadBalancer 사용
- OS - Ubuntu

<br>


## <div id='2.2'> 2.2. 실행파일 소개
- Sidecar를 설치 & 활용하기 위해선 다음과 같은 실행파일이 필요하다.

| 이름   |      설명      |
|----------|-------------|
| [cf cli](https://github.com/cloudfoundry/cli) (v8+) | Sidecar와 상호 작용하는 툴 |
| [kubectl](https://github.com/kubernetes/kubectl) | Kubernetes Cluster를 제어하는 툴 |
| [yq](https://github.com/mikefarah/yq) | YAML 편집 툴 |
| [cmctl](https://github.com/cert-manager/cert-manager) | cert-manager와 상호 작용하는 툴 |
| [ytt](https://carvel.dev/ytt/) | 템플릿을 이용하여 YAML을 구성하는 툴 |
| [kapp](https://carvel.dev/kapp/) | 리소스를 어플리케이션화하여 관리하는 툴 |

  

<br>

- Sidecar를 설치 시 사용되는 스크립트는 다음과 같다.

| 이름   |      설명      | 비고 |
|----------|-------------|----|
| utils-install.sh | Sidecar 설치 & 활용 시 사용되는 툴 설치 스크립트 | cf cli, yq, cmctl, ytt, kapp 설치 |
| variables.yml | Sidecar 설치 시 적용하는 변수 설정 파일 ||
| 1.init.sh | Sidecar 설치 시 사용 할 namespace와 이미지 레지스트리 정보 입력하는 스크립트 ||
| 2.deploy-dependency.sh | Sidecar를 실행 시 필요한 패키지를 설치하는 스크립트 | cert-manager, contour, kpack, servicebinding 설치 |
| 3.deploy-sidecar.sh | Sidecar를 설치하는 스크립트 ||
| delete-sidecar.sh | Sidecar를 삭제하는 스크립트 ||
| delete-dependency.sh | Sidecar Dependency를 삭제하는 스크립트 ||
| deploy-inject-self-signed-cert.sh | 자체 서명된 인증서를 사용하는 Private 레지스트리 사용 시 POD에 CA를 삽입하는 보조 스크립트 | 자세한 가이드는 deploy-inject-self-signed-cert.sh 파일 안 설명이나 [cert-injection-webhook](https://github.com/vmware-tanzu/cert-injection-webhook) 참고 |
| delete-inject-self-signed-cert.sh | inject-self-signed-cert를 삭제하는 스크립트 |  |
| install-test.sh | 설치 후 Test App을 배포하여 확인하는 스크립트 | |


<br>

## <div id='2.3'> 2.3. 실행파일 다운로드

- git clone 명령을 통해 다음 경로에서 Sidecar 다운로드를 진행한다. 본 설치 가이드에서의 Sidecar의 버전은 v2.0.0-beta2 버전이다.

```
$ cd $HOME
$ git clone https://github.com/K-PaaS/sidecar-deployment.git -b v2.0.0-beta2
$ cd sidecar-deployment/install-scripts
$ chmod +x ./install-test.sh
$ chmod +x ./support-files/user/*.sh
```

- utils-install.sh 파일을 실행하여 Sidecar 설치 시 필요한 실행 파일을 설치한다.  
```
$ source utils-install.sh
```

<br>


## <div id='2.4'> 2.4. variable 설정
- variables.yml 파일을 편집하여 Sidecar 설치 시 옵션들을 설정한다.
```
$ vi variables.yml
```
```yaml
# sidecar variable

## k8s variable
sidecar_namespace=sidecar                                    # sidecar install namespace
root_namespace=kpaas                                         # sidecar resource namespace

## sidecar core variable
system_domain=sidecar.com                                    # sidecar system_domain (e.g. 3.35.135.135.nip.io)
admin_username=sidecar-admin                                 # sidecar admin username
user_certificate_expiration_duration_days=365                # user cert duration (days)


## registry variable
use_dockerhub=true                                           # Registry kind (if dockerhub ==> true, harbor... ==> false)
registry_id=registry_id                                      # Registry ID
registry_password=registry_password                          # Registry Password

### registry variable (if use_dockerhub == false)
registry_address=harbor00.nip.io                             # Registry Address
registry_repositry_name=repository_name                      # Registry Name
is_self_signed_certificate=false                             # is private registry use self-signed certificate? (e.g. true or false)

#### registry variable (if use_dockerhub == false && is_self_signed_certificate == true)
registry_cert_path=support-files/private-repository.ca       # if is_self_signed_certificate==true --> add the contents of the private-repository.ca file
                                                             # if is_self_signed_certificate==false --> private-repository.ca is empty
cert_secret_name=harbor-cert                                 # ca cert secret name (k8s secret resource)
```
- 주요 변수의 설명은 다음과 같다.

| 이름   |      설명      |
|----------|-------------|
| sidecar_namespace | Sidecar 설치 namespace |
| root_namespace | Sidecar Resource namespace |
| system_domain | Sidecar 시스템 도메인 |
| admin_username | Sidecar 관리자 이름 |
| user_certificate_expiration_duration_days | 유저를 생성할 시 인증서 기간 |
| use_dockerhub | true 시 Dockerhub 사용, false 시 기타 이미지 레지스트리 사용 |
| registry_id | 이미지 레지스트리 아이디 |
| registry_password | 이미지 레지스트리 패스워드 |
| registry_address | 이미지 레지스트리 주소 (use_dockerhub false 시) |
| registry_repositry_name | 이미지 레지스트리 레포지토리 이름 (use_dockerhub false 시) |
| is_self_signed_certificate | https가 적용된 이미지 레지스트리 사용 시 self signed 인증서 사용하는 경우 true (use_dockerhub false 시) |
| registry_cert_path | self signed 인증서 경로 (is_self_signed_certificate true 시) |
| cert_secret_name | self signed 인증서를 저장하는 Kubernetes 이름 |

<br>

## <div id='2.5'> 2.5. 네임스페이스 생성 & 레지스트리 정보 입력
- 다음 스크립트를 실행하여 Sidecar에서 사용하는 네임스페이스와 레지스트리 정보를 입력한다.

```
$ source 1.init.sh

namespace/sidecar created
namespace/kpaas created
secret/image-registry-credentials created
```

<br>

## <div id='2.6'> 2.6. Sidecar Dependency 배포
- Sidecar를 실행 시 필요한 패키지를 설치한다.

```
$ source 2.deploy-dependency.sh

............

====================cert-manager====================

NAME                                           READY   STATUS    RESTARTS   AGE
pod/cert-manager-7ddd8cdb9f-8knk9              1/1     Running   0          49s
pod/cert-manager-cainjector-57cd76c845-pw24w   1/1     Running   0          49s
pod/cert-manager-webhook-cf8f9f895-rcwq8       1/1     Running   0          49s

NAME                           TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
service/cert-manager           ClusterIP   10.233.24.106   <none>        9402/TCP   49s
service/cert-manager-webhook   ClusterIP   10.233.24.109   <none>        443/TCP    49s

NAME                                      READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/cert-manager              1/1     1            1           49s
deployment.apps/cert-manager-cainjector   1/1     1            1           49s
deployment.apps/cert-manager-webhook      1/1     1            1           49s

NAME                                                 DESIRED   CURRENT   READY   AGE
replicaset.apps/cert-manager-7ddd8cdb9f              1         1         1       49s
replicaset.apps/cert-manager-cainjector-57cd76c845   1         1         1       49s
replicaset.apps/cert-manager-webhook-cf8f9f895       1         1         1       49s


======================contour=======================

NAME                                               READY   STATUS    RESTARTS   AGE
pod/contour-gateway-provisioner-55cb599fd5-zl4jl   1/1     Running   0          25s

NAME                                          READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/contour-gateway-provisioner   1/1     1            1           25s

NAME                                                     DESIRED   CURRENT   READY   AGE
replicaset.apps/contour-gateway-provisioner-55cb599fd5   1         1         1       25s


======================kpack=========================

NAME                                   READY   STATUS    RESTARTS   AGE
pod/kpack-controller-f5c468f4c-bg7p5   1/1     Running   0          21s
pod/kpack-webhook-76475959c6-tg9vj     1/1     Running   0          20s

NAME                    TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
service/kpack-webhook   ClusterIP   10.233.21.15   <none>        443/TCP   21s

NAME                               READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/kpack-controller   1/1     1            1           21s
deployment.apps/kpack-webhook      1/1     1            1           20s

NAME                                         DESIRED   CURRENT   READY   AGE
replicaset.apps/kpack-controller-f5c468f4c   1         1         1       21s
replicaset.apps/kpack-webhook-76475959c6     1         1         1       20s


===================service-binding==================

NAME                                                     READY   STATUS    RESTARTS   AGE
pod/servicebinding-controller-manager-8549ff4457-xhmlm   2/2     Running   0          19s

NAME                                                        TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)    AGE
service/servicebinding-controller-manager-metrics-service   ClusterIP   10.233.20.48   <none>        8443/TCP   19s
service/servicebinding-webhook-service                      ClusterIP   10.233.7.47    <none>        443/TCP    19s

NAME                                                READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/servicebinding-controller-manager   1/1     1            1           19s

NAME                                                           DESIRED   CURRENT   READY   AGE
replicaset.apps/servicebinding-controller-manager-8549ff4457   1         1         1       19s
```
<br>

## <div id='2.7'> 2.7. Sidecar 설치
- Sidecar를 설치한다.

```
$ source 3.deploy-sidecar.sh

Release "sidecar" does not exist. Installing it now.
NAME: sidecar
LAST DEPLOYED: Fri Jul  5 08:24:35 2024
NAMESPACE: sidecar
STATUS: deployed
REVISION: 1
TEST SUITE: None
create admin
certificatesigningrequest.certificates.k8s.io/0d65eb34fb58c67c3ebf04e5df8ceaacb28421d6 created
certificatesigningrequest.certificates.k8s.io/0d65eb34fb58c67c3ebf04e5df8ceaacb28421d6 approved
certificatesigningrequest.certificates.k8s.io/0d65eb34fb58c67c3ebf04e5df8ceaacb28421d6 condition met
certificatesigningrequest.certificates.k8s.io "0d65eb34fb58c67c3ebf04e5df8ceaacb28421d6" deleted
Cluster "cluster1" set.
User "sidecar-admin" set.
Context "sidecar-admin" created.
Switched to context "sidecar-admin".
kubeconfig file : /home/ubuntu/sidecar-deployment/install-scripts/support-files/user/sidecar-sidecar-admin.ua.kubeconfig
NAME                                                         READY   STATUS    RESTARTS   AGE
pod/korifi-api-deployment-7f4fccbb84-pd9k6                   1/1     Running   0          60s
pod/korifi-controllers-controller-manager-76986974d6-sqqk6   1/1     Running   0          60s

NAME                                         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
service/korifi-api-svc                       ClusterIP   10.233.17.37    <none>        443/TCP   60s
service/korifi-controllers-webhook-service   ClusterIP   10.233.13.164   <none>        443/TCP   60s

NAME                                                    READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/korifi-api-deployment                   1/1     1            1           60s
deployment.apps/korifi-controllers-controller-manager   1/1     1            1           60s

NAME                                                               DESIRED   CURRENT   READY   AGE
replicaset.apps/korifi-api-deployment-7f4fccbb84                   1         1         1       60s
replicaset.apps/korifi-controllers-controller-manager-76986974d6   1         1         1       60s
```

<br>

## <div id='2.8'> 2.8. Sidecar 로그인 및 테스트 앱 배포
- 테스트 앱을 배포하여 앱이 정상 배포되는지 확인한다.
- Sidecar v2.0.0-beta2 이상부터는 로그인하는 유저는 Kubernetes의 User로 로그인을 진행한다.
- 배포 자동 테스트

```
$ ./install-test.sh

............
Waiting for app test-node-app to start...

Instances starting...
Instances starting...

name:              temp-test-app
requested state:   started
routes:            temp-test-app.apps.system.domain
last uploaded:     Wed 15 Nov 05:00:28 UTC 2023
stack:             io.buildpacks.stacks.jammy
buildpacks:        

type:            web
sidecars:        
instances:       1/1
memory usage:    1024M
start command:   node "server.js"
     state     since                  cpu    memory   disk     logging      details
#0   running   2023-11-15T05:05:14Z   0.0%   0 of 0   0 of 0   0/s of 0/s   
==============================
check output 'Hello World'
Hello World
==============================
Deleting app temp-test-app in org temp-test-org / space temp-test-space as sidecar-admin...
OK

Deleting org temp-test-org as sidecar-admin...
OK
```
  
- 배포 수동 테스트

```
$ export KUBECONFIG=/home/ubuntu/sidecar-deployment/install-scripts/support-files/user/sidecar-$(grep admin_username ./variables.yml | cut -d "=" -f 2 | cut -d " " -f1).ua.kubeconfig 
$ cf login -a api.$(grep system_domain ./variables.yml | cut -d"=" -f2 | cut -d" " -f1) --skip-ssl-validation -u $(grep admin_username ./variables.yml | cut -d "=" -f 2 | cut -d " " -f1)
  
API endpoint: api.system.domain

Authenticating...
OK

API endpoint:   https://api.system.domain
API version:    3.117.0+cf-k8s
user:           sidecar-admin
No org or space targeted, use 'cf target -o ORG -s SPACE'
```
```
$ cf create-org temp-test-org
  
Creating org temp-test-org as sidecar-admin...
OK

TIP: Use 'cf target -o "temp-test-org"' to target new org
```
```
$ cf create-space temp-test-space -o temp-test-org
  
Creating space temp-test-space in org temp-test-org as sidecar-admin...
OK

Assigning role SpaceManager to user sidecar-admin in org temp-test-org / space temp-test-space as sidecar-admin...
OK

Assigning role SpaceDeveloper to user sidecar-admin in org temp-test-org / space temp-test-space as sidecar-admin...
OK

TIP: Use 'cf target -o "temp-test-org" -s "temp-test-space"' to target new space
```
```
$ cf target -o temp-test-org -s temp-test-space
API endpoint:   https://api.system.domain
API version:    3.117.0+cf-k8s
user:           sidecar-admin
org:            temp-test-org
space:          temp-test-space
```
```
$ cf push -p ./support-files/sample-app/ temp-test-app
  
Pushing app temp-test-app to org temp-test-org / space temp-test-space as sidecar-admin...
Packaging files to upload...
Uploading files...
558 B / 558 B [============================================================] 100.00% 1s

Waiting for API to complete processing files...
.......
.......
Build successful

Waiting for app test-node-app to start...

Instances starting...
Instances starting...

name:              temp-test-app
requested state:   started
routes:            temp-test-app.apps.system.domain
last uploaded:     Wed 15 Nov 05:00:28 UTC 2023
stack:             io.buildpacks.stacks.jammy
buildpacks:        

type:            web
sidecars:        
instances:       1/1
memory usage:    1024M
start command:   node "server.js"
     state     since                  cpu    memory   disk     logging      details
#0   running   2023-11-15T05:05:14Z   0.0%   0 of 0   0 of 0   0/s of 0/s   
```
```
$ curl -k https://temp-test-app.apps.system.domain
Hello World
```

### <div id='2.8.1'> ※ (참고) Sidecar 삭제
```
$ source delete-sidecar.sh
```

### <div id='2.8.2'> ※ (참고) Sidecar Dependency 삭제 (Sidecar 삭제 후)
```
$ source delete-dependency.sh
```

<br>

## <div id='2.9'> 2.9. Sidecar User 생성
- 운영자가 kubeconfig 파일를 생성하여 권한을 설정 한 후, 유저에게 해당 kubeconfig 파일을 전달한다.
### <div id='2.9.1'> 2.9.1. Sidecar User Account 생성
```
# kubernets admin 권한으로 진행
$ cd ~/sidecar-deployment/install-scripts/support-files/user
$ ./create-new-ua.sh <생성할 username>
# create-new-ua.sh 실행 시 해당 디렉토리에 kubeconfig 파일을 생성함
```
```
# Sidecar admin 권한으로 Sidecar 로그인 하여 진행
$ cf set-space-role <생성한 username> <권한을 줄 ORG이름> <권한을 줄 SPACE이름> SpaceDeveloper
```
### <div id='2.9.2'> 2.9.2. Sidecar Service Account 생성
- Service Account로 유저를 생성 시 cf set-space-role 명령어로 권한을 줄 수 없기 때문에, 스크립트를 통하여 Space에 접근할수있는 Rolebinding 권한을 부여한다.
```
# kubernets admin 권한으로 진행
$ cd ~/sidecar-deployment/install-scripts/support-files/user
$ ./create-new-sa.sh <생성할 username>
# create-new-sa.sh 실행 시 해당 디렉토리에 kubeconfig 파일을 생성함
```
```
# create-new-sa.sh를 통해 Service Account를 생성할 경우,
# Service Account namespace는 Sidecar를 배포할 시의 root_namespace 변수값과 동일하다.
$ ./binding-sa.sh <Service Account namespace> <생성한 username> <권한을 줄 ORG이름> <권한을 줄 SPACE이름>
```

#### <div id='2.9.2.1'> ※ (참고) Container Platform Portal 계정을 사용하여 Sidecar 접속
- Container Platform Portal 유저가 사용할 Namespace와 User ID를 운영자에게 전달하여, 운영자가 권한을 부여하여 Sidecar 접속이 가능하다.
- 운영자는 User의 Service Account 정보를 확인하여 다음과 같이 진행한다.
- ※ Service Account 확인방법 (Container Platform Portal > Dashboard > Managements > Users > > User > 해당 ID 클릭 > Services Account  확인)
```
# kubernets admin 권한으로 진행
$ cd ~/sidecar-deployment/install-scripts/support-files/user
$ ./binding-sa.sh <Service Account namespace> <확인 한 Service Account> <권한을 줄 ORG이름> <권한을 줄 SPACE이름>
```
### <div id='2.9.3'> 2.9.3. Sidecar Admin 권한 부여
```
# kubernets admin 권한으로 진행
$ cd ~/sidecar-deployment/install-scripts/support-files/user
$ ./binding-admin.sh <user 종류 (ua, sa)> <username>
```

<br>

## <div id='2.10'> ※ (참고) Container Platform Portal Harbor를 활용한 Sidecar 설치
- 아래는 Container Platform Portal이 설치 되어있다면, 구축되어있는 Harbor를 통해 Sidecar 설치하는 예제를 안내한다.
### <div id='2.10.1'> Harbor Project 생성
```
# Harbor 환경변수 등록 (CP-PORTAL SCRIPT 위치)
$ source ~/workspace/container-platform/cp-portal-deployment/script/cp-portal-vars.sh

# Harbor Project 명 입력
$ export SIDECAR_HARBOR_PROJECT=sidecar

# Harbor Project 생성
$ curl -u $REPOSITORY_USERNAME:$REPOSITORY_PASSWORD -k $REPOSITORY_URL/api/v2.0/projects -XPOST --data-binary "{\"project_name\": \"$SIDECAR_HARBOR_PROJECT\", \"public\": false}" -H "Content-Type: application/json" -i

```
### <div id='2.10.2'> Harbor CA 확인 및 Cert Injection
```
$ ll /usr/local/share/ca-certificates/cp-harbor-ca.crt
total 12
drwxr-xr-x 2 root   root   4096 Jan  4  2024 ./
drwxr-xr-x 5 root   root   4096 Jan  4  2024 ../
-rw-rw-r-- 1 ubuntu ubuntu 1127 Jan  4  2024 cp-harbor-ca.crt

$ cp /usr/local/share/ca-certificates/cp-harbor-ca.crt ~/sidecar-deployment/install-scripts/support-files/private-repository.ca

# Cert Injection에 관한 variables.yml 설정
$ cd ~/sidecar-deployment/install-scripts
$ vim variables.yml

# use_dockerhub=false 수정
# registry_id 수정 (확인 : $ echo $REPOSITORY_USERNAME)
# registry_password 수정 (확인 : $ echo $REPOSITORY_PASSWORD)
# registry_address 수정 (확인 : $ echo $REPOSITORY_URL -> https:// 제외)
# registry_repositry_name 수정 (확인 : $ echo $SIDECAR_HARBOR_PROJECT)
# is_self_signed_certificate=true 수정

$ soure deploy-inject-self-signed-cert.sh
```

### <div id='2.10.3'> 설치 진행
- [네임스페이스 생성 & 레지스트리 정보 입력](#2.5) 부터 가이드를 확인하여 설치를 진행한다.


### [Index](https://github.com/K-PaaS/Guide/blob/master/README.md) > [K-PaaS Sidecar Install](./README.md) > Sidecar
