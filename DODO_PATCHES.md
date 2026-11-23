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

Dockerfile и CI сборки находятся в
[dodopizza/helm, images/istio-cni](https://github.com/dodopizza/helm/tree/fix/istio-cni-cce-hbone/images/istio-cni).
Dockerfile должен ссылаться на конкретный коммит этого форка и контрольную
сумму исходного архива. Дополнительный патч во время сборки не применяется.

Проверки: весь пакет `cni/pkg/iptables` в изолированном Linux-контейнере,
включая golden IPv4/IPv6, reconciliation, cleanup и idempotency.
В m-sber проверены IPv4 same-node/cross-node, отчёты Kuberhealthy,
L7-маршрутизация через временный waypoint и разрешение/запрет запросов
AuthorizationPolicy. Работа с реальным IPv6-трафиком отдельно не проверена.

При обновлении upstream переносите изменение и ожидаемые результаты тестов
на новую версию, повторяйте проверки, обновляйте pinned commit/checksum в
Dockerfile и публикуйте образ под новым тегом. Существующие теги не заменять.
