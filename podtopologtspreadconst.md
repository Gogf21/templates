# Topology Spread Constraints в Kubernetes

## Что это

`topologySpreadConstraints` — механизм планировщика, который равномерно распределяет Pod'ы по доменам топологии (зонам, нодам, регионам) с настраиваемым допуском.

Ключевое отличие от антиаффинности: это не «запрет», а баланс. Планировщик не блокирует размещение, а следит за тем, чтобы разница между доменами не превышала заданный порог.

---

## Полный манифест

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test-deploy
  labels:
    app.kubernetes.io/name: &name test-dp
    app.kubernetes.io/instance: &instance test-dp
    app.kubernetes.io/version: &version v0.0.1
spec:
  replicas: 3
  selector:
    matchLabels:
      app.kubernetes.io/name: *name
      app.kubernetes.io/instance: *instance
      app.kubernetes.io/version: *version
  template:
    metadata:
      labels:
        app.kubernetes.io/name: *name
        app.kubernetes.io/instance: *instance
        app.kubernetes.io/version: *version
    spec:
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app.kubernetes.io/name: test-dp
              app.kubernetes.io/instance: test-dp
      containers:
        - name: whoami
          image: traefik/whoami
          ports:
            - containerPort: 80
              name: http
              protocol: TCP
          resources:
            limits:
              cpu: 100m
              memory: 100Mi
            requests:
              cpu: 100m
              memory: 100Mi
          livenessProbe:
            httpGet:
              path: /healthz
              port: http
              scheme: HTTP
            initialDelaySeconds: 15
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /healthz
              port: http
              scheme: HTTP
            initialDelaySeconds: 15
            periodSeconds: 10
```

---

## Разбор полей

| Поле | Значение | Смысл |
|---|---|---|
| `maxSkew` | `1` | Максимально допустимая разница в количестве Pod'ов между доменами |
| `topologyKey` | `topology.kubernetes.io/zone` | Считаем зоны. Можно заменить на `kubernetes.io/hostname` для нод |
| `whenUnsatisfiable` | `DoNotSchedule` | Жёстко: если правило нарушится — Pod останется `Pending` |
| `labelSelector` | `app.kubernetes.io/name: test-dp` | Какие Pod'ы считать «своими» при подсчёте |

### Значения `whenUnsatisfiable`

| Значение | Поведение |
|---|---|
| `DoNotSchedule` | Жёсткое правило. Нарушение → Pod в `Pending` |
| `ScheduleAnyway` | Мягкое правило. Планировщик старается, но не блокирует |

---

## Отличие от антиаффинности

| | `podAntiAffinity` (`required` + `hostname`) | `topologySpreadConstraints` |
|---|---|---|
| Логика | Запрет: «не более одного Pod на ноду» | Баланс: «разница между доменами ≤ maxSkew» |
| Поведение при нехватке нод | Pod зависает в `Pending` | Pod'ы всё равно запускаются (с допустимым перекосом) |
| Пример: 3 ноды, 10 реплик | Максимум 3 Pod'а, 7 в `Pending` | Распределение 4-3-3, всё запущено |
| Пример: 2 ноды, 3 реплики | Максимум 2 Pod'а, 1 в `Pending` | Распределение 2-1, всё запущено |

Вывод: `topologySpreadConstraints` гибче — она не блокирует планирование, а управляет балансом.

---

## Варианты конфигурации

### 1. По зонам (жёстко)

```yaml
topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        app.kubernetes.io/name: test-dp
```

### 2. По нодам (жёстко)

```yaml
topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: kubernetes.io/hostname
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        app.kubernetes.io/name: test-dp
```

Ближе к антиаффинности, но гибче: если нод 2, а реплик 3, распределение будет `2-1`, и Pod'ы всё равно запустятся.

### 3. По нодам (мягко)

```yaml
topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: kubernetes.io/hostname
    whenUnsatisfiable: ScheduleAnyway
    labelSelector:
      matchLabels:
        app.kubernetes.io/name: test-dp
```

Планировщик постарается разнести Pod'ы, но не заблокирует, если не получится.

---

## Комбинирование с nodeAffinity

Можно ограничить зону поиска через `nodeAffinity`, а раскидывать — через `topologySpreadConstraints`. Планировщик применит оба правила (AND):

```yaml
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: special
                operator: In
                values: [ds-only]
  topologySpreadConstraints:
    - maxSkew: 1
      topologyKey: kubernetes.io/hostname
      whenUnsatisfiable: DoNotSchedule
      labelSelector:
        matchLabels:
          app.kubernetes.io/name: test-dp
```

Pod'ы будут ставиться только на ноды с `special=ds-only`, а внутри этого набора — равномерно распределяться.

---

## Полезные нюансы

### minDomains

Заставляет планировщик ждать появления новых нод, если подходящих доменов меньше заданного числа. Работает только с `DoNotSchedule`.

```yaml
topologySpreadConstraints:
  - maxSkew: 1
    minDomains: 3
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        app.kubernetes.io/name: test-dp
```

### matchLabelKeys: [pod-template-hash]

Считает skew отдельно для каждой ревизии Deployment. Это нужно, чтобы при rolling update новый ReplicaSet распределялся независимо от старого.

```yaml
topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: kubernetes.io/hostname
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        app.kubernetes.io/name: test-dp
    matchLabelKeys:
      - pod-template-hash
```

### Namespace

Pod'ы для подсчёта берутся только из того же namespace. Если у тебя несколько namespace'ов с одинаковыми метками, они не будут влиять друг на друга.

### Ноды без topologyKey

Игнорируются при подсчёте. Если на ноде нет метки `topology.kubernetes.io/zone`, она не участвует в распределении.

### Порядок вычислений

1. Планировщик собирает все Pod'ы, подходящие под `labelSelector`, во всех namespace'ах (но только в своём).
2. Группирует их по значению `topologyKey` на нодах.
3. Считает skew — разницу между максимальным и минимальным количеством Pod'ов в домене.
4. Если добавление Pod'а в домен не превысит `maxSkew` — домен подходит.

---

## Проверка

```bash
kubectl apply -f 06-topologyspread.yaml
kubectl get pods -l app.kubernetes.io/name=test-dp -o wide
kubectl describe pod <pod-name> | tail -15
```

Если Pod'ы в `Pending` — смотри Events. Типичное сообщение:

```
0/3 nodes are available: 3 node(s) didn't match pod topology spread constraints
```

### Полезные команды для отладки

```bash
# Где какие Pod'ы
kubectl get pods -l app.kubernetes.io/name=test-dp -o wide

# Какие зоны у нод
kubectl get nodes -L topology.kubernetes.io/zone

# События по namespace
kubectl get events --sort-by=.lastTimestamp | tail -20
```

---

## Шпаргалка

| Что | Как |
|---|---|
| Распределить по зонам | `topologyKey: topology.kubernetes.io/zone` |
| Распределить по нодам | `topologyKey: kubernetes.io/hostname` |
| Жёсткое правило | `whenUnsatisfiable: DoNotSchedule` |
| Мягкое правило | `whenUnsatisfiable: ScheduleAnyway` |
| Максимальный перекос | `maxSkew: 1` (чем больше, тем свободнее) |
| Учитывать только свою ревизию | `matchLabelKeys: [pod-template-hash]` |
| Ждать минимум доменов | `minDomains: N` (только с `DoNotSchedule`) |
| Ограничить ноды | комбинировать с `nodeAffinity` |

---

## Итог

`topologySpreadConstraints` — это балансировщик, а не блокировщик. Он отвечает на вопрос «как раскидать Pod'ы равномерно», тогда как антиаффинность отвечает на вопрос «где им запрещено быть».

- Нужен жёсткий запрет «два Pod'а на одной ноде» → `podAntiAffinity` с `required`.
- Нужно равномерное распределение с гибкостью → `topologySpreadConstraints` с `maxSkew`.
- Нужно и то, и другое → комбинируй оба механизма.