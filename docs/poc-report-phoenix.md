# Phoenix Helm PoC 정리 (외부 PostgreSQL 연동 중심)

이번 글은 **Arize Phoenix를 Helm으로 배포하면서 진행한 PoC 결과 정리**입니다.
특히 다음 내용을 중심으로 정리합니다.

* 외부 PostgreSQL 연동 방법
* 내부 DB 비활성화 설정
* Secret 수정 포인트
* 리소스(Resource) 설정 방법

최대한 실제 values.yaml 기준으로 설명합니다.

---

# 1. 전체 구조 요약

PoC에서는 다음과 같은 구조를 사용했습니다.

```
[Kubernetes]
   ├── Phoenix (Helm)
   ├── External PostgreSQL (별도 운영)
   └── Secret (DB password 등 관리)
```

Phoenix 내부에 Postgres를 생성하지 않고,
**이미 운영 중인 외부 PostgreSQL을 사용하도록 구성**했습니다.

---

# 2. 내부 데이터베이스 비활성화

Phoenix Helm chart에는 기본적으로 내부 Postgres를 포함할 수 있습니다.

PoC에서는 이를 사용하지 않았습니다.

### ✅ 내부 Postgres 비활성화

values.yaml에서 다음과 같이 설정합니다.

```yaml
postgresql:
  enabled: false
```

또는 chart 버전에 따라:

```yaml
database:
  postgres:
    enabled: false
```

핵심은:

> Phoenix chart가 내부 DB를 생성하지 않도록 설정하는 것

이 설정을 하지 않으면 내부 DB Pod가 생성됩니다.

---

# 3. 외부 PostgreSQL 연동 설정

이제 외부 DB 정보를 명시해야 합니다.

### ✅ externalDatabase 설정

```yaml
externalDatabase:
  enabled: true
  driver:
    value: postgresql
  host:
    value: your-postgres-host
  port:
    value: "5432"
  user:
    value: phoenix
  database:
    value: phoenix_db
```

### 각 필드 설명

| 필드       | 설명                         |
| -------- | -------------------------- |
| driver   | postgresql 사용              |
| host     | 외부 DB 주소 (svc, IP, FQDN 등) |
| port     | 보통 5432                    |
| user     | DB 계정                      |
| database | 이미 생성된 DB 이름               |

⚠️ 중요:

* Helm이 DB를 생성해주지 않습니다.
* DB는 반드시 **미리 생성**되어 있어야 합니다.
* 스키마도 필요 시 사전 준비해야 합니다.

---

# 4. PostgreSQL 비밀번호 설정 (Secret 필수 수정)

외부 DB를 사용할 경우, password 설정을 반드시 수정해야 합니다.

values 일부:

```yaml
# -- Environment variable name for the PostgreSQL password
- key: "PHOENIX_POSTGRES_PASSWORD"
  value: "langflow-pass" # Note: 수정함
```

이 부분이 실제 DB 비밀번호와 일치해야 합니다.

하지만 평문으로 두는 것은 권장되지 않습니다.

---

## ✅ 권장 방식: Kubernetes Secret 사용

### 1️⃣ Secret 생성

```bash
kubectl create secret generic phoenix-postgres-secret \
  --from-literal=PHOENIX_POSTGRES_PASSWORD=실제비밀번호
```

### 2️⃣ values.yaml 수정

```yaml
extraEnv:
  - name: PHOENIX_POSTGRES_PASSWORD
    valueFrom:
      secretKeyRef:
        name: phoenix-postgres-secret
        key: PHOENIX_POSTGRES_PASSWORD
```

이렇게 하면:

* 비밀번호를 values에 직접 노출하지 않음
* 보안 관리가 쉬움

---

# 5. 리소스(Resource) 설정

PoC에서 반드시 점검해야 하는 부분이 리소스입니다.

Phoenix는 trace를 저장하므로 메모리 사용량이 증가할 수 있습니다.

### ✅ 기본 리소스 설정 예시

```yaml
resources:
  requests:
    cpu: "500m"
    memory: "1Gi"
  limits:
    cpu: "1"
    memory: "2Gi"
```

### 설명

| 항목       | 의미          |
| -------- | ----------- |
| requests | 최소 보장 자원    |
| limits   | 최대 사용 가능 자원 |

PoC 기준으로는:

* 최소 1Gi 메모리 권장
* trace 많아지면 2Gi 이상 필요

운영 환경에서는 실제 트래픽 기준으로 조정해야 합니다.

---

# 6. 최종 구성 정리

PoC에서 우리가 한 설정은 다음과 같습니다.

1. 내부 Postgres 비활성화
2. externalDatabase 활성화
3. DB는 사전 생성
4. Secret로 비밀번호 관리
5. 리소스 명시적으로 설정

구성 흐름은 다음과 같습니다.

```
Helm values 수정
   ↓
Secret 생성
   ↓
helm upgrade --install 실행
   ↓
Phoenix Pod 기동
   ↓
외부 PostgreSQL 연결 확인
```

---

# 7. PoC 진행 시 체크리스트

* [ ] DB 접속 테스트 (psql로 직접 확인)
* [ ] Secret 정상 반영 여부 확인 (`kubectl describe pod`)
* [ ] Phoenix 로그에서 DB 연결 오류 확인
* [ ] resource 부족으로 OOM 발생하지 않는지 확인

---

# 8. 결론

이번 PoC에서 핵심은 다음입니다.

> Phoenix는 외부 DB 사용 시 Helm이 DB를 대신 생성하지 않는다.

따라서:

* DB 생성 책임은 운영자에게 있음
* Secret 관리가 필수
* 리소스 설정을 명시해야 안정적

이 구조를 기반으로 Managed Service 확장이 가능합니다.

---
