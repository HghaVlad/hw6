# ДЗ6: dev автоматически, prod по кнопке GitHub Actions

Три сервиса `catalog`, `orders`, `notifications` отдают HTML-страницы через nginx.
Для каждого есть Deployment и Service на порту 8080.

| Application | Ветка | Путь | Автосинхронизация |
| --- | --- | --- | --- |
| hw6-orders-dev | main | overlays/dev | enabled: true |
| hw6-orders-prod | main | overlays/prod | enabled: false |

После push в main Argo CD автоматически обновляет dev. Prod обнаруживает
изменения и может показывать OutOfSync, но не применяет их до команды Sync.
Кнопка **Actions → Release prod → Run workflow** запускает эту команду.
Поля для ввода версии или SHA нет; Argo CD синхронизирует актуальный main.

## Что делает workflow

1. Ручной `workflow_dispatch` запускает job на runner с меткой `hw6` внутри Codespace.
2. Выбирает context `kind-hw6` и namespace `argocd` в kubeconfig пользователя runner.
3. Вызывает `argocd app sync` для prod и ждёт Synced/Healthy через `argocd app wait`.

Все четыре команды находятся прямо в YAML. Checkout репозитория и отдельные
скрипты не нужны: манифесты получает и применяет контроллер Argo CD.
`--core` использует Kubernetes-доступ runner без публичного API и пароля Argo CD.
Выбранный context и namespace сохраняются в kubeconfig пользователя runner.

Workflow не копирует настройки dev в prod: применяется именно `overlays/prod`.
Если main изменится, пока job стоит в очереди, будет синхронизирован актуальный
main на момент Sync. Проверка нужных изменений на dev выполняется вручную.

## Однократная настройка

Отправить файлы в main. В Codespace, из корня репозитория:

```bash
git pull --ff-only
kubectl --context kind-hw6 apply -f argocd/
```

Это обновляет политики существующих Application. Имена должны совпадать с
файлами: `hw6-orders-dev` и `hw6-orders-prod`. Сами Application не входят
в отслеживаемые overlays, поэтому их настройки требуют этого отдельного apply.
Доступ Argo CD к приватному репозиторию настраивается отдельно и только на чтение.

### Runner внутри Codespace

GitHub-hosted runner не имеет прямого доступа к нашему кластеру kind.
Поэтому job запускается на self-hosted runner в том же Codespace.

1. В GitHub открыть **Settings → Actions → Runners → New self-hosted runner**.
2. Выбрать **Linux / x64** и выполнить предложенные GitHub команды в отдельной
   папке Codespace, например `~/actions-runner`, под тем же пользователем,
   у которого работают kubectl и kind.
3. При настройке дополнительных labels указать **hw6**.
4. Установить Argo CD CLI, желательно той же версии, что и сервер.
   Проверить: `argocd version --client` и `kubectl --context kind-hw6 get nodes`.
5. В папке runner запустить `./run.sh` и оставить терминал работающим.
   В GitHub runner должен отображаться как Idle.

[Команды установки CLI](https://argo-cd.readthedocs.io/en/stable/cli_installation/),
[добавление runner](https://docs.github.com/en/actions/how-tos/manage-runners/self-hosted-runners/add-runners).

Пароль Argo CD и дополнительный PAT в Actions Secrets не нужны.
Workflow не использует GITHUB_TOKEN для доступа к репозиторию.
Runner должен иметь доступ к Kubernetes context `kind-hw6` и ресурсам Argo CD.
Если Codespace выключен или runner не запущен, workflow будет ждать runner;
prod сам не обновится.

## Выпуск

1. Отправить изменения в main и проверить dev.
2. Открыть **Actions → Release prod → Run workflow**, оставить ветку **main**.
3. Нажать Run workflow без дополнительных параметров.
4. Проверить успешное завершение job и состояние prod в Argo CD.

Ошибки Sync и ожидания готовности приводят к ошибке workflow. Зелёный результат
означает, что Argo CD сообщил Synced/Healthy; бизнес-проверки нужно делать отдельно.
Несколько релизных job не выполняются одновременно благодаря concurrency.

## HTML и настройки окружений

- `base/apps/*.yaml` — Deployment и Service.
- `base/apps/*.html.template` — HTML-шаблоны трёх страниц.
- `base/nginx/default.conf` — HTTP на 8080 и endpoint `/ready`.
- `overlays/dev` и `overlays/prod` — namespace, версия образа и текст страницы.
- `argocd/*-application.yaml` — два Application.
- `.github/workflows/release-prod.yaml` — ручной Action.

В `configMapGenerator` каждого overlay меняются `PAGE_TITLE`, `PAGE_MESSAGE`
и `APP_ENV`. При запуске nginx подставляет env в шаблон и создаёт index.html.
Хеш ConfigMap вызывает rollout при изменении текста. Значения вставляются
как HTML без автоматического экранирования.

## Проверки

```bash
kubectl kustomize overlays/dev
kubectl kustomize overlays/prod
kubectl -n argocd get applications
kubectl -n hw6-orders-dev exec deployment/catalog -- wget -qO- http://catalog:8080/
kubectl -n hw6-orders-prod exec deployment/catalog -- wget -qO- http://catalog:8080/
```

Документация: [workflow_dispatch](https://docs.github.com/en/actions/how-tos/manage-workflow-runs/manually-run-a-workflow),
[команда Sync](https://argo-cd.readthedocs.io/en/stable/user-guide/commands/argocd_app_sync/),
[автоматическая синхронизация](https://argo-cd.readthedocs.io/en/stable/user-guide/auto_sync/).
