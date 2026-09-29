# Cloudera Manager Configuration Backup & Restore

## 1. 개요

Cloudera Manager는 REST API를 이용하여 현재 Cloudera Manager가 관리하고 있는 Deployment Configuration을 JSON 형식으로 Export하고, 필요할 경우 해당 JSON을 다시 Import하여 구성을 복원할 수 있다.

주요 API는 다음과 같다.

```text
GET /api/vXX/cm/deployment
PUT /api/vXX/cm/deployment
```

Backup은 `GET /cm/deployment`, Restore는 `PUT /cm/deployment` API를 사용한다.

이 방식으로 Export되는 Deployment 정보에는 일반적으로 다음과 같은 Cloudera Manager 관리 정보가 포함된다.

- Cloudera Manager 설정
- Cluster 정의
- Host 정보
- Service 구성
- Role 구성
- Role Configuration Group
- 사용자 및 일부 보안 관련 설정
- 서비스 Configuration
- Cloudera Manager 설정
- Cloudera Manager가 관리하는 Deployment 구조

따라서 단순한 특정 서비스 설정 백업이라기보다 **Cloudera Manager Deployment Configuration 전체의 논리적인 백업**으로 이해하는 것이 적절하다.

> 주의  
> 이 기능은 HDFS DataNode 데이터, Hive 데이터, Kudu 데이터, HBase 데이터 등의 실제 사용자 데이터를 백업하는 기능이 아니다.  
> 또한 Cloudera Manager의 PostgreSQL/MySQL/Oracle Database 자체를 물리적으로 백업하는 기능도 아니다.

---

# 2. 요구 권한

Configuration Export / Import를 수행하려면 기본적으로 다음 권한이 필요하다.

```text
Cluster Administrator
```

또는:

```text
Full Administrator
```

Cloudera Manager가 Data Hub Cluster를 관리하는 환경에서는 해당 기능을 사용할 수 없다.

---

# 3. Backup / Restore 전체 흐름

전체 절차는 다음과 같다.

```text
                  Cloudera Manager
                         │
                         │ GET
                         │ /api/vXX/cm/deployment
                         ▼
              ┌─────────────────────┐
              │ cm-deployment.json  │
              │                     │
              │ Cluster             │
              │ Service             │
              │ Role                │
              │ Host                │
              │ Configuration       │
              │ User / Settings     │
              └──────────┬──────────┘
                         │
                         │ Backup
                         │
                         ▼
                 Backup Repository

                         │
                         │ Restore
                         ▼

              ┌─────────────────────┐
              │ cm-deployment.json  │
              └──────────┬──────────┘
                         │
                         │ PUT
                         │ /api/vXX/cm/deployment
                         │
                         ▼
                  Cloudera Manager
                         │
                         ▼
               Deployment Restore
```

---

# 4. 중요한 사항: Sensitive Configuration Redaction

Cloudera Manager는 기본적으로 Export되는 Configuration의 민감 정보(Sensitive Information)를 보호하기 위해 값을 **redaction**할 수 있다.

예를 들어 Password 또는 Credential 관련 설정이 다음과 같이 저장될 수 있다.

```json
{
  "name": "database_password",
  "value": "REDACTED"
}
```

이 상태의 JSON을 그대로 Restore하면 실제 Password가 없기 때문에 정상적인 복구가 되지 않을 수 있다.

따라서 Restore 가능한 백업을 만들려면 다음 두 방법 중 하나를 사용해야 한다.

### 방법 1. Export 시 Redaction 비활성화

Cloudera Manager API의 Redaction 기능을 Disable하여 실제 값을 포함한 JSON을 생성한다.

### 방법 2. Restore 전에 REDACTED 값 수정

기본 설정으로 Export하고 다음 값을:

```text
REDACTED
```

Restore 전에 실제 Password / Credential 값으로 변경한다.

운영 환경에서는 JSON 파일 안에 Password가 평문 형태로 포함될 가능성이 있으므로 보관 위치 및 접근 권한에 특히 주의해야 한다.

---

# 5. Redaction 비활성화

Restore가 가능한 완전한 Configuration Backup을 생성하려면 Cloudera Manager Server JVM Option에 다음 값을 설정할 수 있다.

Cloudera Manager Server에서:

```bash
vi /etc/default/cloudera-scm-server
```

기존:

```bash
export CMF_JAVA_OPTS="-Xmx4G"
```

다음 옵션을 추가한다.

```bash
-Dcom.cloudera.api.redaction=false
```

예:

```bash
export CMF_JAVA_OPTS="-Xmx4G -Dcom.cloudera.api.redaction=false"
```

설정 변경 후 Cloudera Manager Server를 재시작한다.

```bash
sudo systemctl restart cloudera-scm-server
```

상태 확인:

```bash
systemctl status cloudera-scm-server
```

> 보안 주의  
> Redaction을 비활성화하면 Export JSON에 Password, Credential 등의 민감한 Configuration이 포함될 수 있다.  
> 따라서 생성된 JSON 파일은 일반 Configuration 파일과 동일하게 취급해서는 안 된다.

권장 권한:

```bash
chmod 600 cm-deployment.json
```

가능하면 다음과 같은 별도의 보안 저장소에 보관하는 것이 좋다.

```text
Encrypted Backup Storage
Vault
Restricted NFS
Encrypted Object Storage
```

---

# 6. Cloudera Manager Configuration Backup

## 6.1 API 확인

Backup에 사용되는 API는 다음과 같다.

```text
GET /api/vXX/cm/deployment
```

이 API는 Cloudera Manager Deployment 전체 정보를 반환한다.

Cloudera API 문서에서는 Deployment 정보에 다음과 같은 내용이 포함된다고 설명한다.

```text
hosts
clusters
services
roles
users
settings
...
```

즉 이 JSON을 이용하여 Deployment를 재구성할 수 있다.

---

# 7. API Version 확인

Cloudera Manager 버전에 따라 API Version이 달라질 수 있으므로 운영 환경에서는 먼저 API Version을 확인하는 것이 좋다.

예:

```bash
curl -u admin:password \
http://cm.example.com:7180/api/version
```

예상 결과:

```text
v49
```

이 경우 이후 API는 다음과 같이 사용한다.

```text
/api/v49/cm/deployment
```

Cloudera 문서의 예제에 표시되는 API Version은 제품 릴리스에 따라 달라질 수 있기 때문에, **문서에 있는 숫자를 무조건 고정해서 사용하기보다는 `/api/version`으로 해당 Cloudera Manager가 지원하는 API Version을 확인하는 방법을 권장한다.**

---

# 8. HTTP 환경 Backup

Cloudera Manager가 일반 HTTP를 사용하는 경우 기본적으로 다음과 같이 수행한다.

```bash
curl -u <ADMIN_USER>:<ADMIN_PASSWORD> \
"http://<CM_HOST>:7180/api/v49/cm/deployment" \
-o cm-deployment.json
```

또는 shell redirection을 사용할 수도 있다.

```bash
curl -u <ADMIN_USER>:<ADMIN_PASSWORD> \
"http://<CM_HOST>:7180/api/v49/cm/deployment" \
> cm-deployment.json
```

예:

```bash
curl -u admin:password \
"http://cm01.example.com:7180/api/v49/cm/deployment" \
-o /backup/cloudera/cm-deployment.json
```

---

# 9. HTTPS 환경 Backup

Cloudera Manager TLS가 설정되어 있는 환경에서는 HTTPS URL을 사용한다.

예:

```bash
curl -u <ADMIN_USER>:<ADMIN_PASSWORD> \
"https://<CM_HOST>:7183/api/v49/cm/deployment" \
-o cm-deployment.json
```

예:

```bash
curl -u admin:password \
"https://cm01.example.com:7183/api/v49/cm/deployment" \
-o /backup/cloudera/cm-deployment.json
```

내부 CA 또는 자체 서명 인증서를 사용하는 환경에서 curl이 인증서를 신뢰하지 않는 경우 테스트 목적으로 다음과 같이 사용할 수 있다.

```bash
curl -k -u admin:password \
"https://cm01.example.com:7183/api/v49/cm/deployment" \
-o cm-deployment.json
```

하지만 운영 환경에서는 `-k`를 상시 사용하는 것보다 CA Certificate를 curl이 신뢰하도록 구성하는 것이 바람직하다.

예:

```bash
curl --cacert /etc/pki/ca-trust/source/anchors/company-ca.pem \
-u admin:password \
"https://cm01.example.com:7183/api/v49/cm/deployment" \
-o cm-deployment.json
```

---

# 10. Backup 파일 확인

Export 후 파일 크기를 확인한다.

```bash
ls -lh cm-deployment.json
```

JSON 형식 검증:

```bash
jq empty cm-deployment.json
```

오류가 없다면 정상 JSON이다.

내용을 간단히 확인하려면:

```bash
jq '.' cm-deployment.json | less
```

---

# 11. REDACTED 항목 확인

Restore 목적으로 사용할 경우 반드시 REDACTED 항목이 존재하는지 확인한다.

```bash
grep -n '"REDACTED"' cm-deployment.json
```

또는:

```bash
grep -n 'REDACTED' cm-deployment.json
```

개수 확인:

```bash
grep -o 'REDACTED' cm-deployment.json | wc -l
```

결과가:

```text
0
```

이면 Redaction이 없는 Backup일 가능성이 높다.

결과가 예를 들어:

```text
12
```

이면 Restore 전에 해당 값에 대한 확인이 필요하다.

---

# 12. Backup 파일 보안

Redaction을 Disable한 Deployment JSON은 사실상 Credential Backup의 성격도 갖기 때문에 파일 권한을 제한한다.

```bash
chown root:root cm-deployment.json
chmod 600 cm-deployment.json
```

가능하다면 암호화하여 저장한다.

예:

```bash
gpg -c cm-deployment.json
```

생성:

```text
cm-deployment.json.gpg
```

원본 삭제:

```bash
shred -u cm-deployment.json
```

---

# 13. Backup 파일 Version 관리

Configuration 변경 전마다 다음과 같이 백업하는 방식이 유용하다.

```bash
cm-deployment-YYYYMMDD-HHMMSS.json
```

예:

```bash
BACKUP_DIR=/backup/cloudera
DATE=$(date '+%Y%m%d-%H%M%S')

curl -u admin:password \
"https://cm01.example.com:7183/api/v49/cm/deployment" \
-o "${BACKUP_DIR}/cm-deployment-${DATE}.json"
```

예:

```text
cm-deployment-20260930-090000.json
cm-deployment-20261001-090000.json
cm-deployment-20261002-090000.json
```

---

# 14. Backup Script 예제

운영에서 사용할 수 있는 간단한 예제이다.

```bash
#!/bin/bash

CM_HOST="cm01.example.com"
CM_PORT="7183"
CM_USER="admin"
CM_PASSWORD="password"

API_VERSION="v49"

BACKUP_DIR="/backup/cloudera-manager"
DATE=$(date '+%Y%m%d-%H%M%S')

BACKUP_FILE="${BACKUP_DIR}/cm-deployment-${DATE}.json"

mkdir -p "${BACKUP_DIR}"

curl --fail --silent --show-error \
    -u "${CM_USER}:${CM_PASSWORD}" \
    "https://${CM_HOST}:${CM_PORT}/api/${API_VERSION}/cm/deployment" \
    -o "${BACKUP_FILE}"

if [ $? -ne 0 ]; then
    echo "ERROR: Cloudera Manager configuration backup failed."
    exit 1
fi

if ! jq empty "${BACKUP_FILE}" >/dev/null 2>&1; then
    echo "ERROR: Invalid JSON file."
    exit 1
fi

chmod 600 "${BACKUP_FILE}"

echo "Backup completed:"
echo "${BACKUP_FILE}"
```

실제 운영에서는 Script 내부에 Password를 직접 저장하는 방법은 피하는 것이 좋다.

예를 들어:

```text
Vault
Environment variable
Protected credential file
Secret management system
```

등을 사용할 수 있다.

---

# 15. Cloudera Manager Configuration Restore

Restore는 Backup JSON을 Cloudera Manager Deployment API로 PUT하여 수행한다.

사용 API:

```text
PUT /api/vXX/cm/deployment
```

Restore 시 중요한 Query Parameter가 있다.

```text
deleteCurrentDeployment=true
```

즉:

```text
PUT /api/v49/cm/deployment?deleteCurrentDeployment=true
```

의 형태로 수행한다.

---

# 16. Restore 전 반드시 확인할 사항

Restore 전에 최소 다음 사항을 확인하는 것을 권장한다.

```text
1. Backup JSON의 API/CM Version
2. REDACTED 값 존재 여부
3. Hostname 변경 여부
4. Cluster 이름
5. 서비스 및 Role 배치
6. Repository 설정
7. TLS / AutoTLS 관련 설정
8. Kerberos 관련 Principal / Keytab
9. Database 접속정보
10. External Service URL
```

특히 Hostname이나 Infrastructure가 변경된 환경에 그대로 Deployment Backup을 Restore하는 것은 위험할 수 있다.

---

# 17. REDACTED 값 수정

Backup 파일에 다음과 같은 값이 있다면:

```json
{
    "name": "database_password",
    "value": "REDACTED"
}
```

Restore 전에 실제 값으로 수정해야 한다.

예:

```json
{
    "name": "database_password",
    "value": "ActualPassword"
}
```

REDACTED 값 찾기:

```bash
grep -n REDACTED cm-deployment.json
```

---

# 18. Restore 전 Cluster Service 중지

Cloudera 공식 절차에서는 Restore 전에 Cluster에서 실행 중인 Service를 중지하도록 안내한다.

Cloudera Manager UI에서:

```text
Home
 → Cluster
 → Actions
 → Stop
```

모든 서비스가 중지될 때까지 기다린다.

상태:

```text
All services successfully stopped
```

를 확인한다.

이 과정은 매우 중요하다.

Cluster를 미리 중지하지 않은 상태에서 Deployment Restore API를 호출하면 Cloudera Manager가 서비스 중지를 수행할 수 있으며, 이 과정에서 실행 중인 Job이나 작업이 종료될 수 있다.

따라서 Production에서는 반드시 Maintenance Window를 확보하고 Restore를 수행하는 것이 좋다.

---

# 19. HTTP 환경 Restore

Backup 파일:

```text
/backup/cloudera/cm-deployment.json
```

이 있는 경우 다음과 같이 Restore한다.

```bash
curl \
-H "Content-Type: application/json" \
--upload-file /backup/cloudera/cm-deployment.json \
-u admin:password \
"http://cm01.example.com:7180/api/v49/cm/deployment?deleteCurrentDeployment=true"
```

동일하게 `-X PUT`을 명시해서 사용할 수도 있다.

```bash
curl -X PUT \
-H "Content-Type: application/json" \
-u admin:password \
--data-binary @/backup/cloudera/cm-deployment.json \
"http://cm01.example.com:7180/api/v49/cm/deployment?deleteCurrentDeployment=true"
```

`--upload-file`은 curl에서 PUT 방식으로 파일을 전송하므로 Cloudera 문서의 예제에서 사용된다.

---

# 20. HTTPS 환경 Restore

AutoTLS 또는 HTTPS 환경에서는 다음과 같이 수행한다.

```bash
curl \
-H "Content-Type: application/json" \
--upload-file /backup/cloudera/cm-deployment.json \
-u admin:password \
"https://cm01.example.com:7183/api/v49/cm/deployment?deleteCurrentDeployment=true"
```

CA 검증이 필요한 환경:

```bash
curl \
--cacert /etc/pki/ca-trust/source/anchors/company-ca.pem \
-H "Content-Type: application/json" \
--upload-file /backup/cloudera/cm-deployment.json \
-u admin:password \
"https://cm01.example.com:7183/api/v49/cm/deployment?deleteCurrentDeployment=true"
```

---

# 21. `deleteCurrentDeployment=true` 의미

Restore에서 가장 중요한 Option 중 하나이다.

```text
deleteCurrentDeployment=true
```

이는 현재 Cloudera Manager Deployment Configuration을 제거한 후 전달한 Deployment Description을 적용하도록 요청하는 것이다.

개념적으로:

```text
Existing Deployment

Cluster
 ├─ HDFS
 ├─ YARN
 ├─ Hive
 ├─ Impala
 └─ Kudu

          ↓

deleteCurrentDeployment=true

          ↓

Existing Deployment Definition 제거

          ↓

cm-deployment.json 적용

          ↓

Restored Deployment
```

따라서 Production에서 매우 신중하게 사용해야 한다.

단순한 Configuration 일부 수정 용도로 사용하는 API가 아니다.

---

# 22. Restore 이후 확인

Restore API 수행 후 바로 Cluster를 시작하기보다는 Configuration과 상태를 먼저 확인하는 것을 권장한다.

## 22.1 Cloudera Manager UI

확인:

```text
Hosts
Clusters
Services
Role Instances
Configuration
Parcels
Security
```

---

## 22.2 Host 확인

```bash
curl -u admin:password \
"https://cm01.example.com:7183/api/v49/hosts"
```

---

## 22.3 Cluster 확인

```bash
curl -u admin:password \
"https://cm01.example.com:7183/api/v49/clusters"
```

---

## 22.4 Service 확인

예:

```bash
curl -u admin:password \
"https://cm01.example.com:7183/api/v49/clusters/Cluster1/services"
```

---

# 23. Restore 후 서비스 기동

Configuration 및 Host/Role 상태에 문제가 없는 것을 확인한 후 서비스를 시작한다.

Cloudera Manager:

```text
Cluster
 → Actions
 → Start
```

이후 다음 서비스들의 Health 상태를 확인한다.

예:

```text
ZooKeeper
HDFS
YARN
Hive
Impala
Kudu
HBase
Kafka
Ranger
Atlas
Solr
Knox
```

환경에 존재하는 서비스만 확인한다.

---

# 24. Restore 검증

복원 후 다음 항목을 확인하는 것을 권장한다.

## Cloudera Manager

```text
Cluster Health
Host Health
Service Health
Role Health
Configuration Staleness
Restart Required
```

## HDFS

```bash
hdfs dfsadmin -report
```

```bash
hdfs haadmin -getAllServiceState
```

## YARN

```bash
yarn node -list
```

## Hive

```bash
beeline
```

접속 및 Query 확인.

## Impala

```bash
impala-shell
```

Query 수행 확인.

## Kudu

```bash
kudu cluster ksck <master-list>
```

## Kafka

Topic 조회 또는 Producer/Consumer Test를 수행한다.

---

# 25. Backup/Restore 권장 운영 절차

Production에서는 다음 절차를 권장한다.

## Backup

```text
1. Cloudera Manager API Version 확인
        │
2. Configuration 변경 여부 확인
        │
3. Deployment Export
        │
4. JSON 유효성 검사
        │
5. REDACTED 확인
        │
6. Backup 파일 암호화
        │
7. Backup Repository 저장
```

## Restore

```text
1. Maintenance Window 확보
        │
2. 현재 Configuration 별도 Backup
        │
3. Restore JSON 검증
        │
4. REDACTED 확인 및 수정
        │
5. Cluster Service Stop
        │
6. PUT /cm/deployment
        │
7. Deployment Configuration 확인
        │
8. Cluster Service Start
        │
9. Service Health 확인
        │
10. Application Test
```

---

# 26. Restore 전에 현재 상태 추가 Backup

기존 환경에 Restore하기 직전에 반드시 현재 Deployment를 한 번 더 Export하는 것을 권장한다.

예:

```bash
curl -u admin:password \
"https://cm01.example.com:7183/api/v49/cm/deployment" \
-o before-restore-$(date '+%Y%m%d-%H%M%S').json
```

즉:

```text
Restore Target
     │
     ├── Current Deployment Backup
     │
     ├── Restore Previous Deployment
     │
     └── Validation
```

구조로 운영하는 것이 안전하다.

---

# 27. Configuration Backup과 Database Backup의 차이

Cloudera Manager 운영에서 중요한 부분이다.

### Configuration Export

```text
GET /cm/deployment
```

의 목적은 Cloudera Manager가 관리하는 논리적인 Deployment Configuration을 보존하는 것이다.

### Cloudera Manager Database Backup

Cloudera Manager Database에는 Cloudera Manager의 내부 상태와 Metadata가 저장된다.

예를 들어 PostgreSQL을 사용한다면:

```bash
pg_dump scm > scm.sql
```

형태의 별도 Database Backup이 필요하다.

따라서 실질적인 Disaster Recovery 관점에서는 둘 다 보관하는 것을 권장한다.

```text
Cloudera Manager DR Backup
│
├── Cloudera Manager Deployment JSON
│     └── /api/vXX/cm/deployment
│
├── Cloudera Manager Database
│     └── scm database
│
├── Service Databases
│     ├── Hive Metastore
│     ├── Ranger
│     ├── Hue
│     ├── Oozie
│     └── 기타 DB
│
└── Security Assets
      ├── TLS certificates
      ├── CA
      ├── Keytab
      ├── krb5.conf
      └── External credentials
```

즉 `/cm/deployment` Export만 가지고 Cloudera Platform 전체 DR Backup이 완성되는 것은 아니다.

---

# 28. 권장 Backup 구성

운영 환경에서는 다음과 같은 구조를 권장한다.

```text
/backup/cloudera/
│
├── configuration/
│   ├── 20260928/
│   │   └── cm-deployment.json
│   │
│   ├── 20260929/
│   │   └── cm-deployment.json
│   │
│   └── 20260930/
│       └── cm-deployment.json
│
├── database/
│   ├── scm.sql
│   ├── hive.sql
│   ├── ranger.sql
│   └── hue.sql
│
├── kerberos/
│   ├── krb5.conf
│   └── keytabs/
│
└── tls/
    ├── certificates/
    └── ca/
```

---

# 29. 자동 Backup Cron 예제

예를 들어 매일 새벽 02:00에 Backup한다면:

```bash
crontab -e
```

```cron
0 2 * * * /opt/scripts/cm-config-backup.sh >> /var/log/cm-config-backup.log 2>&1
```

Script:

```bash
#!/bin/bash

CM_HOST="cm01.example.com"
CM_PORT="7183"
CM_USER="admin"

API_VERSION=$(curl -ks \
    -u "${CM_USER}:${CM_PASSWORD}" \
    "https://${CM_HOST}:${CM_PORT}/api/version")

BACKUP_ROOT="/backup/cloudera/configuration"
DATE=$(date '+%Y%m%d')
TIME=$(date '+%H%M%S')

BACKUP_DIR="${BACKUP_ROOT}/${DATE}"
BACKUP_FILE="${BACKUP_DIR}/cm-deployment-${TIME}.json"

mkdir -p "${BACKUP_DIR}"

curl --fail --silent --show-error \
    -k \
    -u "${CM_USER}:${CM_PASSWORD}" \
    "https://${CM_HOST}:${CM_PORT}/api/${API_VERSION}/cm/deployment" \
    -o "${BACKUP_FILE}"

if ! jq empty "${BACKUP_FILE}" >/dev/null 2>&1; then

    echo "[ERROR] Invalid JSON"
    rm -f "${BACKUP_FILE}"

    exit 1
fi

chmod 600 "${BACKUP_FILE}"

REDACTED_COUNT=$(grep -o 'REDACTED' "${BACKUP_FILE}" | wc -l)

echo "Backup: ${BACKUP_FILE}"
echo "REDACTED count: ${REDACTED_COUNT}"
```

---

# 30. 실운영 권장 개선

Script에 Password를 직접 기록하지 않고 다음과 같이 별도로 관리하는 것이 좋다.

예를 들어:

```bash
/root/.cm-credentials
```

```text
CM_USER=backup_admin
CM_PASSWORD=<password>
```

권한:

```bash
chmod 600 /root/.cm-credentials
```

Script에서는:

```bash
source /root/.cm-credentials
```

다만 더 높은 보안 수준이 필요한 환경에서는 Vault 등의 Secret Manager 사용이 적합하다.

---

# 31. 백업 주기 권장

Configuration 변경 빈도에 따라 다음 정도가 현실적이다.

| 종류 | 권장 주기 |
|---|---|
| 정기 Configuration Backup | Daily |
| CM 설정 변경 전 | 반드시 |
| Parcel Upgrade 전 | 반드시 |
| Runtime Upgrade 전 | 반드시 |
| Host 추가/삭제 전후 | 권장 |
| Service 추가/삭제 전 | 반드시 |
| TLS 변경 전 | 반드시 |
| Kerberos 변경 전 | 반드시 |
| Major Configuration 변경 전 | 반드시 |

가장 안전한 운영 방법은:

```text
Regular Daily Backup
+
Change-triggered Backup
```

두 방식을 병행하는 것이다.

---

# 32. 특히 주의할 사항

## 32.1 API Version

Cloudera 문서의 예제에서는 제품 버전에 대응되는 특정 API Version을 사용하지만 실제 환경에서는:

```bash
/api/version
```

으로 확인한다.

```bash
curl -u admin:password \
https://cm.example.com:7183/api/version
```

---

## 32.2 Redaction

다음 값이 있으면:

```text
REDACTED
```

Restore 전에 반드시 확인해야 한다.

```bash
grep -R REDACTED cm-deployment.json
```

---

## 32.3 Password 포함 Backup

다음을 설정한 경우:

```text
-Dcom.cloudera.api.redaction=false
```

민감 정보가 JSON에 포함될 수 있으므로 Backup 파일 자체를 Secret으로 취급한다.

---

## 32.4 Cluster Stop

Restore 전 Cluster Service를 명시적으로 Stop하는 것을 권장한다.

그렇지 않은 경우 Restore API 호출 과정에서 Cloudera Manager가 서비스를 중지할 수 있으며 실행 중인 Job에 영향을 줄 수 있다.

---

## 32.5 `deleteCurrentDeployment=true`

매우 주의해서 사용해야 한다.

```text
deleteCurrentDeployment=true
```

는 기존 Deployment를 유지하면서 일부 Configuration을 Merge하는 개념이 아니다.

기존 Deployment 정의를 제거하고 Backup Deployment를 적용하는 Restore 성격의 작업이다.

---

# 33. 최종 명령어 요약

## API Version 확인

```bash
curl -u admin:password \
https://cm01.example.com:7183/api/version
```

---

## Backup

```bash
curl -u admin:password \
"https://cm01.example.com:7183/api/v49/cm/deployment" \
-o cm-deployment.json
```

---

## JSON 검증

```bash
jq empty cm-deployment.json
```

---

## Redaction 확인

```bash
grep -n REDACTED cm-deployment.json
```

---

## Restore 전 Cluster Stop

```text
Cloudera Manager
 → Cluster
 → Actions
 → Stop
```

---

## Restore

```bash
curl \
-H "Content-Type: application/json" \
--upload-file cm-deployment.json \
-u admin:password \
"https://cm01.example.com:7183/api/v49/cm/deployment?deleteCurrentDeployment=true"
```

---

# 34. 핵심 정리

Cloudera Manager Configuration Backup / Restore의 핵심은 다음 두 API이다.

```text
Backup
GET /api/vXX/cm/deployment

Restore
PUT /api/vXX/cm/deployment?deleteCurrentDeployment=true
```

전체 절차는 다음과 같이 정리할 수 있다.

```text
                BACKUP

 Cloudera Manager
        │
        │ GET /cm/deployment
        ▼
 cm-deployment.json
        │
        ├── JSON Validate
        ├── REDACTED Check
        ├── Encrypt
        └── Backup Storage


                RESTORE

 cm-deployment.json
        │
        ├── JSON Validate
        ├── REDACTED Replace
        │
        ▼
 Cluster Service STOP
        │
        ▼
 PUT /cm/deployment
 deleteCurrentDeployment=true
        │
        ▼
 Configuration Validation
        │
        ▼
 Cluster START
        │
        ▼
 Service / Role / Data Validation
```

운영 관점에서는 `/cm/deployment` Backup만으로 전체 Cloudera DR을 구성하지 말고 다음을 함께 백업하는 것이 중요하다.

```text
1. Cloudera Manager Deployment JSON
2. Cloudera Manager Database
3. Hive/Ranger/Hue 등 Service Database
4. Kerberos Configuration / Keytab
5. TLS Certificate / CA
6. 외부 DB 및 External Service Configuration
```

이렇게 구성해야 실제 장애 또는 Cloudera Manager 재구축 상황에서 복구 가능성이 높아진다.
