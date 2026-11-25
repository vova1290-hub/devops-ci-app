# Подготовка публикации на учебной VM

Образ: `ghcr.io/vova1290-hub/devops-ci-app`.
Итоговое задание image использует runner `ubuntu-24.04-arm`, поскольку учебная VM работает на ARM64.

## Выбор версии

После успешного CI на main запишите полный SHA соответствующего merge-коммита и digest из шага Push image.
Для развёртывания используйте тег полного SHA или digest. latest — изменяемый указатель для ручной проверки.

## Проверка вне CI

```sh
docker pull ghcr.io/vova1290-hub/devops-ci-app:latest
docker run -d --name ci-app -p 127.0.0.1:5001:5000 ghcr.io/vova1290-hub/devops-ci-app:latest
curl --fail http://127.0.0.1:5001/health
docker exec ci-app id
docker logs ci-app
docker rm -f ci-app
```

Повторите запуск по полному SHA последнего зелёного коммита на 127.0.0.1:5002:5000.
Пакет должен быть публичным для анонимного pull. Секреты и адреса внутренней инфраструктуры в документацию не включаются.

Для будущего Ansible-деплоя в отдельном приватном inventory укажите адрес узла и SSH-пользователя; в параметрах роли — полный адрес образа с SHA/digest, архитектуру, публикацию порта и путь /health. Повторная сборка на VM не требуется.

## 10. Ansible

Практическая работа 11 выполнена 25.11.2026. После возврата к `01-clean-install` действующее состояние задаёт `ansible/`; SSH теперь на порту 22, HTTPS в этой работе не настраивается. Перед возвратом сохранён снимок `08-before-pr11` с результатами прежних работ.

Управляющий узел — macOS ARM64 с ansible-core 2.17.14; управляемый — существующая Ubuntu Server 24.04 ARM64, Host-only `<VM_HOST>`. NAT используется VM для загрузки пакетов. Подключение Ansible идёт напрямую через Host-only на порт 22, а не через старый псевдоним `devops`.

Inventory получает адрес из переменной окружения `DEVOPS_VM_HOST`, путь локального ключа — из `DEVOPS_SSH_KEY`. Эти значения задаются на управляющем узле и в Git не сохраняются.

Проект расположен в `ansible/`. Запускать команды из этого каталога. Коллекции фиксируются в `requirements.yml`: ansible.posix 1.6.2, community.docker 4.5.0, community.general 9.5.0.

```sh
cd ansible
ansible-galaxy collection install -r requirements.yml
ansible-inventory -i inventory/bootstrap.ini --graph
ansible-inventory --graph
ansible-playbook -i inventory/bootstrap.ini site.yml --syntax-check
ansible-playbook site.yml --syntax-check
```

`inventory/bootstrap.ini` подключается от student. Использован разрешённый методичкой вариант bootstrap по ключу: открытый ключ установлен student через первичную парольную SSH-сессию; затем sudo-пароль требуется только при первичном прогоне. `inventory/prod.ini` использует devops и `~/.ssh/devops_vm`. Закрытый ключ хранится только на управляющем узле, в Git его нет. Ключ хоста сверяется с ранее доверенным ключом VM; `host_key_checking = True` сохранён.

```sh
ansible-playbook -i inventory/bootstrap.ini site.yml --ask-become-pass
ansible all -m ping
ansible-playbook site.yml
ansible-playbook verify.yml
```

Роли выполняются в порядке base → ssh → nginx → app:

- base проверяет Ubuntu 24.04 и архитектуру, отключает устаревший CD-ROM источник APT чистого снимка, устанавливает зависимости и Docker Engine из официального deb822-репозитория; aarch64 соответствует arm64, x86_64 — amd64;
- ssh создаёт devops, устанавливает открытый ключ, валидирует sudoers, отключает парольный/root SSH после установки ключа, разрешает 22 и 80 в UFW;
- nginx управляет reverse proxy и symlink, удаляет default site, проверяет `nginx -t` перед reload;
- app сначала загружает проверенный образ, затем управляет контейнером и ожидает `GET /health` с HTTP 200 и JSON `status=ok`.

Образ — `ghcr.io/vova1290-hub/devops-ci-app`.
Версия в `group_vars/all.yml` — `7ac2890c45ac483888cc830789670af2d6efb04d`, зелёный коммит ПР10; архитектура linux/arm64.
Контейнер — `devops-ci-app`, публикация `127.0.0.1:5000:5000`, restart policy `unless-stopped`.
Digest — `sha256:4521a30d58d95cf510a0f0ed3861fea8486be6adcc1e446644590b4be313ec16`.
Приложение на VM не пересобирается. Внешний health endpoint — `http://<VM_HOST>/health`, запрос идёт через Nginx.

`force_handlers = True` обеспечивает выполнение уже уведомлённых handlers даже при позднем отказе. При повторном прогоне без изменений ожидается `changed=0`, `failed=0`. Read-only `verify.yml` проверяет эффективные SSH-параметры, существование и состояние контейнера и JSON health, не исправляя конфигурацию. `command: sshd -T` имеет `changed_when: false`.

Проверки инцидентов: удаление контейнера и замена proxy upstream на 5001 дают 502; `site.yml` восстанавливает оба расхождения. Несуществующий тег из 40 нулей приводит к отказу pull до задачи изменения контейнера, прежний сервис остаётся доступен. После возврата SHA повторный запуск идемпотентен. Остановленный контейнер обнаруживается аудитом; восстанавливается основным плейбуком.

Для следующей контрольной нужны отдельный ключ деплоя, доверенный host key и защищённое хранение SSH-настроек. GitHub-hosted runner не видит Host-only сеть: потребуется защищённый туннель либо подходящий self-hosted runner с ограниченным доступом в приватном репозитории.
