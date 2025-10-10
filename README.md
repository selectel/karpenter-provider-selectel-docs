# 🚀 Автомасштабирование с помощью Karpenter

В кластере [Managed Kubernetes](https://docs.selectel.ru/managed-kubernetes/) для автомасштабирования можно использовать Karpenter.

Так же как и Cluster Autoscaler, который установлен по умолчанию и используется, когда вы [включаете автомасштабирование](https://doc-5069-dbaas-backups-in-opensearch-reapply.docs-v3.stg.sites.selectel.org/managed-kubernetes/node-groups/cluster-autoscaler/#enable-autoscaling) в панели управления, Karpenter помогает оптимально использовать ресурсы кластера — в зависимости от нагрузки на кластер количество нод в группе будет автоматически уменьшаться или увеличиваться. В отличие от Cluster Autoscaler Karpenter позволяет более гибко настраивать параметры масштабирования. Например, после настройки Karpenter может:

- создавать только ноды, у которых больше одной видеокарты;
- исключать старые поколения линеек;
- использовать только конфигурации с локальным диском.

У Karpenter есть информация о доступных конфигурациях и об их стоимости. Он знает, какие ресурсы нужны клиентскому приложению и автоматически подбирает оптимальную конфигурацию. Karpenter может управлять только нодами, которые он создал.

Кластер, где вы будете использовать Karpenter, должен соответствовать [требованиям](#requirements). По умолчанию Karpenter не установлен в кластере, для начала работы [установите Karpenter](#install-karpenter) и [настройте](#configure-karpenter) его.

## Требования {#requirements}

1. Кластер с версией Kubernetes 1.28 и выше. Вы можете [обновить версию кластера](https://docs.selectel.ru/managed-kubernetes/clusters/upgrade-version/).
2. В кластере минимум одна нода, в которой не менее 2 vCPU и 4 GiB RAM.

Для оптимальной работы Karpenter рекомендуем:

- добавить в кластер две ноды, в каждой из которых не менее 2 vCPU и 4 GiB RAM;
- выключить [автомасштабирование](https://docs.selectel.ru/managed-kubernetes/node-groups/cluster-autoscaler/) и [автовосстановление](https://docs.selectel.ru/managed-kubernetes/node-groups/reinstall-nodes/).

## Установить Karpenter {#install-karpenter}

Экспортируйте в переменную окружения KUBECONFIG путь к kubeconfig-файлу.

```bash
export KUBECONFIG=<path>
```

Укажите <path> — путь к kubeconfig-файлу имя_кластера.yaml

Добавьте официальный репозиторий Helm и установите Karpenter:

```bash
helm install karpenter-helmrelease oci://ghcr.io/selectel/mks-charts/karpenter:0.1.0 \
--namespace kube-system \
--set controller.settings.clusterID=$(<cluster_id>)
```

Укажите `<cluster_id>` — ID кластера Managed Kubernetes, можно посмотреть в [панели управления](https://my.selectel.ru/vpc/default/mks/): в верхнем меню нажмите **Продукты** → **Managed Kubernetes** → страница кластера → скопируйте ID под именем кластера, рядом с регионом и пулом.

## Настроить Karpenter {#configure-karpenter}

## 1. Создать NodeClass

`NodeClass` определяет инфраструктурные параметры зсетевых дисков, которые будут использоваться нодами в кластере.
В одном кластере может быть несколько разных NodeClass, например которые отличаются типом или размером диска. Если вы используете локальные загрузочные диски

1. Создайте yaml-файл `nodeclass.yaml` с манифестом для объекта NodeClass.

  Пример манифеста NodeClass для универсального сетевого диска:

  ```yaml
  apiVersion: karpenter.k8s.selectel/v1alpha1
  kind: SelectelNodeClass
  metadata:
    name: default
  spec:
    disk:
      categories:
        - universal
      sizeGiB: 30
  ```

  Здесь:

  - `сфеупщкшуы` — [тип сетевого диска](https://docs.selectel.ru/cloud-servers/volumes/about-network-volumes/#network-volume-types-list). ;
  - `30` — размер сетевого диска в ГБ.

2. Примените манифест:

  ```bash
  kubectl apply -f nodeclass.yaml
  ```

---

## 2. Создать NodePool

NodePool описывает правила подбора и масштабирования нод. Например:

- какие типы нод можно создавать;
- с какими флейворами и ресурсами;
- когда эти ноды можно удалить или пересоздать.

Каждый NodePool ссылается на конкретный NodeClass, где указаны параметры сетевых дисков. Подробнее о NodePool в статье [NodePools](https://karpenter.sh/docs/concepts/nodepools/) документации Karpenter.

1. Создайте yaml-файл `nodepool.yaml` с манифестом для объекта NodePool.

  Пример манифеста NodePool:

  ```yaml
  apiVersion: karpenter.sh/v1
  kind: NodePool
  metadata:
    name: default
  spec:
    template:
      spec:
        # Ссылка на NodeClass
        nodeClassRef:
          name: default
          kind: SelectelNodeClass
          group: karpenter.k8s.selectel
        # Требования (фильтры) для создаваемых нод
        requirements:
          # Сегменты (зоны), где разрешено создавать ноды
          - key: topology.kubernetes.io/zone
            operator: In
            values: ["ru-7a", "ru-7b"]
          # Допустимые конфигурации (flavors) из облака
          - key: node.kubernetes.io/instance-type
            operator: In
            values: ["SL1.1-2048", "SL1.2-4096", "SL1.2-8192"]
          # Тип создаваемых нод (on-demand, spot)
          - key: karpenter.sh/capacity-type
            operator: In
            values: ["on-demand"]
    # Настройки политики "жизни" нод
    disruption:
      # WhenEmptyOrUnderutilized — удалять или заменять ноды, если они пустые или используются не в полном объеме
      consolidationPolicy: WhenEmptyOrUnderutilized
      # Время ожидания перед консолидацией после изменения нагрузки
      consolidateAfter: 0s
      # Время жизни ноды (30 дней)
      expireAfter: 720h
    # Общие лимиты на ресурсы пула
    limits:
      cpu: "1000"       # Максимум 1000 vCPU на все ноды пула
      memory: 1000Gi    # Максимум 1000 GiB памяти
  ```

2. Примените манифест:

  ```bash
  kubectl apply -f nodepool.yaml
  ```

---

### Поддерживаемые ключи в `requirements`

Karpenter использует ключи для выбора конфигураций, которые подходят под требования подов.

| Ключ | Пример значения | Описание |
|---|---|---|
| **karpenter.sh/capacity-type** | `["on-demand"]`, `["spot"]` | Тип создаваемых нод: on-demand (классические) или spot (прерываемые)|
| **karpenter.k8s.selectel/instance-category** | `["SL", "GL"]` | Категория инстанса (Standard Line, GPU Line и т. д.). Подробнее: <https://docs.selectel.ru/cloud-servers/create/configurations/#server-flavors-list> |
| **karpenter.k8s.selectel/instance-family** | `["SL1", "GL1"]` | Линейка конфигураций |
| **karpenter.k8s.selectel/instance-generation** | `["1", "2"]` | Поколение линейки|
| **karpenter.k8s.selectel/instance-cpu** | `Gt: "4"` | Количество vCPU (например, больше 4) |
| **karpenter.k8s.selectel/instance-memory** | `Gt: "8"` | Объем оперативной памяти в ГиБ|
| **karpenter.k8s.selectel/instance-gpu-name** | `["A100", "H100"]` | Название GPU для конфигураций с GPU. Доступные GPU можно посмотреть в инструкции [Графические процессоры (GPU)](https://docs.selectel.ru/cloud-servers/create/gpus/)|
| **karpenter.k8s.selectel/instance-gpu-manufacturer** | `["NVIDIA"]` | Производитель GPU |
| **karpenter.k8s.selectel/instance-gpu-count** | `Gt: "0"` | Количество GPU (например, больше одной видеокарты) |
| **karpenter.k8s.selectel/instance-local-disk** | `["true"]`, `["false"]` | Наличие локального загрузочного диска (NVMe/SSD) |
