---
title: "Сбор custom metrics"
linkTitle: "Custom Metrics"
description: "Подключение собственных Prometheus exporters к tenant monitoring stack Cozystack с помощью VMServiceScrape и VMPodScrape."
weight: 15
---

## Обзор

Tenant monitoring в Cozystack поддерживает scraping custom metrics из ваших приложений и exporters. Tenant VMAgent обнаруживает scrape targets через labels Kubernetes namespaces, позволяя подключить любое приложение, которое предоставляет Prometheus-compatible endpoint `/metrics`.

В этом руководстве показано, как создать ресурсы `VMServiceScrape` и `VMPodScrape`, чтобы tenant VMAgent собирал ваши custom metrics и делал их доступными в Grafana.

## Предварительные требования

- Monitoring включен для вашего tenant (см. [Monitoring Setup]({{< ref "setup" >}}))
- Ваше приложение или exporter развернуты и предоставляют Prometheus-compatible endpoint `/metrics`
- У вас есть доступ к кластеру через `kubectl`

## Использование VMServiceScrape

`VMServiceScrape` указывает tenant VMAgent собирать метрики с endpoints за Kubernetes Service.

### Пример

Предположим, у вас есть Service `my-app` в namespace `my-app-ns`, который предоставляет метрики на port `metrics` по path `/metrics`:

```yaml
apiVersion: operator.victoriametrics.com/v1beta1
kind: VMServiceScrape
metadata:
  name: my-app-metrics
  namespace: my-app-ns
spec:
  selector:
    matchLabels:
      app: my-app
  endpoints:
    - port: metrics
      path: /metrics
      interval: 30s
```

Примените ресурс:

```bash
kubectl apply --filename vmservicescrape.yaml --namespace my-app-ns
```

### Ключевые поля

| Поле | Описание |
| --- | --- |
| `spec.selector.matchLabels` | Label selector для поиска целевого Service |
| `spec.endpoints[].port` | Именованный port на Service для scraping |
| `spec.endpoints[].path` | HTTP path для метрик (по умолчанию `/metrics`) |
| `spec.endpoints[].interval` | Scrape interval (по умолчанию наследуется от VMAgent, обычно `30s`) |

## Использование VMPodScrape

`VMPodScrape` собирает метрики напрямую с Pods без необходимости в Service. Это полезно для sidecar exporters или приложений, у которых нет соответствующего Service.

### Пример

Предположим, у вас есть Pods с label `app: my-worker`, которые предоставляют метрики на port `9090` по path `/metrics`:

```yaml
apiVersion: operator.victoriametrics.com/v1beta1
kind: VMPodScrape
metadata:
  name: my-worker-metrics
  namespace: my-app-ns
spec:
  selector:
    matchLabels:
      app: my-worker
  podMetricsEndpoints:
    - port: "9090"
      path: /metrics
```

Примените ресурс:

```bash
kubectl apply --filename vmpodscrape.yaml --namespace my-app-ns
```

### Ключевые поля

| Поле | Описание |
| --- | --- |
| `spec.selector.matchLabels` | Label selector для поиска целевых Pods |
| `spec.podMetricsEndpoints[].port` | Имя или номер port на Pod для scraping |
| `spec.podMetricsEndpoints[].path` | HTTP path для метрик (по умолчанию `/metrics`) |

## Проверка сбора метрик

После создания `VMServiceScrape` или `VMPodScrape` проверьте, что tenant VMAgent собирает данные с ваших targets.

### Проверьте targets VMAgent

Выведите pods VMAgent в namespace tenant и откройте страницу targets:

```bash
kubectl get pods --namespace <tenant-namespace> --selector app.kubernetes.io/name=vmagent
```

Сделайте port-forward к UI VMAgent, чтобы посмотреть активные targets:

```bash
kubectl port-forward --namespace <tenant-namespace> service/vmagent-vmagent 8429:8429
```

Затем откройте `http://localhost:8429/targets` в браузере. Новый scrape target должен появиться в списке со статусом `UP`.

### Запросите метрики в Grafana

1. Откройте Grafana по адресу `https://grafana.<tenant-host>`
2. Перейдите в **Explore**
3. Выберите datasource **VictoriaMetrics**
4. Выполните PromQL query для одной из custom metrics, например:

   ```promql
   up{job="my-app-ns/my-app-metrics"}
   ```

Результат `1` подтверждает, что target успешно scrape-ится.

## Устранение неполадок

- **Target не появляется в VMAgent**: проверьте, что namespace имеет label `namespace.cozystack.io/monitoring: <tenant-namespace>` и что `VMServiceScrape`/`VMPodScrape` создан в этом namespace
- **Target показывает status DOWN**: проверьте, что приложение запущено и metrics endpoint доступен на настроенном port и path
- **Метрик нет в Grafana**: убедитесь, что VMAgent пишет в правильный VMCluster, проверив logs VMAgent:

  ```bash
  kubectl logs --namespace <tenant-namespace> --selector app.kubernetes.io/name=vmagent
  ```
