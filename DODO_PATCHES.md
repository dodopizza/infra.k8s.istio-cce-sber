# Изменения Dodo для CCE-Sber

Это форк официального [istio/istio](https://github.com/istio/istio).
Ветка `cce-sber-1.29.10` основана на теге Istio `1.29.10`, коммит
`cca610c61900bd48b5a7b3ee3d8af19ff265c495`. Go module остаётся
`istio.io/istio`: переименование GitHub-репозитория не меняет пути Go imports.

## Исправление локального HBONE через OVS

В сети CCE-Sber Open vSwitch сохраняет packet mark `0x539` при передаче
между pod одной ноды. Правило Istio CNI назначает HBONE-транспорту connmark
`0x111`, из-за чего ответный SYN-ACK направляется через таблицу 100 в loopback.

В `cni/pkg/iptables/iptables.go` перед назначением app-facing CONNMARK
добавлено исключение для TCP source/destination port 15008. Правило должно
действовать на обоих концах соединения. Обновлены 12 upstream golden fixtures
в `cni/pkg/iptables/testdata`, включая IPv4 и IPv6.

Исключение также затрагивает обычные приложения, использующие порт 15008.
Это локальный workaround для этой сетевой модели, не официальный фикс Istio.

## Сборка и проверки

Сборка полностью находится в этом форке:

- Dockerfile: `cni/deployments/kubernetes/Dockerfile.install-cni-cce-sber`.
- Версия образа: `cni/deployments/kubernetes/cce-sber-cni.version`.
- Workflow: `.github/workflows/build-cce-sber-cni.yml`.

Dockerfile собирает исходники из текущего checkout, без скачивания другой
копии репозитория или применения отдельного патча. Workflow передаёт SHA
checkout в BuildInfo обоих бинарников и OCI labels.

Push/PR запускают тесты, сборку amd64 runtime, smoke checks обоих бинарников
и сохраняют проверенный образ как Actions artifact на 7 дней.
Ручной запуск с `publish=true` дополнительно публикует тот же проверенный образ
в `cr.yandex/crpv7tk7maoqsq3usck3/istio/install-cni`.
Для этого нужен repository secret `YCR_CNI_SA_KEY_BASE64` — JSON-ключ сервисного
аккаунта Yandex в base64 с правом записи в registry. Значение добавляется через
GitHub Settings → Secrets and variables → Actions, а не в исходники.
Существующие теги не перезаписываются; для публикации увеличьте номер ревизии
в `cce-sber-cni.version`. Ревизия `.2` опубликована из этого форка;
пошаговые команды обновления находятся в [README](README.md#обновление-образа-cni-для-cce-sber).

В Helm остаётся только потребитель: pinned image digest в
`charts/istio-control-plane/values-cce-sber.yaml`.
После новой публикации скопируйте digest из workflow summary в профиль чарта.

Локальная проверка из корня форка:

```sh
docker buildx build --target tests --load \
  -f cni/deployments/kubernetes/Dockerfile.install-cni-cce-sber -t cni-tests .
docker run --rm --privileged cni-tests -test.v -test.timeout=180s

docker buildx build --platform linux/amd64 --target runtime --load \
  --build-arg CNI_VERSION="$(cat cni/deployments/kubernetes/cce-sber-cni.version)" \
  --build-arg SOURCE_REVISION="$(git rev-parse HEAD)" \
  -f cni/deployments/kubernetes/Dockerfile.install-cni-cce-sber -t cni-runtime .
```

Тестовый контейнер привилегирован для изолированных netns/userns тестов.
Не запускайте этот тестовый бинарник в production-pod или namespace хоста.

Проверки: весь пакет `cni/pkg/iptables` в изолированном Linux-контейнере,
включая golden IPv4/IPv6, reconciliation, cleanup и idempotency.
В m-sber проверены IPv4 same-node/cross-node, отчёты Kuberhealthy,
L7-маршрутизация через временный waypoint и разрешение/запрет запросов
AuthorizationPolicy. Работа с реальным IPv6-трафиком отдельно не проверена.

Команды перехода на новый upstream minor находятся в
[README](README.md#переход-на-новый-upstream-minor-например-130).
