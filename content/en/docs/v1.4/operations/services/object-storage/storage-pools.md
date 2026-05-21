---
title: "Storage Pools"
linkTitle: "Storage Pools"
description: "Настройка SeaweedFS storage pools для tiered object storage"
weight: 10
---

Storage pools позволяют разделять SeaweedFS volume servers по типу диска.
Каждый pool создает отдельный Volume StatefulSet с tag SeaweedFS `diskType` и соответствующий набор COSI resources (BucketClasses и BucketAccessClasses), на которые могут ссылаться buckets.

## Когда использовать pools

Используйте storage pools, когда в кластере есть разные storage tiers и нужно управлять тем, какой tier использует bucket.
Например, у вас могут быть быстрые NVMe-диски для hot data и большие HDD-диски для архивного хранения.

Если все volume servers используют одно и то же хранилище, pools не нужны: достаточно BucketClass по умолчанию.

## Включение SeaweedFS для tenant

Перед настройкой pools включите SeaweedFS для tenant:

```bash
kubectl patch -n tenant-root tenants.apps.cozystack.io root --type=merge -p '{
  "spec":{
    "seaweedfs": true
  }
}'
```

Дождитесь готовности SeaweedFS HelmRelease:

```bash
kubectl -n tenant-root get hr seaweedfs
```

Ожидаемый вывод:

```console
NAME        AGE   READY   STATUS
seaweedfs   2m    True    Helm upgrade succeeded for release tenant-root/seaweedfs.v1 with chart seaweedfs@...
```

## Конфигурация pool

Когда SeaweedFS запущен, пропатчите его HelmRelease, чтобы добавить storage pools:

```bash
kubectl patch -n tenant-root helmreleases.helm.toolkit.fluxcd.io seaweedfs --type=merge -p '{
  "spec":{
    "values":{
      "volume":{
        "pools":{
          "ssd":{
            "diskType": "ssd",
            "size": "50Gi",
            "storageClass": "local-nvme"
          },
          "hdd":{
            "diskType": "hdd",
            "size": "500Gi",
            "storageClass": "local-hdd",
            "replicas": 3
          }
        }
      }
    }
  }
}'
```

Эквивалентный полный ресурс выглядит так:

```yaml
apiVersion: apps.cozystack.io/v1alpha1
kind: SeaweedFS
metadata:
  name: seaweedfs
  namespace: tenant-example
spec:
  host: s3.example.com
  topology: Simple
  volume:
    replicas: 2
    size: 100Gi
    pools:
      ssd:
        diskType: ssd
        size: 50Gi
        storageClass: local-nvme
      hdd:
        diskType: hdd
        size: 500Gi
        storageClass: local-hdd
        replicas: 3
```

### Параметры pool

| Параметр | Обязательный | Описание |
| --- | --- | --- |
| `diskType` | Да | Tag типа диска SeaweedFS: lowercase alphanumeric, например `ssd`, `hdd`, `nvme` |
| `replicas` | Нет | Количество реплик volume server. По умолчанию `volume.replicas` |
| `size` | Нет | Размер PVC на реплику. По умолчанию `volume.size` |
| `storageClass` | Нет | Kubernetes StorageClass для PVC. По умолчанию `volume.storageClass` |
| `resources` | Нет | Явные CPU/memory limits. По умолчанию `volume.resources` |
| `resourcesPreset` | Нет | Sizing preset, когда `resources` не задан. По умолчанию `volume.resourcesPreset` |

### Правила именования

Имена pools должны быть валидными DNS labels: строчные буквы, цифры и hyphens. Следующие суффиксы зарезервированы и не должны использоваться как имена pools:

- Имена, заканчивающиеся на `-lock` (зарезервировано для object-lock BucketClasses)
- Имена, заканчивающиеся на `-readonly` (зарезервировано для read-only BucketAccessClasses)

## COSI resources, создаваемые для каждого pool

Каждый pool автоматически создает четыре COSI resources:

| Ресурс | Шаблон имени | Назначение |
| --- | --- | --- |
| BucketClass | `{namespace}-{pool}` | Стандартное создание bucket |
| BucketClass | `{namespace}-{pool}-lock` | Создание bucket с включенным object locking |
| BucketAccessClass | `{namespace}-{pool}` | Read-write credentials |
| BucketAccessClass | `{namespace}-{pool}-readonly` | Read-only credentials |

Например, pool с именем `ssd` в namespace `tenant-example` создает:

- BucketClass `tenant-example-ssd`
- BucketClass `tenant-example-ssd-lock`
- BucketAccessClass `tenant-example-ssd`
- BucketAccessClass `tenant-example-ssd-readonly`

{{< note >}}
Набор COSI resources по умолчанию без pool всегда создается с использованием только имени namespace, например `tenant-example`, `tenant-example-lock`.
Он соответствует volume servers, работающим с настройкой верхнего уровня `volume.diskType`.
{{< /note >}}

## MultiZone topology с pools

В MultiZone topology pools определяются для каждой зоны в `volume.zones[zone].pools`.

### Параметры зоны

Каждая зона, помимо pools, принимает следующие параметры:

| Параметр | Обязательный | Описание |
| --- | --- | --- |
| `replicas` | Нет | Количество реплик volume server в этой зоне. По умолчанию `volume.replicas` |
| `size` | Нет | Размер PVC на реплику. По умолчанию `volume.size` |
| `storageClass` | Нет | Kubernetes StorageClass для PVC. По умолчанию `volume.storageClass` |
| `dataCenter` | Нет | Имя data center в SeaweedFS. По умолчанию имя ключа zone, например zone `dc1` получает `dataCenter: dc1` |
| `nodeSelector` | Нет | YAML nodeSelector для планирования pods volume server. По умолчанию `topology.kubernetes.io/zone: <zoneName>` |
| `pools` | Нет | Map storage pools для этой зоны. Структура такая же, как у `volume.pools` |

### Пример

Пропатчите SeaweedFS HelmRelease, чтобы добавить pools для каждой зоны:

```bash
kubectl patch -n tenant-example helmreleases.helm.toolkit.fluxcd.io seaweedfs --type=merge -p '{
  "spec":{
    "values":{
      "volume":{
        "zones":{
          "dc1":{
            "pools":{
              "ssd":{"diskType": "ssd", "size": "50Gi"},
              "hdd":{"diskType": "hdd", "size": "500Gi"}
            }
          },
          "dc2":{
            "pools":{
              "ssd":{"diskType": "ssd", "size": "50Gi"},
              "hdd":{"diskType": "hdd", "size": "500Gi"}
            }
          }
        }
      }
    }
  }
}'
```

В этом примере zone `dc1` автоматически получает `dataCenter: dc1` и `nodeSelector: {topology.kubernetes.io/zone: dc1}`.

Чтобы переопределить значения по умолчанию, задайте их явно:

```yaml
volume:
  zones:
    us-east-1a:
      dataCenter: us-east
      nodeSelector:
        topology.kubernetes.io/zone: us-east-1a
        node.kubernetes.io/instance-type: storage-optimized
      pools:
        ssd:
          diskType: ssd
          size: 50Gi
```

Эквивалентный полный ресурс с явными настройками zones:

```yaml
apiVersion: apps.cozystack.io/v1alpha1
kind: SeaweedFS
metadata:
  name: seaweedfs
  namespace: tenant-example
spec:
  host: s3.example.com
  topology: MultiZone
  volume:
    replicas: 2
    size: 100Gi
    zones:
      dc1:
        pools:
          ssd:
            diskType: ssd
            size: 50Gi
          hdd:
            diskType: hdd
            size: 500Gi
      dc2:
        pools:
          ssd:
            diskType: ssd
            size: 50Gi
          hdd:
            diskType: hdd
            size: 500Gi
```

Каждая комбинация zone+pool создает собственный Volume StatefulSet.
В этом примере это четыре StatefulSets: `seaweedfs-volume-dc1-ssd`, `seaweedfs-volume-dc1-hdd`, `seaweedfs-volume-dc2-ssd` и `seaweedfs-volume-dc2-hdd`.

COSI resources дедуплицируются между зонами: если и `dc1`, и `dc2` определяют pool `ssd` с одинаковым `diskType`, создается только один набор ресурсов BucketClass/BucketAccessClass.

{{< alert color="warning" >}}
`volume.pools` верхнего уровня не разрешен в MultiZone topology. Вместо этого определяйте pools внутри каждой зоны.
{{< /alert >}}

## Проверка

После развертывания SeaweedFS с pools проверьте ресурсы:

```bash
# Проверьте, что volume server StatefulSets созданы для каждого pool
kubectl get statefulset -n tenant-example -l app.kubernetes.io/name=seaweedfs

# Проверьте BucketClasses
kubectl get bucketclass

# Проверьте BucketAccessClasses
kubectl get bucketaccessclass
```

Вы должны увидеть ресурсы BucketClass и BucketAccessClass для каждого имени pool.

## Связанная документация

- [Buckets]({{% ref "buckets" %}}) -- создание buckets, использующих конкретный storage pool
- [SeaweedFS Service Reference]({{% ref "/docs/v1.4/operations/services/seaweedfs" %}}) -- полный справочник параметров
- [SeaweedFS Multi-DC Configuration]({{% ref "/docs/v1.4/operations/stretched/seaweedfs-multidc" %}}) -- руководство по multi-DC развертыванию
