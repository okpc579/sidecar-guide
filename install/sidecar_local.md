### [Index](https://github.com/K-PaaS/Guide/blob/master/README.md) > [K-PaaS Sidecar Install](./README.md) > Sidecar - local

## Table of Contents

1. [문서 개요](#1)  
  1.1. [목적](#1.1)  
  1.2. [범위](#1.2)  
  1.3. [참고자료](#1.3)  

2. [K-PaaS Sidecar - local 설치](#2)  
  2.1. [Prerequisite](#2.1)  
  2.2. [설치 파일 다운로드](#2.2)  
  2.3. [실행 파일 소개 및 설치](#2.3)  
  2.4. [Local Kubernetes Cluster 구성](#2.4)  
  　2.4.1 [kind](#2.4.1)  
  　2.4.2 [minikube](#2.4.2)  
  2.5. [Sidecar 설치](#2.5)  
  　2.5.1 [kind](#2.5.1)  
  　2.5.2 [minikube](#2.5.2)  

# <div id='1'> 1. 문서 개요
## <div id='1.1'> 1.1. 목적
본 문서는 Local Kubenetes Cluster를 구성하고 해당 환경에서 K-PaaS Sidecar(이하 Sidecar)를 설치하기 위한 가이드를 제공하는 데 목적이 있다.

<br>

## <div id='1.2'> 1.2. 범위
본 문서는 [sidecar-deployment v2.0.0-beta](https://github.com/K-PaaS/sidecar-deployment/tree/v2.0.0-beta)를 기준으로 작성하였다.  
본 문서는 [kind](https://kind.sigs.k8s.io/)로 Local Kubernetes Cluster를 구성 후 Sidecar 설치 기준으로 작성하였다.

<br>

## <div id='1.3'> 1.3. 참고자료
korifi github : [https://github.com/cloudfoundry/korifi](https://github.com/cloudfoundry/korifi)  
kind Document : [https://kind.sigs.k8s.io/](https://kind.sigs.k8s.io/)  

<br>

# <div id='2'> 2. K-PaaS Sidecar - local 설치
## <div id='2.1'> 2.1. Prerequisite
- 4 CPU, 8GB Memory 이상 권장

<br>

## <div id='2.2'> 2.2. 설치 파일 다운로드

- git clone 명령을 통해 다음 경로에서 Sidecar 다운로드를 진행한다. 본 설치 가이드에서의 Sidecar의 버전은 v1.0.3.1 버전이다.
```
$ cd $HOME
$ git clone https://github.com/K-PaaS/sidecar-deployment.git -b v2.0.0-beta
$ cd sidecar-deployment/install-scripts/local
```

<br>

## <div id='2.3'> 2.3. 실행 파일 소개 및 설치

- Sidecar를 설치 & 활용하기 위해선 다음과 같은 실행파일이 필요하다.

| 이름   |      설명      |
|----------|-------------|
| [docker](https://www.docker.com/) | 컨테이너 기반의 가상화 플랫폼 |
| [kind](https://kind.sigs.k8s.io/) | 로컬 Kubernetes cluster |
| [kubectl](https://github.com/kubernetes/kubectl) | Kubernetes Cluster를 제어하는 툴 |
| [cf cli](https://github.com/cloudfoundry/cli) (v8.5+) | Sidecar와 상호 작용하는 툴 |

- util 설치
```
$ source utils-install.sh
```

<br>

## <div id='2.4'> 2.4. Local Kubernetes Cluster 구성
본 가이드에서 제공되는 cluster 구성 도구 kind로 진행한다.  
### <div id='2.4.1'> 2.4.1. kind

- cluster 생성
```
$ ./deploy-kind.sh
```

<br>

## <div id='2.5'> 2.5. Sidecar 설치
### <div id='2.5.1'> 2.5.1. kind

- Sidecar 배포 스크립트를 실행한다.
```
$ ./deploy-sidecar-kind.sh

namespace/sidecar-installer created
namespace/kpaas created
namespace/sidecar created
secret/image-registry-credentials created
serviceaccount/sidecar-installer created
clusterrolebinding.rbac.authorization.k8s.io/sidecar-installer created
job.batch/install-sidecar created
track the job progress command : 'kubectl -n sidecar-installer logs --follow job/install-sidecar'
```

- 해당 커맨드를 이용하여 배포되는 로그를 확인한다
```
$ kubectl -n sidecar-installer logs --follow job/install-sidecar
```

- Sidecar가 정상설치 되었는지 샘플앱을 통해 확인한다.

```
$ ./install-test.sh

Setting API endpoint to https://localhost...
OK

API endpoint:   https://localhost
API version:    3.117.0+cf-k8s

Not logged in. Use 'cf login' or 'cf login --sso' to log in.
API endpoint: https://localhost

Authenticating...
OK

Use 'cf target' to view or set your target org and space.
Creating org org as kubernetes-admin...
OK

TIP: Use 'cf target -o "org"' to target new org
Warning: The client certificate you provided for user authentication expires at 2024-11-13T07:25:46Z
which exceeds the recommended validity duration of 168h0m0s. Ask your platform provider to issue you a short-lived certificate credential or to configure your authentication to generate short-lived credentials automatically.
Creating space space in org org as kubernetes-admin...
OK

Assigning role SpaceManager to user kubernetes-admin in org org / space space as kubernetes-admin...
OK

Assigning role SpaceDeveloper to user kubernetes-admin in org org / space space as kubernetes-admin...
OK

TIP: Use 'cf target -o "org" -s "space"' to target new space
Warning: The client certificate you provided for user authentication expires at 2024-11-13T07:25:46Z
which exceeds the recommended validity duration of 168h0m0s. Ask your platform provider to issue you a short-lived certificate credential or to configure your authentication to generate short-lived credentials automatically.
API endpoint:   https://localhost
API version:    3.117.0+cf-k8s
user:           kubernetes-admin
org:            org
space:          space
Pushing app sample-app to org org / space space as kubernetes-admin...
Packaging files to upload...
Uploading files...
 558 B / 558 B [================================================================================================================================================================================] 100.00% 1s

Waiting for API to complete processing files...

Staging app and tracing logs...
   
   Build reason(s): CONFIG
   CONFIG:
    + env:
    + - name: VCAP_SERVICES
    +   valueFrom:
    +     secretKeyRef:
    +       key: VCAP_SERVICES
    +       name: 6b75e061-73d9-46a0-8bc9-85105476d790-vcap-services
    + - name: VCAP_APPLICATION
    +   valueFrom:
    +     secretKeyRef:
    +       key: VCAP_APPLICATION
    +       name: 6b75e061-73d9-46a0-8bc9-85105476d790-vcap-application
    resources: {}
    - source: {}
    + source:
    +   registry:
    +     image: localregistry-docker-registry.default.svc.cluster.local:30050/6b75e061-73d9-46a0-8bc9-85105476d790-packages@sha256:0d83fdc9f5e888e20bb3507c670cebd06f1f9bf0f5c5628bd5b6274bbd4ea54c
    +     imagePullSecrets:
    +     - name: image-registry-credentials
   Loading registry credentials from service account secrets
   Loading secret for "localregistry-docker-registry.default.svc.cluster.local:30050" from secret "image-registry-credentials" at location "/var/build-secrets/image-registry-credentials"
   Loading cluster credential helpers
   Pulling localregistry-docker-registry.default.svc.cluster.local:30050/6b75e061-73d9-46a0-8bc9-85105476d790-packages@sha256:0d83fdc9f5e888e20bb3507c670cebd06f1f9bf0f5c5628bd5b6274bbd4ea54c...
   Successfully pulled localregistry-docker-registry.default.svc.cluster.local:30050/6b75e061-73d9-46a0-8bc9-85105476d790-packages@sha256:0d83fdc9f5e888e20bb3507c670cebd06f1f9bf0f5c5628bd5b6274bbd4ea54c in path "/workspace"
   Timer: Analyzer started at 2023-11-14T07:45:12Z
   Image with name "localregistry-docker-registry.default.svc.cluster.local:30050/6b75e061-73d9-46a0-8bc9-85105476d790-droplets" not found
   Timer: Analyzer ran for 689.315685ms and ended at 2023-11-14T07:45:13Z
   Timer: Detector started at 2023-11-14T07:45:17Z
   ======== Output: paketo-buildpacks/node-run-script@1.0.14 ========
   could not find script(s) [build] in package.json
   err:  paketo-buildpacks/node-run-script@1.0.14 (1)
   ======== Output: paketo-buildpacks/node-run-script@1.0.14 ========
   could not find script(s) [build] in package.json
   err:  paketo-buildpacks/node-run-script@1.0.14 (1)
   4 of 11 buildpacks participating
   paketo-buildpacks/ca-certificates 3.6.6
   paketo-buildpacks/node-engine     3.0.1
   paketo-buildpacks/npm-install     1.3.1
   paketo-buildpacks/node-start      1.1.3
   Timer: Detector ran for 116.163281ms and ended at 2023-11-14T07:45:17Z
   Timer: Restorer started at 2023-11-14T07:45:18Z
   Timer: Restorer ran for 933.086µs and ended at 2023-11-14T07:45:18Z
   Timer: Builder started at 2023-11-14T07:45:19Z
   
   Paketo Buildpack for CA Certificates 3.6.6
     https://github.com/paketo-buildpacks/ca-certificates
     Launch Helper: Contributing to layer
       Creating /layers/paketo-buildpacks_ca-certificates/helper/exec.d/ca-certificates-helper
   Paketo Buildpack for Node Engine 3.0.1
     Resolving Node Engine version
       Candidate version sources (in priority order):
                   -> ""
         <unknown> -> ""
   
       Selected Node Engine version (using ): 20.9.0
   
     Executing build process
       Installing Node Engine 20.9.0
         Completed in 5.895s
   
     Generating SBOM for /layers/paketo-buildpacks_node-engine/node
         Completed in 0s
   
     Configuring build environment
       NODE_ENV     -> "production"
       NODE_HOME    -> "/layers/paketo-buildpacks_node-engine/node"
       NODE_OPTIONS -> "--use-openssl-ca"
       NODE_VERBOSE -> "false"
   
     Configuring launch environment
       NODE_ENV     -> "production"
       NODE_HOME    -> "/layers/paketo-buildpacks_node-engine/node"
       NODE_OPTIONS -> "--use-openssl-ca"
       NODE_VERBOSE -> "false"
   
       Writing exec.d/0-optimize-memory
         Calculates available memory based on container limits at launch time.
         Made available in the MEMORY_AVAILABLE environment variable.
   
   Paketo Buildpack for NPM Install 1.3.1
     Resolving installation process
       Process inputs:
         node_modules      -> "Not found"
         npm-cache         -> "Not found"
         package-lock.json -> "Not found"
   
       Selected NPM build process: 'npm install'
   
     Executing launch environment install process
       Running 'npm install --unsafe-perm --cache /layers/paketo-buildpacks_npm-install/npm-cache'
         
         up to date, audited 1 package in 244ms
         
         found 0 vulnerabilities
         Completed in 506ms
   
     Configuring launch environment
       NODE_PROJECT_PATH   -> "/workspace"
       NPM_CONFIG_LOGLEVEL -> "error"
       PATH                -> "$PATH:/layers/paketo-buildpacks_npm-install/launch-modules/node_modules/.bin"
   
     Generating SBOM for /layers/paketo-buildpacks_npm-install/launch-modules
         Completed in 7ms
   
   
   Paketo Buildpack for Node Start 1.1.3
     Assigning launch processes:
       web (default): node server.js
   
   Timer: Builder ran for 6.493812331s and ended at 2023-11-14T07:45:25Z
   Timer: Exporter started at 2023-11-14T07:45:30Z
   Adding layer 'paketo-buildpacks/ca-certificates:helper'
   Adding layer 'paketo-buildpacks/node-engine:node'
   Adding layer 'paketo-buildpacks/npm-install:launch-modules'
   Adding layer 'buildpacksio/lifecycle:launch.sbom'
   Adding 1/1 app layer(s)
   Adding layer 'buildpacksio/lifecycle:launcher'
   Adding layer 'buildpacksio/lifecycle:config'
   Adding layer 'buildpacksio/lifecycle:process-types'
   Adding label 'io.buildpacks.lifecycle.metadata'
   Adding label 'io.buildpacks.build.metadata'
   Adding label 'io.buildpacks.project.metadata'
   Setting default process type 'web'
   Timer: Saving localregistry-docker-registry.default.svc.cluster.local:30050/6b75e061-73d9-46a0-8bc9-85105476d790-droplets... started at 2023-11-14T07:45:34Z
   *** Images (sha256:0b8753283ed9b101b5f43b7c9b784053e1f5e29a3ccdc8b8c50487657fdd956a):
         localregistry-docker-registry.default.svc.cluster.local:30050/6b75e061-73d9-46a0-8bc9-85105476d790-droplets
         localregistry-docker-registry.default.svc.cluster.local:30050/6b75e061-73d9-46a0-8bc9-85105476d790-droplets:b1.20231114.074443
   Timer: Saving localregistry-docker-registry.default.svc.cluster.local:30050/6b75e061-73d9-46a0-8bc9-85105476d790-droplets... ran for 2.644458794s and ended at 2023-11-14T07:45:36Z
   Timer: Exporter ran for 6.680619598s and ended at 2023-11-14T07:45:36Z
   Timer: Cache started at 2023-11-14T07:45:36Z
   Adding cache layer 'paketo-buildpacks/node-engine:node'
   Adding cache layer 'paketo-buildpacks/npm-install:npm-cache'
   Adding cache layer 'buildpacksio/lifecycle:cache.sbom'
   Timer: Cache ran for 177.494357ms and ended at 2023-11-14T07:45:36Z
   Build successful

Waiting for app sample-app to start...

Instances starting...
Instances starting...
Instances starting...
Instances starting...

name:              sample-app
requested state:   started
routes:            sample-app.apps-127-0-0-1.nip.io
last uploaded:     Tue 14 Nov 07:44:43 UTC 2023
stack:             io.buildpacks.stacks.jammy
buildpacks:        

type:            web
sidecars:        
instances:       1/1
memory usage:    1024M
start command:   node "server.js"
     state     since                  cpu    memory   disk     logging      details
#0   running   2023-11-14T07:45:59Z   0.0%   0 of 0   0 of 0   0/s of 0/s   
Hello World

```

- (참고) kind cluster 삭제
```
$ ./delete-kind.sh
```

<br>

### [Index](https://github.com/K-PaaS/Guide/blob/master/README.md) > [K-PaaS Sidecar Install](./README.md) > Sidecar - local
