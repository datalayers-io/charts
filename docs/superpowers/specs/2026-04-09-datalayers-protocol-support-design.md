# Design: PostgreSQL / Redis / Prometheus Protocol Support for datalayers-single

**Date:** 2026-04-09
**Chart:** datalayers-single (v0.0.1-alpha.9, appVersion v2.4.0)
**Reference:** https://docs.datalayers.cn/datalayers/latest/development-guide/connection.html

## Summary

Add optional PostgreSQL, Redis, and Prometheus protocol support to the datalayers-single Helm chart. Each protocol is toggled by an `enabled` flag under `service.*` in values.yaml. When enabled, the chart injects the appropriate Datalayers environment variable, exposes the container port, and adds a Service port.

## Background

Datalayers supports six protocols. Two are always on (gRPC/FlightSQL on 8360, HTTP/REST on 8361). Three are disabled by default and activated by setting their `addr` configuration:

| Protocol   | Default Port | Env Var to Enable                        |
|------------|-------------|------------------------------------------|
| PostgreSQL | 5432        | `DATALAYERS_SERVER__POSTGRES__ADDR`      |
| Redis      | 6379        | `DATALAYERS_SERVER__REDIS__ADDR`         |
| Prometheus | 9090        | `DATALAYERS_SERVER__PROMETHEUS__ADDR`    |

Redis also requires `USERNAME` and `PASSWORD` env vars when enabled.
Prometheus has optional `memtable_size` and `ttl` params (kept at Datalayers defaults; configurable via `datalayersConfig` TOML if needed).

## Design

### values.yaml

Add three blocks under `service`, all disabled by default:

```yaml
service:
  type: ClusterIP
  grpc:
    port: 8360
  http:
    port: 8361
  postgres:
    enabled: false
    port: 5432
  redis:
    enabled: false
    port: 6379
  prometheus:
    enabled: false
    port: 9090
```

### templates/statefulset.yaml

#### Container ports

Append conditional ports after the existing `grpc` and `http` ports:

```yaml
{{- if .Values.service.postgres.enabled }}
- name: postgres
  containerPort: 5432
  protocol: TCP
{{- end }}
{{- if .Values.service.redis.enabled }}
- name: redis
  containerPort: 6379
  protocol: TCP
{{- end }}
{{- if .Values.service.prometheus.enabled }}
- name: prometheus
  containerPort: 9090
  protocol: TCP
{{- end }}
```

#### Environment variables

Inject `ADDR` env vars when each protocol is enabled. Place these after the existing auth env vars block:

```yaml
{{- if $.Values.service.postgres.enabled }}
- name: DATALAYERS_SERVER__POSTGRES__ADDR
  value: "0.0.0.0:5432"
{{- end }}
{{- if $.Values.service.redis.enabled }}
- name: DATALAYERS_SERVER__REDIS__ADDR
  value: "0.0.0.0:6379"
- name: DATALAYERS_SERVER__REDIS__USERNAME
  valueFrom:
    secretKeyRef:
      name: {{ include "datalayers-single.authSecretName" $ }}
      key: {{ $.Values.auth.static.secretKeys.username }}
- name: DATALAYERS_SERVER__REDIS__PASSWORD
  valueFrom:
    secretKeyRef:
      name: {{ include "datalayers-single.authSecretName" $ }}
      key: {{ $.Values.auth.static.secretKeys.password }}
{{- end }}
{{- if $.Values.service.prometheus.enabled }}
- name: DATALAYERS_SERVER__PROMETHEUS__ADDR
  value: "0.0.0.0:9090"
{{- end }}
```

Redis credentials reuse the existing auth secret (`datalayers-single.authSecretName`).

### templates/service.yaml

Append conditional ports after existing `grpc` and `http` ports:

```yaml
{{- if .Values.service.postgres.enabled }}
- port: {{ .Values.service.postgres.port }}
  targetPort: postgres
  protocol: TCP
  name: postgres
{{- end }}
{{- if .Values.service.redis.enabled }}
- port: {{ .Values.service.redis.port }}
  targetPort: redis
  protocol: TCP
  name: redis
{{- end }}
{{- if .Values.service.prometheus.enabled }}
- port: {{ .Values.service.prometheus.port }}
  targetPort: prometheus
  protocol: TCP
  name: prometheus
{{- end }}
```

### templates/NOTES.txt

Add conditional lines showing enabled protocol ports and include them in the port-forward command:

```
{{- if .Values.service.postgres.enabled }}
  - PostgreSQL:     {{ .Values.service.postgres.port }}
{{- end }}
{{- if .Values.service.redis.enabled }}
  - Redis:          {{ .Values.service.redis.port }}
{{- end }}
{{- if .Values.service.prometheus.enabled }}
  - Prometheus:     {{ .Values.service.prometheus.port }}
{{- end }}
```

### Files changed

| File | Change |
|------|--------|
| `values.yaml` | Add `service.postgres/redis/prometheus` blocks |
| `templates/statefulset.yaml` | Conditional containerPorts + env vars |
| `templates/service.yaml` | Conditional Service ports |
| `templates/NOTES.txt` | Conditional port info + port-forward args |

### Files not changed

- `_helpers.tpl` — no new helpers needed
- `secret.yaml` — Redis reuses existing auth secret
- `configmap.yaml` — no config changes
- `ingress.yaml` / `httproute.yaml` — TCP protocols, not HTTP routing

## Testing

- `helm template` with all protocols disabled (default) — output unchanged
- `helm template` with each protocol enabled individually — verify correct ports and env vars
- `helm template` with all three enabled — verify all ports and env vars present
- `helm lint` passes
