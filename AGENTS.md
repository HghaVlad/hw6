# AGENTS.md — k8s-hw6

Учебная pull-доставка через Argo CD. **Минимальные HTTP-заглушки, без бизнес-логики.**

## Назначение

Продолжение ДЗ5 с теми же именами catalog, orders, notifications.
Пользователь попросил работать поэтапно, пока только с pull-моделью,
и ограничиться возвратом пустого файла с проверкой через kubectl exec.

## Структура

- base/apps — Deployment и Service трёх заглушек nginx.
- base/nginx — общий конфиг nginx и намеренно пустой index.html.
- base/kustomization.yaml — ресурсы и ConfigMapGenerator.
- overlays/dev и overlays/prod — namespace и независимый тег образа.
- argocd — по одному Application на окружение.
- README.md — процесс продвижения и команды проверки.

## Решения

- PostgreSQL, Kafka, Redis, PVC и секреты для заглушек не нужны.
- Не усложнять стенд реальными приложениями, CI-ботом или Image Updater без запроса.
- Исходную k8s-hw5 не изменять.
- Все заглушки используют один образ nginx; версии задаются в overlays.
- Сначала изменить dev, проверить, затем отдельным MR перенести точную версию в prod.
- Проверка dev перед prod организационная, автоматической блокировки в Argo CD нет.
- repoURL — заглушка; source.path рассчитан от корня репозитория education.
- ConfigMapGenerator должен сохранять хеш имени для rollout при изменении файлов.

## Проверка

- Собрать оба overlay через kubectl kustomize.
- Проверить ссылки ConfigMap, Service selector, порт 8080 и namespace.
- Не считать локальную сборку подтверждением запуска контейнеров или Argo CD.
