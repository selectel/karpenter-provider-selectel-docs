# 🚀 Установка и базовая настройка Karpenter

## 1. Требования

Перед установкой убедитесь, что у вас уже создан кластер в **Managed Kubernetes**.

Рекомедуется иcпользовать кластер с выключенными функциями автоскейла и автовосстановления, так как Karpenter самостоятельно управляет масштабированием нод.

📘 Как создать кластер:
https://docs.selectel.ru/managed-kubernetes/create/create-cloud-cluster/

- Kubernetes версии **1.28+**
- Karpenter запускается как pod, поэтому ему необходимы ресурсы в кластере  
  🔧 Минимальные требования:
  - одна нода с **2 vCPU** и **4 GiB RAM** для Karpenter

  💡 Рекомендуемые требования:
  - две ноды с **2 vCPU** и **4 GiB RAM** для Karpenter 
  - кластер с выключенными функциями автоскейла и автовосстановления

⚠️ Важно:
Karpenter не управляет нодами, которые он сам не создавал.

---

## 2. Установка

karpenter-provider-selectel - это провайдер Karpenter для Selectel Managed Kubernetes.
Он использует core-часть Karpenter версии v1.7.1.

Установите Karpenter через helm:

```bash
helm install karpenter-helmrelease oci://ghcr.io/selectel/mks-charts/karpenter:0.1.0 \
--namespace kube-system \
--set controller.settings.clusterID=$CLUSTER_ID
```

> CLUSTER_ID - идентификатор вашего кластера Managed Kubernetes. Можно установить как переменную в вашем окружении или передать строку c идентификатором напрямую в команде helm install.

---

## 3. Создание NodeClass (nodeclass.yaml)

Для начала необходимо настроить конфигурацию **сетевых дисков**, которые будут использоваться нодами в кластере.  
`NodeClass` определяет инфраструктурные параметры дисков.  
В одном кластере может быть несколько `NodeClass`, отличающихся, например, типом или размером диска.

**Список типов дисков:**  
<https://docs.selectel.ru/cloud-servers/volumes/about-network-volumes/#network-volume-types-list>

#### Пример `NodeClass`

```yaml
apiVersion: karpenter.k8s.selectel/v1alpha1
kind: SelectelNodeClass
metadata:
  name: default
spec:
  disk:
    # Тип (категория) сетевого диска
    categories:
      - universal
    # Размер диска в GiB
    sizeGiB: 30
```

---

## 4. Создание NodePool

После настройки `NodeClass` необходимо создать **NodePool** — объект, который описывает правила подбора и масштабирования нод.

Karpenter использует `NodePool`, чтобы определять:
- какие типы нод можно создавать,
- с какими флейворами и ресурсами,
- и когда эти ноды можно удалить или пересоздать.

Каждый `NodePool` ссылается на конкретный `NodeClass`, где указаны инфраструктурные параметры.

#### Пример `NodePool` (nodepool.yaml)

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
        # Сегменты пула, где разрешено создавать ноды
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
    # WhenEmptyOrUnderutilized — удалять или заменять ноды, если они пустые или недоиспользуются
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

**Подробнее о NodePool:** <https://karpenter.sh/docs/concepts/nodepools/>

### Применение в кластере

```bash
kubectl apply -f nodeclass.yaml
kubectl apply -f nodepool.yaml
```

---

## 5. Поддерживаемые ключи в `requirements`

По всем этим ключам можно **сортировать и фильтровать** подбор нод при масштабировании.  
Karpenter использует их для выбора подходящих типов инстансов (flavors) под требования подов.

| Ключ | Пример значения | Описание |
|---|---|---|
| **karpenter.sh/capacity-type** | `["on-demand"]`, `["spot"]` | Тип создаваемых нод — on-demand (классические) или spot (прерываемые). |
| **karpenter.k8s.selectel/instance-category** | `["SL", "GL"]` | Категория инстанса (Standard Line, GPU Line и т. д.). Подробнее: <https://docs.selectel.ru/cloud-servers/create/configurations/#server-flavors-list> |
| **karpenter.k8s.selectel/instance-family** | `["SL1", "GL1"]` | Семейство инстансов (линейка). |
| **karpenter.k8s.selectel/instance-generation** | `["1", "2"]` | Поколение семейства (например, второе поколение). |
| **karpenter.k8s.selectel/instance-cpu** | `Gt: "4"` | Количество vCPU (например, больше 4). |
| **karpenter.k8s.selectel/instance-memory** | `Gt: "8"` | Объем оперативной памяти (GiB). |
| **karpenter.k8s.selectel/instance-gpu-name** | `["A100", "H100"]` | Название GPU (для GPU-инстансов). Подробнее: <https://docs.selectel.ru/cloud-servers/create/gpus/> |
| **karpenter.k8s.selectel/instance-gpu-manufacturer** | `["NVIDIA"]` | Производитель GPU. |
| **karpenter.k8s.selectel/instance-gpu-count** | `Gt: "0"` | Количество GPU (например, больше одной видеокарты). |
| **karpenter.k8s.selectel/instance-local-disk** | `["true"]`, `["false"]` | Наличие локального диска (NVMe/SSD). |

**Примеры использования:**
- Создавать только GPU-ноды, у которых больше одной видеокарты.
- Исключать старые поколения, задав `instance-generation > 1`.
- Использовать только инстансы с локальным диском.

---

## 6. Готово

Теперь Karpenter настроен и готов к работе.  
После появления подов, требующих дополнительных ресурсов, Karpenter автоматически создаст ноды, соответствующие условиям из `NodePool` + `NodeClass`.
