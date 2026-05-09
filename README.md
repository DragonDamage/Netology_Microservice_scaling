# Микросервисная инфраструктура: решение для масштабирования

## Общее описание

В качестве платформы для развёртывания, запуска и управления микросервисным приложением предлагается следующее решение:

| Компонент | Продукт | Назначение |
|-----------|---------|------------|
| Оркестратор | **Kubernetes** | Запуск и управление контейнерами |
| Service Mesh | **Istio** | Маршрутизация, обнаружение сервисов, безопасность трафика |
| Хранение секретов | **HashiCorp Vault** | Безопасное хранение паролей, ключей, токенов |
| CI/CD | **GitLab CI + ArgoCD** | Непрерывная интеграция и доставка (GitOps) |
| Мониторинг | **Prometheus + Grafana** | Сбор метрик, визуализация, автомасштабирование |

---

## Соответствие требованиям

### ✅ Поддержка контейнеров

- **Kubernetes** нативно работает с контейнерами (Docker, containerd, CRI-O).
- Приложения упаковываются в Docker-образы и описываются декларативными манифестами (Deployment, Pod, Service).

### ✅ Обнаружение сервисов и маршрутизация запросов

- **Kubernetes DNS** обеспечивает базовое обнаружение: `service-a.default.svc.cluster.local`.
- **Istio** добавляет продвинутую маршрутизацию:
  - Канареечные релизы (10% трафика на v2)
  - A/B-тестирование по заголовкам
  - Таймауты, ретраи, circuit breakers
  - Зеркалирование трафика (shadowing)

### ✅ Горизонтальное масштабирование

```bash
kubectl scale deployment/myapp --replicas=10
```

* Ручное масштабирование через kubectl.
* Автоматическое масштабирование через HPA (см. следующий пункт).

### ✅ Автоматическое масштабирование

Horizontal Pod Autoscaler (HPA) — встроенный механизм Kubernetes:

```yml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  minReplicas: 2
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```
* HPA отслеживает CPU, RAM, кастомные метрики (например, кол-во запросов в секунду).
* Автоматически добавляет или удаляет поды при изменении нагрузки.

Vertical Pod Autoscaler (VPA) — опционально, для автоматической настройки requests/limits.

### ✅ Разделение ресурсов (изоляция)
* Namespaces в Kubernetes:
  * `dev` — разработка
  * `staging` — тестирование
  * `prod` — продуктив
* Network Policies — запрещают/разрешают доступ между сервисами по умолчанию.
* Istio Authorization Policies — более тонкое управление на уровне service mesh.
 
### ✅ Конфигурация через переменные среды и безопасное хранение

| Способ | Инструмент | Для чего |
|-----------|---------|------------|
| Переменные окружения | ConfigMap | Нечувствительные настройки |
| Секреты | Kubernetes Secret | Базовое хранение (base64) |
| Безопасное хранение | HashiCorp Vault | Пароли, ключи, токены, сертификаты |

Пример ConfigMap:

```yml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_MODE: "production"
  LOG_LEVEL: "info"
```

Интеграция с Vault через CSI Driver:

```yml
kind: Pod
spec:
  containers:
    - name: myapp
      volumeMounts:
        - name: vault-secrets
          mountPath: /secrets
  volumes:
    - name: vault-secrets
      csi:
        driver: vault.csi.k8s.io
        volumeAttributes:
          secretProviderClass: vault-database
```
Vault не хранит секреты пода (в отличие от Kubernetes Secrets), поддерживает ротацию и аудит доступа.

## Архитектурная схема взаимодействия
```txt
Git (манифесты + код)
       │
       ▼
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│  GitLab CI  │────▶│ Container   │───▶│  Kubernetes │
│  (сборка)   │     │  Registry   │     │  (образы)   │
└─────────────┘     └─────────────┘     └──────┬──────┘
                                               │
                                               ▼
                                         ┌────────────┐
                                         │   ArgoCD   │
                                         │  (GitOps)  │
                                         └─────┬──────┘
                                               │
          ┌────────────────────────────────────┼──────────────────────────────────┐
          │                                    │                                  │
          ▼                                    ▼                                  ▼
   ┌────────────┐                      ┌────────────┐                      ┌────────────┐
   │  Istio     │◀─────трафик────────▶│   Pod A   │◀─────трафик─────────▶│   Pod B    │
   │ (sidecar)  │                      │ (myapp v1) │                      │ (myapp v2) │
   └────────────┘                      └────────────┘                      └────────────┘
          │                                                                      │
          │ mTLS                                                                 │
          └──────────────────────────────────────────────────────────────────────┘
                                              │
                                              ▼
                                     ┌─────────────┐
                                     │  Prometheus │
                                     │    HPA      │
                                     └─────────────┘
                                              │
                                              ▼
                                     ┌─────────────┐
                                     │   Grafana   │
                                     │ (дашборды)  │
                                     └─────────────┘
```

Описание:
* Сплошные стрелки — поток данных/образов.
* Пунктирные — управление и мониторинг.
* mTLS — взаимная аутентификация трафика между сервисами (обеспечивается Istio).

### Обоснование выбора

| Требование | Как обеспечивается | Почему именно этот инструмент |
|-----------|---------|------------|
| Поддержка контейнеров | Kubernetes | Индустриальный стандарт, поддерживается всеми облачными провайдерами |
| Обнаружение сервисов и маршрутизация | Istio | Даёт больше возможностей, чем стандартный K8s Ingress (canary, A/B, circuit breaking) |
| Горизонтальное масштабирование | Kubernetes + HPA | Встроенный механизм, работает с любыми метриками из Prometheus |
| Автоматическое масштабирование | HPA + Custom Metrics | Реагирует на реальную нагрузку, а не только на CPU/RAM |
| Разделение ресурсов | Namespaces + Network Policies | Чёткое разграничение сред и изоляция трафика по умолчанию |
| Переменные окружения | ConfigMap / Secret / Vault | Vault решает проблему безопасности секретов, встроенный в K8s механизм — для простых случаев |
| Безопасное хранение чувствительных данных | HashiCorp Vault | Шифрование, ротация, аудит, интеграция с K8s через CSI |

### Почему не альтернативы?

| Требование | Как обеспечивается |
|-----------|---------|
| Альтернатива | Минусы для данной задачи |
| Docker Swarm | Слабое автомасштабирование, плохая изоляция, устаревающая технология |
| Nomad | Нет встроенного service discovery — требуются дополнительные инструменты (Consul) |
| Plain NGINX + K8s | Нет canary-релизов, circuit breaking, автоматического обнаружения сервисов |
| Kubernetes Secrets без Vault | Нет шифрования, ротации, аудита — base64 только с шифрованием etcd |
| Traefik вместо Istio | Не даёт mTLS и тонкой политики безопасности между сервисами |

## Пример конфигурации (быстрый старт)
### 1. Deployment с ConfigMap и автоподключением к Vault
```yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: prod
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: app
          image: myregistry/myapp:latest
          env:
            - name: APP_MODE
              valueFrom:
                configMapKeyRef:
                  name: app-config
                  key: APP_MODE
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: vault-db-secret
                  key: password
          volumeMounts:
            - name: vault-secrets
              mountPath: /secrets
      volumes:
        - name: vault-secrets
          csi:
            driver: vault.csi.k8s.io
            volumeAttributes:
              secretProviderClass: vault-database
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_MODE: "production"
  LOG_LEVEL: "info"
```

### 2. HPA для автоматического масштабирования
```yml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp-hpa
  namespace: prod
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  minReplicas: 3
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
```

### 3. Istio VirtualService для canary-релиза
```yml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: myapp-vs
  namespace: prod
spec:
  hosts:
    - myapp
  http:
    - match:
        - headers:
            x-canary:
              exact: "true"
      route:
        - destination:
            host: myapp
            subset: v2
          weight: 100
    - route:
        - destination:
            host: myapp
            subset: v1
          weight: 90
        - destination:
            host: myapp
            subset: v2
          weight: 10
```

## Заключение

#### Предложенная архитектура на базе Kubernetes + Istio + Vault является отраслевым стандартом для построения масштабируемых, безопасных и управляемых микросервисных систем. Она:

* ✅ Закрывает все сформулированные требования.
* ✅ Проверена в production-средах крупных компаний (Google, Airbnb, Spotify).
* ✅ Имеет большое сообщество и документацию.
* ✅ Может быть реализована как в облаке (GKE, EKS, AKS), так и on‑premises.

#### Рекомендуемый стек для внедрения (MVP):
1. Kubernetes (база)
2. Istio (service mesh)
3. HashiCorp Vault (секреты)
4. ArgoCD (GitOps)
5. Prometheus + Grafana (мониторинг)

При ограниченном числе микросервисов (до 5–10) допускается упрощение: убрать Istio, использовать NGINX Ingress + External Secrets Operator для интеграции с любым хранилищем секретов.
