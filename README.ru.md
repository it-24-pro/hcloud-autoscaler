# hcloud-autoscaler

Helm chart для полноценного развёртывания Cluster Autoscaler на Hetzner Cloud с поддержкой Rancher/RKE2.

> **[Read in English →](README.md)**

## Что включено

| Компонент | Описание |
|---|---|
| **cluster-autoscaler** | Deployment с hetzner cloud provider, RBAC, Secret с hcloud конфигом |
| **node-finalizer-cleaner** | CronJob каждые 2 минуты снимает Rancher `wrangler.cattle.io/*` finalizers с нод помеченных к удалению autoscaler'ом |
| **protected-nodes hook** | Post-install Job: добавляет `scale-down-disabled=true` на перманентные ноды (masters, VMware workers) |

## Проблема которую решает node-finalizer-cleaner

При удалении hcloud VM через cluster-autoscaler нода становится `unreachable`.
Rancher добавляет финалайзеры `wrangler.cattle.io/node` и `wrangler.cattle.io/managed-etcd-controller`
на все ноды, которые блокируют удаление объекта ноды в Kubernetes.
В результате нода висит часами в состоянии `NotReady,SchedulingDisabled`.

`node-finalizer-cleaner` ждёт `graceSeconds` (default: 300s) и затем:
1. Снимает finalizers
2. Удаляет объект ноды
3. *(Опционально)* Удаляет запись machine из Rancher UI через API

**Условия срабатывания** (оба должны выполняться):
- тейнт `ToBeDeletedByClusterAutoscaler` — autoscaler пометил ноду
- тейнт `node.kubernetes.io/unreachable` — нода недоступна

## Установка

```bash
# Создать values.local.yaml с реальными данными (не коммитить!)
cp values.yaml values.local.yaml
# Заполнить hcloud.token, hcloud.sshKey, hcloud.network, cloudInitBase64

helm upgrade --install hcloud-autoscaler . \
  -n cluster-autoscaler \
  --create-namespace \
  -f values.yaml \
  -f values.local.yaml
```

## Node Groups — как это работает

Конфигурация группы нод задаётся в **двух местах** values.yaml:

### 1. `autoscaler.nodeGroups` — лимиты и тип инстанса

Говорит autoscaler'у какие группы отслеживать и в каких границах масштабировать:

```yaml
autoscaler:
  nodeGroups:
    - name: cx23-workers    # имя группы (должно совпадать с ключом в hcloud.nodeGroups)
      instanceType: cx23    # тип инстанса Hetzner (cx22, cx32, cx52 и т.д.)
      region: fsn1          # регион (fsn1, nbg1, hel1, ash, sin)
      min: 1                # минимум нод (0 = группа может быть пустой)
      max: 5                # максимум нод
```

### 2. `hcloud.nodeGroups` — что создавать в Hetzner

Конфиг для Hetzner API: какой образ, SSH ключ, сеть и главное — cloud-init скрипт:

```yaml
hcloud:
  token: ""
  image: ubuntu-24.04
  sshKey: ""        # имя SSH ключа в Hetzner
  network: ""       # имя private network
  nodeGroups:
    cx23-workers:
      cloudInitBase64: ""   # base64 от cloud-init скрипта регистрации в Rancher
    cx33-workers:
      cloudInitBase64: ""
```

Получить `cloudInitBase64`:
```bash
base64 -w0 examples/cloud-init-cx23.yaml
```

### Принцип масштабирования

**Scale-up:**
1. Autoscaler обнаруживает Pending поды
2. Симулирует, на какую группу они поместятся
3. Выбирает группу с наименьшим расходом (waste CPU/memory)
4. Увеличивает size через Hetzner API → VM создаётся → cloud-init регистрирует ноду в Rancher
5. Поды назначаются на новую ноду

**Scale-down:**
1. Нода пустая дольше `scaleDownUnneededTime` (default: 5m)
2. Прошёл cooldown после последнего scale-up (`scaleDownDelayAfterAdd`: 5m)
3. Autoscaler ставит тейнт `ToBeDeletedByClusterAutoscaler`, дрейнит ноду, удаляет VM в Hetzner
4. Нода становится `unreachable` → Rancher вешает wrangler finalizers
5. `node-finalizer-cleaner` через 300s снимает finalizers, удаляет объект ноды и machine из Rancher UI

**Ноды защищённые от scale-down** (`protectedNodes.nodes`) получают аннотацию
`cluster-autoscaler.kubernetes.io/scale-down-disabled=true` через post-install hook.

### Добавить новую группу нод

1. Создать cloud-init скрипт по образцу `examples/cloud-init-cx23.yaml` (изменить `instanceType` в labels)
2. Добавить в `autoscaler.nodeGroups`:
   ```yaml
   - name: cx42-workers
     instanceType: cx42
     region: fsn1
     min: 0
     max: 2
   ```
3. Добавить в `hcloud.nodeGroups`:
   ```yaml
   cx42-workers:
     cloudInitBase64: "<base64 от нового cloud-init>"
   ```
4. `helm upgrade hcloud-autoscaler . -f values.yaml -f values.local.yaml`

## Интеграция с Rancher (опционально)

При `rancher.enabled: true` после удаления k8s node cleaner дополнительно вызывает Rancher API
и удаляет запись `Nodenotfound` / `Failed` machine из Rancher UI.

```yaml
# values.local.yaml
rancher:
  enabled: true
  url: "https://your-rancher.example.com"
  token: "token-xxxxx:yyyyyyy"   # Rancher → Account & API Keys → Create API Key
  clusterId: "c-m-xxxxxxxx"      # виден в URL при открытии кластера в Rancher UI
```

## cloud-init

cloud-init для нод должен:
1. Настроить default route через `10.102.24.1` (hcloud private network gateway)
2. Настроить DNS
3. Получить hcloud server ID из metadata и записать в RKE2 config как `provider-id=hcloud://<id>`
4. Добавить node labels: `hcloud/node-group=<name>`, `node.kubernetes.io/instance-type=<type>`
5. Запустить Rancher system-agent для регистрации в кластере

Пример cloud-init смотри в `examples/cloud-init-cx23.yaml`.

## Параметры

| Параметр | Default | Описание |
|---|---|---|
| `autoscaler.nodeGroups` | cx23(1-5), cx33(0-3) | Node groups конфигурация |
| `autoscaler.scaleDownUnneededTime` | `5m` | Через сколько удалять ненужные ноды |
| `autoscaler.scaleDownDelayAfterAdd` | `5m` | Задержка scale-down после добавления ноды |
| `finalizerCleaner.enabled` | `true` | Включить cleaner |
| `finalizerCleaner.graceSeconds` | `300` | Grace period перед чисткой финалайзеров |
| `finalizerCleaner.schedule` | `*/2 * * * *` | Расписание CronJob |
| `rancher.enabled` | `false` | Включить чистку Rancher machines |
| `rancher.url` | `""` | URL Rancher |
| `rancher.clusterId` | `""` | ID кластера в Rancher (напр. `c-m-5g86mm7f`) |
| `protectedNodes.nodes` | masters + workers | Ноды защищённые от scale-down |

## Требования

- Kubernetes 1.28+
- Rancher/RKE2
- hcloud private network для нод (напр. 10.102.24.0/24)
- hcloud API token с правами Read/Write на Servers, Networks, SSHKeys

## Структура

```
hcloud-autoscaler/
├── Chart.yaml
├── values.yaml              # Публичные значения (без секретов)
├── values.local.yaml        # Реальные данные — в .gitignore!
├── .gitignore
├── templates/
│   ├── namespace.yaml
│   ├── autoscaler-secret.yaml
│   ├── autoscaler-rbac.yaml
│   ├── autoscaler-deployment.yaml
│   ├── serviceaccount.yaml
│   ├── rbac.yaml
│   ├── configmap.yaml
│   ├── rancher-secret.yaml
│   ├── finalizer-cleaner-cronjob.yaml
│   └── protected-nodes-hook.yaml
└── examples/
    └── cloud-init-cx23.yaml
```

---

## Профессиональная поддержка

Нужна помощь с развёртыванием, настройкой Kubernetes инфраструктуры или интеграцией Hetzner/Rancher?

Мы предоставляем профессиональные DevOps и инфраструктурные услуги:

- 🌐 **[nexops.eu](https://nexops.eu/en/)** — DevOps & Cloud инжиниринг (EN)
- 🌐 **[it-24.pro](https://it-24.pro/)** — IT инфраструктура и managed services (RU/UA)

Открывайте issue или пишите напрямую.
