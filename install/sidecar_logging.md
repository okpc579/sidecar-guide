### [Index](https://github.com/PaaS-TA/Guide/blob/master/README.md) > [PaaS-TA Sidecar Install](./README.md) > Logging Service

## Table of Contents

1. [문서 개요](#1)  
  1.1. [목적](#1.1)  
  1.2. [범위](#1.2)  
  1.3. [참고자료](#1.3)  

2. [Logging 서비스 설치](#2)  
  2.1. [Prerequisite](#2.1)  
  2.2. [Stemcell 확인](#2.2)  
  2.3. [Deployment 다운로드](#2.3)  
  2.4. [Deployment 파일 수정](#2.4)  
  2.5. [서비스 설치](#2.5)  
  2.6. [서비스 설치 확인](#2.6)  
  2.7. [Sidecar 설정 변경](#2.7)  
  2.8. [Sidecar 설정 변경 확인](#2.8)  

3. [Logging 서비스 관리](#3)  
  3.1. [보안 그룹 적용](#3.1)  
  3.2. [Logging 서비스 활성화](#3.2)

# <div id='1'> 1. 문서 개요
## <div id='1.1'> 1.1. 목적
본 문서는 PaaS-TA Sidecar(이하 Sidecar) 환경에서 Logging 서비스를 사용하기 위한 가이드를 제공하는 데 목적이 있다.

<br>

## <div id='1.2'> 1.2. 범위
본 문서는 Logging 서비스를 검증하기 위한 기본 설치를 기준으로 작성하였다.

<br>


## <div id='1.3'> 1.3. 참고자료
BOSH Document: [http://bosh.io](http://bosh.io)  
cf-for-k8s github : [https://github.com/cloudfoundry/cf-for-k8s](https://github.com/cloudfoundry/cf-for-k8s)  
cf-for-k8s Document : [https://cf-for-k8s.io/docs/](https://cf-for-k8s.io/docs/)  
cf-k8s-logging github : [https://github.com/cloudfoundry/cf-k8s-logging](https://github.com/cloudfoundry/cf-k8s-logging)

<br>

# <div id='2'> 2. Logging 서비스 설치
## <div id='2.1'> 2.1. Prerequisite
서비스 설치를 위해서 BOSH 설치가 사전에 진행되어야 한다.  
BOSH 설치는 아래 가이드를 참고한다.
> [BOSH 설치](https://github.com/PaaS-TA/application-platform-guide/blob/master/install/application_platform/bosh.md)

<br>

## <div id='2.2'> 2.2. Stemcell 확인
Stemcell 목록을 확인하여 서비스 설치에 필요한 Stemcell이 업로드 되어 있는 것을 확인한다.  
본 가이드의 Stemcell은 ubuntu-bionic 1.195를 사용한다.

> $ bosh -e ${BOSH_ENVIRONMENT} stemcells

```
Using environment '10.0.1.6' as client 'admin'

Name                                     Version  OS             CPI  CID
bosh-vsphere-esxi-ubuntu-bionic-go_agent  1.195*  ubuntu-bionic  -    sc-c9eeb237-f344-4396-ab29-90bfab2b6a75

(*) Currently deployed

1 stemcells

Succeeded
```

만약 해당 Stemcell이 업로드 되어 있지 않다면 [bosh.io 스템셀](https://bosh.io/stemcells/) 에서 해당되는 IaaS환경과 버전에 해당되는 스템셀 링크를 복사 후 다음과 같은 명령어를 실행한다.

```
# Stemcell 업로드 명령어 예제
bosh -e ${BOSH_ENVIRONMENT} upload-stemcell -n {STEMCELL_URL}
```

<br>

### <div id="2.3"/> 2.3. Deployment 다운로드
서비스 설치에 필요한 Deployment를 Git Repository에서 받아 서비스 설치 작업 경로로 위치시킨다.

- Service Deployment Git Repository URL : https://github.com/PaaS-TA/service-deployment/tree/logging

```
$ cd $HOME

# Deployment 파일 다운로드
$ git clone https://github.com/PaaS-TA/service-deployment.git -b logging

# common_vars.yml 파일 다운로드
$ git clone https://github.com/PaaS-TA/common.git
```

### <div id="2.4"/> 2.4. Deployment 파일 수정
- `common_vars.yml`을 서버 환경에 맞게 수정한다.
- Logging 서비스에서 사용하는 변수는 system_domain, uaa_client_admin_id, uaa_client_admin_secret 이다.

> $ vi $HOME/common/common_vars.yml
```yaml
... ((생략)) ...

# PAAS-TA INFO
system_domain: "system_domain"		# Domain (nip.io를 사용하는 경우 HAProxy Public IP와 동일)
uaa_client_admin_id: "admin"			# UAAC Admin Client Admin ID
uaa_client_admin_secret: "zmj2sw6kwd6sedzku45f"		# UAAC Admin Client에 접근하기 위한 Secret 변수

... ((생략)) ...
```


- Deployment YAML에서 사용하는 변수 파일을 서버 환경에 맞게 수정한다.
> $ vi $HOME/service-deployment/logging-service/vars.yml

```yaml
# STEMCELL INFO
stemcell_os: "ubuntu-bionic"		# Stemcell OS
stemcell_version: "1.195"		# Stemcell Version


# VARIABLE
syslog_forwarder_custom_rule: 'if ($msg contains "DEBUG") then stop'      # PaaS-TA Logging Agent에서 전송할 Custom Rule
syslog_forwarder_fallback_servers: []
paasta_deploy_type: "sidecar"                 # PaaS-TA 배포 타입(ap, sidecar)
portal_deploy_type: "app"                     # PaaS-TA Portal 배포 타입(vm, app)


# Fluentd
fluentd_azs: ["z4"]                    # fluentd : azs
fluentd_instances: 1                   # fluentd : instances (1)
fluentd_vm_type: "small"               # fluentd : vm type
fluentd_network: "default"             # fluentd 네트워크
fluentd_ip: "<FLUENTD_IP>"
fluentd_port: "3514"                   # fluentd Port
fluentd_transport: "tcp"               # fluentd Logging Protocol


# INFLUXDB
influxdb_azs: ["z4"]			            # InfluxDB : azs
influxdb_instances: 1			            # InfluxDB : instances (1)
influxdb_vm_type: "large"		          # InfluxDB : vm type
influxdb_network: "default"		        # InfluxDB 네트워크
influxdb_persistent_disk_type: "10GB"	# InfluxDB 영구 Disk 종류

influxdb_ip: "10.0.1.115"
influxdb_http_port: "8086"                  # default 8086
influxdb_username: "admin"	  # InfluxDB Admin 계정 Username
influxdb_password: "PaaS-TA2022"	  # InfluxDB Admin 계정 Password
influxdb_interval: "7d"                     # InfluxDB Retention Policy (bootstrapper)
influxdb_https_enabled: "true"              # InfluxDB HTTPS 설정
influxdb_ssl_key_path: "/var/vcap/jobs/paas-ta-portal-log-api/data"     # InfluxDB SSL key store path
influxdb_ssl_password: "paasta2022"         # InfluxDB SSL password

influxdb_database: "logging_db"          # InfluxDB Database명
influxdb_measurement: "logging_measurement"    # InfluxDB Measurement명
influxdb_time_precision: "s"    # hour(h), minutes(m), second(s), millisecond(ms), microsecond(u), nanosecond(ns)
influxdb_query_limit: "50"                  # InfluxDB query limit (default "50")


# COLLECTOR
collector_azs: ["z4"]           # collector : azs
collector_instances: 1          # collector : instances (1)
collector_vm_type: "small"      # collector : vm type
collector_network: "default"    # collector 네트워크


# LOG_API
log_api_azs: ["z4"]                                             # log-api : azs
log_api_instances: 1                                            # log-api : instances (1)
log_api_vm_type: "small"                                        # log-api : vm type
log_api_network: "default"                                      # log-api 네트워크
```

### <div id="2.5"/> 2.5. 서비스 설치
- 서버 환경에 맞추어 Deploy 스크립트 파일의 VARIABLES 설정을 수정한다.
> $ vi $HOME/workspace/portal-deployment/logging-service/deploy.sh

```shell script
#!/bin/bash

# VARIABLES
COMMON_VARS_PATH="<COMMON_VARS_FILE_PATH>"             # common_vars.yml File Path (e.g. ../../common/common_vars.yml)
BOSH_ENVIRONMENT="${BOSH_ENVIRONMENT}"                   # bosh director alias name (PaaS-TA에서 제공되는 create-bosh-login.sh 미 사용시 bosh envs에서 이름을 확인하여 입력)


# PaaS-TA, Portal 설치 타입 및 프로토콜 종류에 따라 옵션 파일 사용 여부를 분기한다.
PAASTA_DEPLOY_TYPE=`grep paasta_deploy_type vars.yml | cut -d "#" -f1`
PORTAL_DEPLOY_TYPE=`grep portal_deploy_type vars.yml | cut -d "#" -f1`
FLUENTD_TRANSPORT=`grep fluentd_transport vars.yml`


if [[ "${PAASTA_DEPLOY_TYPE}" =~ "sidecar" ]]; then
  source operations/create-sidecar-ops.sh
  if [[ "${PORTAL_DEPLOY_TYPE}" =~ "app" ]]; then
      bosh -e ${BOSH_ENVIRONMENT} -d logging-service -n deploy logging-service.yml \
            -o operations/portal-app-type.yml \
            -o operations/paasta-sidecar-type.yml \
            -l vars.yml \
            -l ${COMMON_VARS_PATH}
  else
    echo "Logging Service can't install. Please check 'portal_deploy_type'."
  fi
elif [[ "${PAASTA_DEPLOY_TYPE}" =~ "ap" ]]; then
  if [[ "${PORTAL_DEPLOY_TYPE}" =~ "app" ]]; then
    if [[ "${FLUENTD_TRANSPORT}" =~ "tcp" ]]; then
      bosh -e ${BOSH_ENVIRONMENT} -d logging-service -n deploy logging-service.yml \
            -o operations/portal-app-type.yml \
            -o operations/use-protocol-tcp.yml \
            -l vars.yml \
            -l ${COMMON_VARS_PATH}
    else
      bosh -e ${BOSH_ENVIRONMENT} -d logging-service -n deploy logging-service.yml \
            -o operations/portal-app-type.yml \
            -l vars.yml \
            -l ${COMMON_VARS_PATH}
    fi
  elif [[ "${PORTAL_DEPLOY_TYPE}" =~ "vm" ]]; then
    if [[ "${FLUENTD_TRANSPORT}" =~ "tcp" ]]; then
      bosh -e ${BOSH_ENVIRONMENT} -d logging-service -n deploy logging-service.yml \
            -o operations/use-protocol-tcp.yml \
            -l vars.yml \
            -l ${COMMON_VARS_PATH}
    else
      bosh -e ${BOSH_ENVIRONMENT} -d logging-service -n deploy logging-service.yml \
            -l vars.yml \
            -l ${COMMON_VARS_PATH}
    fi
  else
    echo "Logging Service can't install. Please check 'portal_deploy_type'."
  fi
else
  echo "Logging Service can't install. Please check 'paasta_deploy_type'."
fi
```

- 서비스를 설치한다.
```shell
$ cd $HOME/workspace/service-deployment/logging-service
$ source deploy.sh
```


### <div id="2.6"/> 2.6. 서비스 설치 확인

설치 완료된 서비스를 확인한다.

> $ bosh -e ${BOSH_ENVIRONMENT} -d logging-service vms
```shell
Using environment '10.0.1.6' as client 'admin'

Task 193. Done

Deployment 'logging-service'

Instance                                        Process State  AZ  IPs         VM CID                                   VM Type  Active  Stemcell
log-api/0d710aae-3c0f-44b2-ac79-4a134ad2c601    running        z4  10.0.1.163  vm-d04c9c5f-55c8-44ca-b64b-57db7c60d619  small    true    bosh-vsphere-esxi-ubuntu-bionic-go_agent/1.195
influxdb/e4078a4d-b71b-4e91-8f30-85189c3ecf3f   running        z4  10.0.1.115  vm-7d3b4e7f-eb90-406e-b2a4-c16c458d4bdc  large    true    bosh-vsphere-esxi-ubuntu-bionic-go_agent/1.195

```

## <div id='2.7'> 2.7. Sidecar 설정 변경
Logging 서비스를 활성화 하기 위해서는 Sidecar 설정 중 일부를 변경해야 한다.  
Sidecar 배포 시 `variables.yml` 파일 내 Logging Service 변수 설정을 했다면 [2.8. Sidecar 설정 변경 확인](#2.8)부터 진행한다.
- `variables.yml` 파일을 편집하여 Logging Service 변수를 설정한다.
```
$ cd $HOME/sidecar-deployment/install-scripts
$ vi variables.yml

...

## LOGGING VARIABLE
use_logging_service=true                                    # (e.g. true or false)
logging_output_plugin=influxdb                              # Logging output plugin (influxdb or http) --> recommend using influxdb plugin
influxdb_ip=10.0.1.115                                      # InfluxDB IP
influxdb_http_port=8086                                     # InfluxDB Port
influxdb_username=admin                                     # InfluxDB Username
influxdb_password=PaaS-TA2022                               # InfluxDB Password
influxdb_https_enabled=true                                 # (e.g. true or false)
influxdb_database=logging_db                                # InfluxDB DB Name
influxdb_measurement=logging_measurement                    # InfluxDB Measurement Name
influxdb_time_precision=s                                   # Level of timestamp stored (hour(h), minutes(m), second(s), millisecond(ms), microsecond(u), nanosecond(ns))

```

- `enable-logging-service.sh` 파일을 실행하여 Sidecar 설정을 변경한다.
```
$ cd $HOME/sidecar-deployment/install-scripts
$ source enable-logging-service.sh
```

## <div id='2.8'> 2.8. Sidecar 설정 변경 확인
- Sidecar 설정이 정상적으로 변경되었는지 확인한다.
> $ kubectl get configmap fluentd-config-ver-1 -n cf-system -o yaml
```diff
apiVersion: v1
data:
  aggregate_drains.conf: ""
  fluentd.conf: |

...

+    # seperate process for logging-service
+    <match **>
+      @type copy
+      <store>
+        @type relabel
+        @label @SIDECAR
+      </store>
+      <store>
+        @type relabel
+        @label @LOGGING
+      </store>
+    </match>
+
+    # for sidecar
+    <label @SIDECAR>
       <match **>
         @type copy
         <store ignore_error>
           @type syslog_rfc5424
           host log-cache-syslog
           port 8082
           transport tcp
           <format>
             @type syslog_rfc5424
             proc_id_field instance_id
             app_name_field app_id
             structured_data_field structured_data
           </format>

           <buffer>
             @type memory
             flush_mode immediate
             flush_thread_count 8
           </buffer>
         </store>
         @include /fluentd/etc/aggregate_drains.conf
       </match>
+    </label>
+
+    # for logging-service
+    <label @LOGGING>
+      <filter **>
+        @type grep
+        <exclude>
+          key $.instance_id
+          pattern /^0$/
+        </exclude>
+      </filter>
+
+      <filter kubernetes.**>
+        @type record_transformer
+        remove_keys stream,docker,kubernetes
+      </filter>
+
+      <filter forwarded.**>
+        @type record_transformer
+        remove_keys source_type
+      </filter>
+
+      <filter kubernetes.** forwarded.**>
+        @type record_transformer
+        enable_ruby true
+        <record>
+          id ${record.dig("app_id")}
+          message \{\"cf_app_id\":\"${record.dig("app_id")}\"\,\"msg\":\"${record.dig("log")}\"\,\"instance_id\"\:\"${record.dig("instance_id")}\"\}
+        </record>
+      </filter>
+
+      <filter kubernetes.** forwarded.**>
+        @type record_transformer
+        remove_keys app_id,instance_id,log,structured_data
+      </filter>
+
+      <match **>
+        @type influxdb
+        host 10.0.1.115
+        port 8086
+        user admin
+        password PaaS-TA2022
+        dbname logging_db
+        measurement logging_measurements
+        time_precision s
+        tag_keys ["id"]
+        time_key time
+        flush_interval 60
+        use_ssl true
+        verify_ssl false
+        sequence_tag _seq
+      </match>

...

```

> $ kubectl get daemonset,pods -n cf-system
```
NAME                     DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
daemonset.apps/fluentd   5         5         5       5            5           <none>          47h

NAME                                                      READY   STATUS      RESTARTS   AGE
pod/ccdb-migrate-cf4pf                                    0/2     Completed   0          47h
pod/cf-api-clock-589587f877-mtvbb                         2/2     Running     0          47h
pod/cf-api-controllers-6f9b86449d-sws96                   3/3     Running     1          47h
pod/cf-api-deployment-updater-5948c9df7f-ntbxj            2/2     Running     1          47h
pod/cf-api-server-544485c45b-2xvqn                        6/6     Running     2          47h
pod/cf-api-worker-8c6cd69c7-npgp7                         3/3     Running     0          47h
pod/eirini-api-6f78d469cf-pf2xp                           2/2     Running     0          47h
pod/eirini-app-migration-fgxq6                            0/1     Completed   0          47h
pod/eirini-event-reporter-779958dc9d-5g4fv                2/2     Running     6          47h
pod/eirini-event-reporter-779958dc9d-jpl6v                2/2     Running     5          47h
pod/eirini-instance-index-env-injector-556b856f89-vnl9d   1/1     Running     0          47h
pod/eirini-task-reporter-5f7b5766cf-725hc                 2/2     Running     5          47h
pod/eirini-task-reporter-5f7b5766cf-x6fnq                 2/2     Running     2          47h
pod/fluentd-l87rf                                         2/2     Running     0          25h
pod/fluentd-mfkpx                                         2/2     Running     0          25h
pod/fluentd-nlf8l                                         2/2     Running     1          25h
pod/fluentd-rwqf6                                         2/2     Running     1          25h
pod/fluentd-vbmms                                         2/2     Running     0          25h
pod/log-cache-backend-5f795cbdcd-f2msz                    3/3     Running     0          47h
pod/log-cache-frontend-764dd79ccc-jxzv7                   3/3     Running     0          47h
pod/metric-proxy-b5c86f55f-rgff4                          2/2     Running     0          47h
pod/routecontroller-66458dfdbb-pztvw                      2/2     Running     6          47h
pod/uaa-6fc9cf8bcb-9mq7r                                  3/3     Running     0          41h
```

<br>

# <div id='3'> 3. Logging 서비스 관리
## <div id='3.1'> 3.1. 보안 그룹 적용
Service와의 통신을 위하여 보안 그룹을 추가한다.

- rule.json을 편집한다.
```
$ cd $HOME/service-deployment/logging-service
$ vi rule.json

## Logging의 log-api IP를 destination에 설정
[
  {
    "destination": "<log-api_IP>",
    "protocol": "all"
  }
]
```

- 보안 그룹을 생성한다.
```
$ cf create-security-group logging rule.json
Creating security group logging as admin...

OK
```

- Logging 서비스를 사용할 수 있도록 생성한 보안 그룹을 적용한다.
```
$ cf bind-staging-security-group logging
Binding security group logging to staging as admin...
OK

TIP: If Dynamic ASG's are enabled, changes will automatically apply for running and staging applications. Othetart (for running) or restage (for staging) to apply to existing applications.


$ cf bind-running-security-group logging
Binding security group logging to running as admin...
OK

TIP: If Dynamic ASG's are enabled, changes will automatically apply for running and staging applications. Othetart (for running) or restage (for staging) to apply to existing applications.
```

<br>

## <div id='3.2'> 3.2. Logging 서비스 활성화
PaaS-TA 포탈에서 서비스를 사용하기 위해 Logging 서비스 활성화 코드 등록을 해 주어야 한다.

-	PaaS-TA 운영자 포탈에 접속한다.
![001]

-	운영관리의 코드관리 메뉴로 이동하여 다음과 같이 코드를 등록한다.

> ※ Group Table  
> 코드 ID  : LOGGING  
> 코드 이름 : Logging Service  
> ![002]
>
> ※ Detail Table  
> Key : enable_logging_service  
> Value : true  
> 요약 : Logging Service Enable Code  
> 사용 : Y  
> ![003]

![004]

[001]:./images/service/logging-service/image001.png
[002]:./images/service/logging-service/image002.png
[003]:./images/service/logging-service/image003.png
[004]:./images/service/logging-service/image004.png

<br>


### [Index](https://github.com/PaaS-TA/Guide/blob/master/README.md) > [PaaS-TA Sidecar Install](./README.md) > Logging Service
