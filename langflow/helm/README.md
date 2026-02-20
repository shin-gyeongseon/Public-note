# Common Labels 적용 방식 (README)

## 결론

`values.yaml`의 `commonLabels`에 공통 라벨을 선언하고, `_helpers.tpl`에서 이를 템플릿 함수로 정의한 뒤, **각 리소스 템플릿에서 `include`로 주입**해서 모든 리소스에 동일 라벨을 일관 적용합니다.

---

## 1) values.yaml: 공통 라벨 선언

`values.yaml`에 공통으로 붙일 라벨들을 `commonLabels`로 정의합니다.

```yaml
commonLabels:
  env: "production"
  billing-id: "12345"
  project: "langflow"
```

* 여기 선언된 값은 **모든 리소스의 `metadata.labels`에 추가**되는 것을 목표로 합니다.
* (중요) `selectorLabels`는 차트에서 의도적으로 무시하도록 주석 처리되어 있습니다. 보통 selector는 불변(immutable) 성격이 강해서, 사용자 입력으로 바꾸게 하면 배포/업그레이드 시 리소스 교체/실패 리스크가 커집니다.

---

## 2) _helpers.tpl: 공통 라벨 렌더링 함수 정의

`templates/_helpers.tpl`에서 `commonLabels`를 YAML로 렌더링하는 helper를 정의합니다.

```tpl
{{- define "langflow.labels.common" -}}
{{- with .Values.commonLabels }}
{{ toYaml . }}
{{- end }}
{{- end }}
```

* `with .Values.commonLabels` : 값이 있을 때만 출력
* `toYaml` : key/value를 YAML 블록으로 변환
* 결과적으로 `commonLabels`가 있으면 라벨 블록이 그대로 삽입됩니다.

---

## 3) 각 템플릿: metadata.labels에 include로 주입

리소스 템플릿(예: `backend-statefulset.yaml`)에서 `metadata.labels` 하위에 helper를 포함합니다.

```yaml
metadata:
  labels:
    release: {{ .Release.Name }}
    langflow-scope: "backend"
    {{- include "langflow.labels.common" . | nindent 4 }}
```

* `include "langflow.labels.common" .` : helper 실행 결과를 삽입
* `nindent` : YAML 들여쓰기 정렬(템플릿 위치에 맞춰 조정)

이 방식으로 **템플릿마다 공통 라벨을 “필요한 곳에만” 명시적으로 주입**할 수 있어, 라벨 정책을 중앙(`values.yaml`)에서 관리하면서도 렌더링 위치는 템플릿이 통제합니다.

---

## 운영 팁

* 공통 라벨은 보통 `env`, `project`, `owner/team`, `cost-center(billing)` 같이 **필터링/과금/관측**에 필요한 것 위주로 고정합니다.
* selector 성격(`app.kubernetes.io/name`, `instance` 등)은 helper로 강제 주입하기보다, 차트가 관리하는 selector 로직을 따르는 편이 안전합니다.

---

### 다음 단계 질문(추천)

지금은 `backend-statefulset.yaml`에만 적용되어 있는데, **공통 라벨을 어떤 리소스들(Deployment/Service/Ingress/Job 등)까지 “반드시” 강제 적용**할지 범위를 정해드릴까요?
