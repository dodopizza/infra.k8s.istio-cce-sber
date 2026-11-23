# Istio

## Переход на новый upstream minor, например 1.30

Цель: взять новый официальный Istio, перенести наше исправление для сети Sber,
собрать CNI и подключить образ через Helm/Terraform. Сам форк собирает только
CNI; istiod, ztunnel и остальные компоненты обновляются через зависимости чарта.
`1.30.0` ниже — пример, используйте реально опубликованный полный тег релиза.

Понадобятся Git, GitHub CLI (`gh`), Docker с Buildx и Helm.
Доступ для публикации уже настроен через Actions secret `YCR_CNI_SA_KEY_BASE64`.
Все команды до шага 6 выполняйте в каталоге этого форка, в одном терминале.

### 1. Получить официальный релиз

`origin` — наш форк, `upstream` — официальный репозиторий Istio.
Проверьте `git status --short`: вывод должен быть пустым, иначе сначала
сохраните текущую работу. Если `upstream` отсутствует в `git remote -v`, добавьте:

```sh
git remote add upstream https://github.com/istio/istio.git
```

Создайте отдельную ветку от официального тега. Старую ветку оставьте для отката:

```sh
UPSTREAM_VERSION=1.30.0
CNI_BRANCH="cce-sber-$UPSTREAM_VERSION"
git fetch origin
git fetch upstream "refs/tags/$UPSTREAM_VERSION:refs/tags/$UPSTREAM_VERSION"
git switch -c "$CNI_BRANCH" "$UPSTREAM_VERSION"
```

Если тег не найден, проверьте номер релиза; остальные команды пока не выполняйте.

### 2. Перенести исправление и файлы сборки

Патч исключает HBONE-соединения из правила, которое ошибочно назначало им
метку соединения. Команда переносит только этот код и ожидаемые результаты
тестов (`golden`), без старых исходников Istio:

```sh
git show b09a841ecea29c9baebb2cdf38cd3ce6c60ceaa5 --format= -- cni/pkg/iptables |
  git apply --3way
```

Если появились конфликты, остановитесь: нужно сопоставить новый upstream-код
с патчем и обновить тесты. Успешное применение патча само по себе не доказывает
его совместимость. Если upstream уже устранил проблему, сначала проверьте
воспроизведение без нашего исправления.

Скопируйте из предыдущей ветки workflow, Dockerfile и документацию:

```sh
git restore --source origin/cce-sber-1.29.10 -- \
  .dockerignore .github/workflows/build-cce-sber-cni.yml \
  cni/deployments/kubernetes/Dockerfile.install-cni-cce-sber \
  cni/deployments/kubernetes/cce-sber-cni.version README.md DODO_PATCHES.md
printf '%s\n' "$UPSTREAM_VERSION-cce-sber-hbone.1" > cni/deployments/kubernetes/cce-sber-cni.version
```

`1.30.0` — версия официального Istio; `.1` после `hbone` — первая наша сборка
для этой версии. Повторная публикация требует `.2`, затем `.3` и т. д.
Существующие теги образов не перезаписываются.

### 3. Настроить сборку для новой версии

Обновите файлы вручную:

| Файл | Что изменить |
| --- | --- |
| `.github/workflows/build-cce-sber-cni.yml` | Обе настройки `branches` в `push` и `pull_request`: заменить `cce-sber-1.29.10` на `cce-sber-1.30.0`. |
| `cni/deployments/kubernetes/Dockerfile.install-cni-cce-sber` | В `ARG RUNTIME_IMAGE` указать `docker.io/istio/install-cni:1.30.0-distroless@sha256:НОВЫЙ_DIGEST`. |
| Тот же Dockerfile, стадия `source` | Проверить версию Go по новому `go.mod` и требованиям сборки upstream. При необходимости обновить Go-образ и его digest. |
| Тот же Dockerfile, label `io.dodopizza.istio.upstream-revision` | Указать SHA нового официального тега. |
| `DODO_PATCHES.md` и эта инструкция | Обновить базовую версию, SHA и имя предыдущей ветки для следующего обновления. |

Digest — неизменяемый идентификатор содержимого образа. Получите его и SHA тега:

```sh
docker buildx imagetools inspect "istio/install-cni:$UPSTREAM_VERSION-distroless"
git rev-parse "$UPSTREAM_VERSION^{commit}"
rg '^(go|toolchain) ' go.mod
```

В выводе первой команды возьмите верхний `Digest: sha256:…`.
Для нового Go-образа получите digest такой же командой, подставив выбранный
`golang:<версия>-bookworm`. Не оставляйте digest от старого тега.
Проверьте также, что новый официальный CNI-образ по-прежнему использует пути
`/usr/local/bin/install-cni` и `/opt/cni/bin/istio-cni`, заданные в нашем Dockerfile.

### 4. Проверить сборку в GitHub Actions

```sh
git add cni/pkg/iptables .dockerignore .github/workflows/build-cce-sber-cni.yml \
  cni/deployments/kubernetes/Dockerfile.install-cni-cce-sber \
  cni/deployments/kubernetes/cce-sber-cni.version README.md DODO_PATCHES.md
git diff --cached --check
git diff --cached --stat
git commit -m "build(cni): port CCE-Sber workaround to Istio $UPSTREAM_VERSION"
git push -u origin "$CNI_BRANCH"
gh run list --repo dodopizza/infra.k8s.istio-cce-sber --branch "$CNI_BRANCH" \
  --workflow build-cce-sber-cni.yml --limit 1
```

Push запускает тесты iptables, сборку `linux/amd64` и проверку запуска обоих
бинарников. Образ пока не публикуется. Из списка возьмите ID запуска:

```sh
gh run watch <RUN_ID> --repo dodopizza/infra.k8s.istio-cce-sber --exit-status
```

`<RUN_ID>` заменяйте числом, без угловых скобок. Продолжайте после результата
`success`. При ошибке откройте лог:

```sh
gh run view <RUN_ID> --repo dodopizza/infra.k8s.istio-cce-sber --log-failed
```

### 5. Опубликовать образ в Yandex Container Registry

```sh
gh workflow run build-cce-sber-cni.yml --repo dodopizza/infra.k8s.istio-cce-sber \
  --ref "$CNI_BRANCH" -f publish=true
```

Команда выведет ссылку на новый запуск. Возьмите ID из этой ссылки:

```sh
gh run watch <RUN_ID> --repo dodopizza/infra.k8s.istio-cce-sber --exit-status
gh run view <RUN_ID> --repo dodopizza/infra.k8s.istio-cce-sber --web
```

После `success` в Summary скопируйте всю строку после
`Pinned reference for Helm values`, например:

```text
cr.yandex/crpv7tk7maoqsq3usck3/istio/install-cni@sha256:НОВЫЙ_DIGEST
```

Это ссылка на наш опубликованный образ. Digest официального runtime из шага 3
здесь не подходит: он относится к образу без нашего исправления.

### 6. Обновить Helm

Перейдите в отдельный checkout [dodopizza/helm](https://github.com/dodopizza/helm).
Файл с нашим образом добавляется в [PR #980](https://github.com/dodopizza/helm/pull/980);
пока PR не слит, используйте его ветку `fix/istio-cni-cce-hbone` как основу.
После слияния создавайте рабочую ветку от актуального `main`.

**Образ меняется здесь:**
[`charts/istio-control-plane/values-cce-sber.yaml`, поле `cni.image`](https://github.com/dodopizza/helm/blob/fix/istio-cni-cce-hbone/charts/istio-control-plane/values-cce-sber.yaml#L4).
Замените значение на всю строку из Summary шага 5:

```yaml
cni:
  image: cr.yandex/crpv7tk7maoqsq3usck3/istio/install-cni@sha256:НОВЫЙ_DIGEST
```

В [`charts/istio-control-plane/Chart.yaml`](https://github.com/dodopizza/helm/blob/fix/istio-cni-cce-hbone/charts/istio-control-plane/Chart.yaml)
обновите:

- `version` — следующая свободная версия нашего Helm-чарта; она не обязана совпадать с версией Istio.
- `appVersion` — новый upstream tag, например `1.30.0`.
- Все четыре `dependencies[].version` (`base`, `cni`, `istiod`, `ztunnel`) — тот же upstream tag.

Проверьте чарт из корня Helm-репозитория:

```sh
helm dependency update charts/istio-control-plane
helm lint charts/istio-control-plane -f charts/istio-control-plane/values-cce-sber.yaml
helm template istio-control-plane charts/istio-control-plane \
  --namespace istio-system -f charts/istio-control-plane/values-cce-sber.yaml \
  > /tmp/istio-cce-sber-rendered.yaml
```

Создайте PR с изменениями. Дождитесь успешной публикации новой версии чарта
в workflow `Release Charts OCI`.

### 7. Обновить Terraform и проверить кластер

Для каждого обновляемого кластера Sber найдите в Terraform-репозитории
описание Helm-релиза `istio-control-plane` и проверьте, откуда он получает values.
Это относится к m-sber, d-sber и другим кластерам Sber с установленным Istio.
Найти релизы и явно заданные образы можно из корня Terraform-репозитория:

```sh
rg -n 'istio-control-plane|install-cni|values-cce-sber' clouds/sber environments
```

В описании релиза замените версию чарта. Если `cni.image` задан прямо в Jsonnet,
обновите digest и там: изменение только Helm-профиля не заменит образ,
явно переданный Terraform. Если релиз загружает отдельный values-файл,
проверьте, что он получает обновлённый профиль и не переопределяет образ ниже.
Сохраните настройки `imagePullSecrets` и `reconcileIptablesOnStartup`.

Пример явной передачи values —
[`clouds/sber/services/.kube-warp-maint.libsonnet`](https://github.com/dodopizza/infra-terraform-dubai/blob/feat/m-sber-kube-warp/clouds/sber/services/.kube-warp-maint.libsonnet#L26)
для m-sber ([PR #6120](https://github.com/dodopizza/infra-terraform-dubai/pull/6120)).
Для остальных кластеров найдите соответствующее описание релиза.

Подготовьте Terraform PR, проверьте Atlantis plan и выполните apply по принятому
процессу для выбранных кластеров. В каждом обновлённом кластере проверьте
готовность CNI и ztunnel, запросы между pod
на одной и на разных нодах, несколько циклов Kuberhealthy и работу
waypoint/AuthorizationPolicy. Для отката верните предыдущую версию чарта и CNI digest
и повторите проверки. Публикация образа сама по себе кластер не обновляет.

[![CII Best Practices](https://bestpractices.coreinfrastructure.org/projects/1395/badge)](https://bestpractices.coreinfrastructure.org/projects/1395)
[![Go Report Card](https://goreportcard.com/badge/github.com/istio/istio)](https://goreportcard.com/report/github.com/istio/istio)
[![GoDoc](https://godoc.org/istio.io/istio?status.svg)](https://godoc.org/istio.io/istio)

<a href="https://istio.io/">
    <picture>
      <source media="(prefers-color-scheme: dark)" srcset="https://raw.githubusercontent.com/cncf/artwork/refs/heads/main/projects/istio/icon/color/istio-icon-color.svg">
      <source media="(prefers-color-scheme: light)" srcset="https://github.com/istio/istio/raw/master/logo/istio-bluelogo-whitebackground-unframed.svg">
      <img title="Istio" height="100" width="100" alt="Istio logo" src="https://github.com/istio/istio/raw/master/logo/istio-bluelogo-whitebackground-unframed.svg">
    </picture>
</a>

---

Istio is an open source service mesh that layers transparently onto existing distributed applications. Istio’s powerful features provide a uniform and more efficient way to secure, connect, and monitor services. Istio is the path to load balancing, service-to-service authentication, and monitoring – with few or no service code changes.

- For in-depth information about how to use Istio, visit [istio.io](https://istio.io)
- To ask questions and get assistance from our community, visit [GitHub Discussions](https://github.com/istio/istio/discussions)
- To learn how to participate in our overall community, visit [our community page](https://istio.io/about/community)

In this README:

- [Introduction](#introduction)
- [Repositories](#repositories)
- [Issue management](#issue-management)

In addition, here are some other documents you may wish to read:

- [Istio Community](https://github.com/istio/community#istio-community) - describes how to get involved and contribute to the Istio project
- [Istio Developer's Guide](https://github.com/istio/istio/wiki/Preparing-for-Development) - explains how to set up and use an Istio development environment
- [Project Conventions](https://github.com/istio/istio/wiki/Development-Conventions) - describes the conventions we use within the code base
- [Creating Fast and Lean Code](https://github.com/istio/istio/wiki/Writing-Fast-and-Lean-Code) - performance-oriented advice and guidelines for the code base

You'll find many other useful documents on our [Wiki](https://github.com/istio/istio/wiki).

## Introduction

[Istio](https://istio.io/latest/docs/concepts/what-is-istio/) is an open platform for providing a uniform way to [integrate
microservices](https://istio.io/latest/docs/examples/microservices-istio/), manage [traffic flow](https://istio.io/latest/docs/concepts/traffic-management/) across microservices, enforce policies
and aggregate telemetry data. Istio's control plane provides an abstraction
layer over the underlying cluster management platform, such as Kubernetes.

Istio is composed of these components:

- **Envoy** - Sidecar proxies per microservice to handle ingress/egress traffic
   between services in the cluster and from a service to external
   services. The proxies form a _secure microservice mesh_ providing a rich
   set of functions like discovery, rich layer-7 routing, circuit breakers,
   policy enforcement and telemetry recording/reporting
   functions.

  > Note: The service mesh is not an overlay network. It
  > simplifies and enhances how microservices in an application talk to each
  > other over the network provided by the underlying platform.

* **Ztunnel** - A lightweight data plane proxy written in Rust,
    used in Ambient mesh mode to provide secure connectivity and observability for workloads without sidecar proxies.

- **Istiod** - The Istio control plane. It provides service discovery, configuration and certificate management.

## Repositories

The Istio project is divided across a few GitHub repositories:

- [istio/api](https://github.com/istio/api). This repository defines
component-level APIs and common configuration formats for the Istio platform.

- [istio/community](https://github.com/istio/community). This repository contains
information on the Istio community, including the various documents that govern
the Istio open source project.

- [istio/istio](README.md). This is the main code repository. It hosts Istio's
core components, install artifacts, and sample programs. It includes:

    - [istioctl](istioctl/). This directory contains code for the
[_istioctl_](https://istio.io/latest/docs/reference/commands/istioctl/) command line utility.

    - [pilot](pilot/). This directory
contains platform-specific code to populate the
[abstract service model](https://istio.io/docs/concepts/traffic-management/#pilot), dynamically reconfigure the proxies
when the application topology changes, as well as translate
[routing rules](https://istio.io/latest/docs/reference/config/networking/) into proxy specific configuration.

    - [security](security/). This directory contains [security](https://istio.io/latest/docs/concepts/security/) related code.

- [istio/proxy](https://github.com/istio/proxy). The Istio proxy contains
extensions to the [Envoy proxy](https://github.com/envoyproxy/envoy) (in the form of
Envoy filters) that support authentication, authorization, and telemetry collection.

- [istio/ztunnel](https://github.com/istio/ztunnel). The repository contains the Rust implementation of the ztunnel
component of Ambient mesh.

- [istio/client-go](https://github.com/istio/client-go). This repository defines
  auto-generated Kubernetes clients for interacting with Istio resources programmatically.

> [!NOTE]
> Only the `istio/api` and `istio/client-go` repositories expose stable interfaces intended for direct usage as libraries.

## Issue management

We use GitHub to track all of our bugs and feature requests. Each issue we track has a variety of metadata:

- **Epic**. An epic represents a feature area for Istio as a whole. Epics are fairly broad in scope and are basically product-level things.
Each issue is ultimately part of an epic.

- **Milestone**. Each issue is assigned a milestone. This is 0.1, 0.2, ..., or 'Nebulous Future'. The milestone indicates when we
think the issue should get addressed.

- **Priority**. Each issue has a priority which is represented by the column in the [Prioritization](https://github.com/orgs/istio/projects/6) project. Priority can be one of
P0, P1, P2, or >P2. The priority indicates how important it is to address the issue within the milestone. P0 says that the
milestone cannot be considered achieved if the issue isn't resolved.

---

<div align="center">
    <picture>
      <source media="(prefers-color-scheme: dark)" srcset="https://raw.githubusercontent.com/cncf/artwork/refs/heads/main/other/cncf/horizontal/color-whitetext/cncf-color-whitetext.svg">
      <source media="(prefers-color-scheme: light)" srcset="https://raw.githubusercontent.com/cncf/artwork/master/other/cncf/horizontal/color/cncf-color.svg">
      <img width="300" alt="Cloud Native Computing Foundation logo" src="https://raw.githubusercontent.com/cncf/artwork/refs/heads/main/other/cncf/horizontal/color-whitetext/cncf-color-whitetext.svg">
    </picture>
    <p>Istio is a <a href="https://cncf.io">Cloud Native Computing Foundation</a> project.</p>
</div>
