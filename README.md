# Ansible: раскатка ash-script на серверах БД

Playbook разворачивает `ash-script.sh` на серверах PostgreSQL по контурам (DEV, AT, TEST, PROD): скачивает скрипт из GitLab, выставляет права и владельца, при необходимости запускает скрипт и добавляет задание в crontab пользователя `postgres`.

## Структура

```
ansible-playbook/
├── ansible.cfg
├── inventory/hosts.yml       # контур → хост сервера БД
├── group_vars/
│   ├── all.yml               # пути, cron, токен по умолчанию
│   ├── DEV.yml / AT.yml / TEST.yml / PROD.yml  # URL и токен GitLab
└── playbooks/deploy_ash_script.yml
```

## Требования

- Ansible 2.10+
- SSH-доступ к серверам БД с правами `sudo`
- Пользователь `postgres` на целевых хостах
- GitLab Personal/Project Access Token с правом чтения репозитория

## Настройка

### 0: Задайте путь до ansible.cfg 

```bash
# либо передав переменную перед запуском команды
ANSIBLE_CONFIG=ansible.cfg ansible-play...
# либо 
export ANSIBLE_CONFIG=/path/to/ansible.cfg
```


### 1. Инвентарь — `inventory/hosts.yml`

Укажите реальные хосты для каждого контура:

```yaml
DEV:
  hosts:
    dev-db.example.com:
      ansible_host: 10.0.1.10
```

### 2. URL скрипта — `group_vars/<КОНТУР>.yml`

Для каждого контура задайте URL raw-файла через GitLab API:

```yaml
ash_script_gitlab_url: "https://gitlab.example.com/api/v4/projects/123/repository/files/ash-script%2Fash-script.sh/raw?ref=main"
ash_script_gitlab_token: "{{ gitlab_token_default }}"
```

Для PROD можно указать отдельный GitLab и токен (`gitlab_token_prod`).

### 3. Токен GitLab

Один из вариантов:

```bash
# при запуске
-e gitlab_token_default=glpat-xxx

# или ansible-vault в group_vars/all/vault.yml
vault_gitlab_token_default: glpat-xxx
```

## Запуск

```bash
# несколько контуров (JSON)
ANSIBLE_CONFIG=ansible.cfg ansible-playbook playbooks/deploy_ash_script.yml -e 'contours=["DEV","TEST"]'

# через запятую
ansible-playbook playbooks/deploy_ash_script.yml -e contours=DEV,AT

# с токеном
ansible-playbook playbooks/deploy_ash_script.yml \
  -e contours=PROD \
  -e gitlab_token_prod=glpat-prod-xxx

# проверка без изменений
ansible-playbook playbooks/deploy_ash_script.yml -e contours=DEV --check
```

Параметр `contours` обязателен. Допустимые значения: `DEV`, `AT`, `TEST`, `PROD`.

## Логика работы

На каждом хосте playbook проверяет crontab пользователя `postgres` на наличие записи:

```
* * * * * /var/lib/pgsql/ash-script/ash-script.sh
```

| Состояние | Действие |
|-----------|----------|
| Запись в crontab уже есть | Хост пропускается, изменений нет |
| Скрипта нет | Скачивание из GitLab → `chmod 0750` → владелец `postgres:postgres` |
| Скрипт есть, cron нет | Скрипт не перекачивается → запуск под `postgres` → добавление cron |
| Скрипт и cron на месте | Повторный запуск ничего не меняет |

Порядок при первичной установке: скачать скрипт → сделать исполняемым → запустить один раз от `postgres` → добавить cron.

## Переменные

| Переменная | Файл | Описание |
|------------|------|----------|
| `ash_script_path` | `all.yml` | Путь к скрипту на сервере |
| `ash_script_gitlab_url` | `<КОНТУР>.yml` | URL файла в GitLab API |
| `ash_script_gitlab_token` | `<КОНТУР>.yml` | Токен для заголовка `PRIVATE-TOKEN` |
| `gitlab_token_default` | `all.yml` / `-e` | Общий токен для DEV/AT/TEST |
| `gitlab_token_prod` | `-e` | Отдельный токен для PROD |
