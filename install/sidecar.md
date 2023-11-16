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
  　2.9.3. [Sidecar Admin 권한 부여](#2.9.3)  

<br><br>

# <div id='1'> 1. 문서 개요
## <div id='1.1'> 1.1. 목적
본 문서는 K-PaaS Container-Platform 단독 배포 환경에서 K-PaaS Sidecar(이하 Sidecar)를 설치하기 위한 가이드를 제공하는 데 목적이 있다.

<br>

## <div id='1.2'> 1.2. 범위
본 문서는 [korifi v0.10.0](https://github.com/cloudfoundry/korifi/tree/v0.10.0), [sidecar-deployment v2.0.0-beta](https://github.com/K-PaaS/sidecar-deployment/tree/v2.0.0-beta), [cp-deployment v1.5.0](https://github.com/k-paas/cp-deployment/tree/v1.5.0)을 기준으로 작성하였다.    
본 문서는 K-PaaS Container-Platform 단독 배포(Kubespray)를 활용하여 Kubernetes Cluster를 구성 후 Sidecar 설치 기준으로 작성하였다.  
본 문서는 IaaS, Kubernetes에 대한 기본 이해도가 있다는 전제하에 가이드를 진행하였다.  

<br>


## <div id='1.3'> 1.3. 참고자료
K-PaaS 컨테이너 플랫폼 : [https://github.com/K-PaaS/container-platform](https://github.com/k-paas/cp-deployment)  
Kubespray : [https://kubespray.io](https://kubespray.io)  
Kubespray github : [https://github.com/kubernetes-sigs/kubespray](https://github.com/kubernetes-sigs/kubespray)  
korifi github : [https://github.com/cloudfoundry/korifi](https://github.com/cloudfoundry/korifi)  

<br>

# <div id='2'> 2. K-PaaS Sidecar 설치

## <div id='2.1'> 2.1. Prerequisite
- Default StorageClass 지정
- OCI 호환 이미지 레지스트리 제공 (e.g. [Docker Hub](https://hub.docker.com/), [Google container registry](https://cloud.google.com/container-registry),  [Azure container registry](https://hub.docker.com/), [Harbor](https://goharbor.io/), etc....)  

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

- git clone 명령을 통해 다음 경로에서 Sidecar 다운로드를 진행한다. 본 설치 가이드에서의 Sidecar의 버전은 v2.0.0-beta 버전이다.

  ```
  $ cd $HOME
  $ git clone https://github.com/K-PaaS/sidecar-deployment.git -b v2.0.0-beta
  $ cd sidecar-deployment/install-scripts
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

  ## dependency variable
  use_lb=true                                                  # (e.g. true or false)
  lb_ip=                                                       # if k8s support loadBalancerIP ==> ip input (e.g. 23.45.23.45), k8s not support loadBalancerIP ==> blank

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
  | use_lb | true 시 LoadBalancer, false 시 NodePort 배포 |
  | lb_ip | LoadBalancerIP가 지원되는 K8S일 시 입력, 미 지원 시 공백 |
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

## <div id='2.6'> 2.6. Sidecar Dependency 배포
- Sidecar를 실행 시 필요한 패키지를 설치한다.

  ```
  $ source 2.deploy-dependency.sh

  ............

  ====================cert-manager====================

  NAME                                           READY   STATUS    RESTARTS   AGE
  pod/cert-manager-75d57c8d4b-9xrvz              1/1     Running   0          21h
  pod/cert-manager-cainjector-69d6f4d488-l6464   1/1     Running   0          21h
  pod/cert-manager-webhook-869b6c65c4-rv9rf      1/1     Running   0          21h
  
  NAME                           TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
  service/cert-manager           ClusterIP   10.233.14.159   <none>        9402/TCP   21h
  service/cert-manager-webhook   ClusterIP   10.233.46.192   <none>        443/TCP    21h
  
  NAME                                      READY   UP-TO-DATE   AVAILABLE   AGE
  deployment.apps/cert-manager              1/1     1            1           21h
  deployment.apps/cert-manager-cainjector   1/1     1            1           21h
  deployment.apps/cert-manager-webhook      1/1     1            1           21h
  
  NAME                                                 DESIRED   CURRENT   READY   AGE
  replicaset.apps/cert-manager-75d57c8d4b              1         1         1       21h
  replicaset.apps/cert-manager-cainjector-69d6f4d488   1         1         1       21h
  replicaset.apps/cert-manager-webhook-869b6c65c4      1         1         1       21h
  
  
  ======================contour=======================
  
  NAME                                READY   STATUS      RESTARTS   AGE
  pod/contour-7f56bcc895-9p9hg        1/1     Running     0          21h
  pod/contour-7f56bcc895-rhqzp        1/1     Running     0          21h
  pod/contour-certgen-v1-26-0-2zstl   0/1     Completed   0          21h
  pod/envoy-5kfzm                     2/2     Running     0          21h
  pod/envoy-6nvqw                     2/2     Running     0          21h
  pod/envoy-bkddj                     2/2     Running     0          21h
  pod/envoy-m78x7                     2/2     Running     0          21h
  
  NAME              TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
  service/contour   ClusterIP   10.233.63.8     <none>        8001/TCP                     21h
  service/envoy     NodePort    10.233.61.191   <none>        80:32330/TCP,443:30781/TCP   21h
  
  NAME                   DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
  daemonset.apps/envoy   4         4         4       4            4           <none>          21h
  
  NAME                      READY   UP-TO-DATE   AVAILABLE   AGE
  deployment.apps/contour   2/2     2            2           21h
  
  NAME                                 DESIRED   CURRENT   READY   AGE
  replicaset.apps/contour-7f56bcc895   2         2         2       21h
  
  NAME                                COMPLETIONS   DURATION   AGE
  job.batch/contour-certgen-v1-26-0   1/1           5s         21h
  
  
  ======================kpack=========================
  
  NAME                                    READY   STATUS    RESTARTS   AGE
  pod/kpack-controller-7d7f477784-gjhx4   1/1     Running   0          21h
  pod/kpack-webhook-848896f7c7-hfsnx      1/1     Running   0          21h
  
  NAME                    TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
  service/kpack-webhook   ClusterIP   10.233.31.79   <none>        443/TCP   21h
  
  NAME                               READY   UP-TO-DATE   AVAILABLE   AGE
  deployment.apps/kpack-controller   1/1     1            1           21h
  deployment.apps/kpack-webhook      1/1     1            1           21h
  
  NAME                                          DESIRED   CURRENT   READY   AGE
  replicaset.apps/kpack-controller-7d7f477784   1         1         1       21h
  replicaset.apps/kpack-webhook-848896f7c7      1         1         1       21h
  
  
  ===================service-binding==================
  
  NAME                                                    READY   STATUS    RESTARTS   AGE
  pod/servicebinding-controller-manager-dff969cdc-tdbgg   2/2     Running   0          21h
  
  NAME                                                        TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
  service/servicebinding-controller-manager-metrics-service   ClusterIP   10.233.37.65    <none>        8443/TCP   21h
  service/servicebinding-webhook-service                      ClusterIP   10.233.30.136   <none>        443/TCP    21h
  
  NAME                                                READY   UP-TO-DATE   AVAILABLE   AGE
  deployment.apps/servicebinding-controller-manager   1/1     1            1           21h
  
  NAME                                                          DESIRED   CURRENT   READY   AGE
  replicaset.apps/servicebinding-controller-manager-dff969cdc   1         1         1       21h


  ```
<br>

## <div id='2.7'> 2.7. Sidecar 설치
- Sidecar를 설치한다.

  ```
  $ source 3.deploy-sidecar.sh
  
  Release "sidecar" has been upgraded. Happy Helming!
  NAME: sidecar
  LAST DEPLOYED: Wed Nov 15 04:57:12 2023
  NAMESPACE: sidecar
  STATUS: deployed
  REVISION: 2
  TEST SUITE: None
  create admin
  certificatesigningrequest.certificates.k8s.io/694d3401ce46a074381d50a7dc1baf4c1e8b9a22 created
  certificatesigningrequest.certificates.k8s.io/694d3401ce46a074381d50a7dc1baf4c1e8b9a22 approved
  certificatesigningrequest.certificates.k8s.io/694d3401ce46a074381d50a7dc1baf4c1e8b9a22 condition met
  certificatesigningrequest.certificates.k8s.io "694d3401ce46a074381d50a7dc1baf4c1e8b9a22" deleted
  Cluster "cluster1" set.
  User "sidecar-admin" set.
  Context "sidecar-admin" modified.
  Switched to context "sidecar-admin".
  kubeconfig file : /home/ubuntu/sidecar-deployment/install-scripts/support-files/user/sidecar-sidecar-admin.ua.kubeconfig
  
  Use "cf set-space-role sidecar-admin ORG SPACE SpaceDeveloper" to grant this user permissions in a space.
  NAME                                                         READY   STATUS    RESTARTS   AGE
  pod/korifi-api-deployment-84555d64d5-9x5n2                   1/1     Running   0          2m36s
  pod/korifi-controllers-controller-manager-69996595fb-w42qs   1/1     Running   0          2m36s
  
  NAME                                         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
  service/korifi-api-svc                       ClusterIP   10.233.33.57    <none>        443/TCP   2m36s
  service/korifi-controllers-webhook-service   ClusterIP   10.233.42.150   <none>        443/TCP   2m36s
  
  NAME                                                    READY   UP-TO-DATE   AVAILABLE   AGE
  deployment.apps/korifi-api-deployment                   1/1     1            1           2m36s
  deployment.apps/korifi-controllers-controller-manager   1/1     1            1           2m36s
  
  NAME                                                               DESIRED   CURRENT   READY   AGE
  replicaset.apps/korifi-api-deployment-84555d64d5                   1         1         1       2m36s
  replicaset.apps/korifi-controllers-controller-manager-69996595fb   1         1         1       2m36s
  ```


## <div id='2.8'> 2.8. Sidecar 로그인 및 테스트 앱 배포
- 테스트 앱을 배포하여 앱이 정상 배포되는지 확인한다.
- Sidecar v2.0.0-beta 이상부터는 로그인하는 유저는 Kubernetes의 User로 로그인을 진행한다.
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
  $ KUBECONFIG=/home/ubuntu/sidecar-deployment/install-scripts/support-files/user/sidecar-$(grep admin_username ./variables.yml | cut -d "=" -f 2 | cut -d " " -f1).ua.kubeconfig 
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

<br>
  
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
- 운영자가 kubeconfig를 생성하여 권한을 설정 한 후, 유저에게 해당 kubeconfig을 전달한다.
### <div id='2.9.1'> 2.9.1. Sidecar User Account 생성
```
# kubernets admin 권한으로 진행
$ cd ~/sidecar-deployment/install-scripts/support-files/user
$ ./create-new-ua.sh <생성할 username>

# create-new-ua.sh 실행 시 해당 디렉토리에 kubeconfig 파일을 생성함
# 이하의 커맨드를 Sidecar admin 권한으로 Sidecar 로그인 하여 진행
$ cf set-space-role <생성한 username> <권한을 줄 ORG이름> <권한을 줄 SPACE이름> SpaceDeveloper
```
### <div id='2.9.2'> 2.9.2. Sidecar Service Account 생성
- Service Account로 유저를 생성 시 cf set-space-role 명령어로 권한을 줄 수 없기 때문에, 스크립트를 통하여 Space에 접근할수있는 Rolebinding 권한을 부여한다.
```
# kubernets admin 권한으로 진행
$ cd ~/sidecar-deployment/install-scripts/support-files/user
$ ./create-new-sa.sh <생성할 username>

# create-new-sa.sh 실행 시 해당 디렉토리에 kubeconfig 파일을 생성함
$ ./binding-sa.sh <생성한 username> <권한을 줄 ORG이름> <권한을 줄 SPACE이름>
```
### <div id='2.9.3'> 2.9.3. Sidecar Admin 권한 부여
```
# kubernets admin 권한으로 진행
$ cd ~/sidecar-deployment/install-scripts/support-files/user
$ ./binding-admin.sh <user 종류 (ua, sa)> <username>
```
  
### [Index](https://github.com/K-PaaS/Guide/blob/master/README.md) > [K-PaaS Sidecar Install](./README.md) > Sidecar
