# AGENTS.md — k8s-hw6

Учебная pull-доставка через Argo CD. **Минимальные HTML-заглушки, без бизнес-логики.**

## Назначение

Продолжение ДЗ5 с именами catalog, orders, notifications.
Текст HTML настраивается через env из Kustomize. Пользователь требует два
Application: dev с automated.enabled=true, prod с automated.enabled=false.
Prod синхронизируется по ручной кнопке GitHub Actions без ввода SHA.

## Структура

- base/apps — Deployment, Service и HTML-шаблоны.
- base/nginx — общий конфиг nginx.
- base/kustomization.yaml — ресурсы и ConfigMapGenerator.
- overlays/dev и overlays/prod — namespace, версия образа и текст страниц.
- argocd — два Application, оба следят за main.
- .github/workflows/release-prod.yaml — workflow_dispatch без inputs.
- README.md — настройка runner и процесс выпуска.

## Решения

- Не изменять исходную k8s-hw5.
- Не возвращать продвижение ветки prod, ручной ввод SHA или automated=true для prod.
- HTML рендерит штатный entrypoint nginx из env при старте.
- Сохранять хеш ConfigMap: изменение текста должно вызывать rollout.
- Job работает на self-hosted Linux/x64 runner с меткой hw6 внутри Codespace.
- argocd --core использует Kubernetes-доступ runner; публичный API и токен Argo CD не нужны.
- Вся логика Action находится в YAML: выбор context/namespace, Sync и ожидание готовности.
- Не добавлять отдельные shell-скрипты, checkout, временный kubeconfig и служебные отчёты без запроса.
- Workflow меняет текущий context/namespace в kubeconfig пользователя runner.
- Workflow синхронизирует overlays/prod из актуального main, не копирует настройки dev.
- Проверка dev перед нажатием кнопки ручная; workflow ждёт готовности prod.
- Изменение Application требует отдельного kubectl apply; App of Apps не используется.

## Проверка

- Собрать оба overlay через kubectl kustomize.
- Проверить YAML workflow, bash-синтаксис, обработку ошибок CLI и отключённый automated prod.
- Не выполнять Sync в реальном кластере ради локальной проверки кода.
- Не считать локальную проверку YAML и bash-синтаксиса реальным запуском GitHub Actions.
