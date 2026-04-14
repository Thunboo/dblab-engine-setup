# Документация по использованию файлов проекта DBLab

Этот проект содержит конфигурации и Docker файлы для настройки и запуска DBLab Engine для создания Thin-clones с PostgreSQL для 1C.

## 1. Подготовка системы

Согласно гайду, необходимо создать ZFS pool для хранения данных. Подробная инструкция: [Ссылка на документацию по созданию ZFS pool](https://postgres.ai/docs/dblab-howtos/administration/install-dle-manually)

## 2. Сборка контейнеров из образов

> (!) Для сборки нужны .rpm пакеты сборки postgresql 1c

Проект содержит два Docker образа для PostgreSQL с поддержкой 1C:

### Сборка образа pg-1c-ru-centos-rpms:v2
```
cd /path/to/folder/custom_images/postgres-1c-17-rpm/
sudo docker build -t pg-1c-ru-centos-rpms:v2 -f custom_images/postgres-1c-17-rpm/Dockerfile.1c-rpm-soft-links .
```

### Сборка образа pg-dblab-ru:v2
```
cd /path/to/folder/custom_images/postgres-1c-17-rpm/
sudo docker build -t pg-dblab-ru:v2 -f custom_images/postgres-1c-17-rpm/Dockerfile.dblab-ru .
```

Эти команды собирают образы, необходимые для работы DBLab Engine.

## 3. Конфигурация DBLab Engine

### Размещение конфигураций
Конфигурационные файлы для DBLab Engine размещаются в домашней директории пользователя:
```
~/.dblab/engine/configs/
```

В этой папке должен находиться файл `server.yml`, который читается DBLab сервером.

### Управление конфигурациями
Для удобного переключения между разными конфигурациями используйте бэкапы в подпапке `config-backups/`:
- `logical-backup.server.yml` - конфигурация для логического бэкапа
- `physical-backup.server.yml` - конфигурация для физического бэкапа

Рекомендуется положить эту папку сразу в папку `~/.dblab/engine/configs/` для удобства.

Создайте символическую ссылку на нужную конфигурацию:
```
ln -sf ~/.dblab/engine/configs/config-backups/CONFIG_NAME ~/.dblab/engine/configs/server.yml
```
Где `CONFIG_NAME` - имя файла конфигурации (например, `logical-backup.server.yml`).

## 4. Запуск DBLab Server

Для запуска DBLab сервера используйте следующую команду Docker:

```
sudo docker run \
  --name dblab_server \
  --label dblab_control \
  --privileged \
  --publish 0.0.0.0:2345:2345 \
  --volume /var/run/docker.sock:/var/run/docker.sock \
  --volume /var/lib/dblab:/var/lib/dblab/:rshared \
  --volume ~/.dblab/engine/configs:/home/dblab/configs \
  --volume ~/.dblab/engine/meta:/home/dblab/meta \
  --volume ~/.dblab/engine/logs:/home/dblab/logs \
  --volume /sys/kernel/debug:/sys/kernel/debug:rw \
  --volume /lib/modules:/lib/modules:ro \
  --volume /proc:/host_proc:ro \
  --env DOCKER_API_VERSION=1.40 \
  --detach \
  --restart on-failure \
  postgresai/dblab-server:4.1.0
```

### Параметры запуска:
В целом все и так видно. Команда взята из [документации](https://postgres.ai/docs/dblab-howtos/administration/install-dle-manually). Но отметим один момент:
- `--publish 0.0.0.0:2345:2345`: Публикация порта 2345 для доступа к DBLab с любого IP. Можно поставить 127.0.0.1 (как в документации) или любой другой IP доступный на сетевых интерфейсах вашего сервера.

После запуска сервер будет доступен на порту 2345.

## 5. Настройка Nginx

### Размещение конфигурации Nginx
Конфигурационный файл Nginx находится в папке `nginx/sites-available/dblab`. Этот файл содержит настройки для сайта DBLab. Его необходимо поместить в `/etc/nginx/sites-available/`.

### Активация конфигурации
Чтобы активировать конфигурацию:
1. Создайте символическую ссылку из `sites-available` в `sites-enabled`:
   ```
   sudo ln -s /etc/nginx/sites-available/dblab /etc/nginx/sites-enabled/
   ```
   (Предполагается, что файлы скопированы в `/etc/nginx/sites-available/`)

2. Проверьте конфигурацию на ошибки:
   ```
   sudo nginx -t
   ```

3. Перезапустите Nginx:
   ```
   sudo systemctl reload nginx
   ```

### Настройка Firewall
Для доступа к Nginx (обычно на портах 80 и 443) настройте firewall:
- Для UFW (Ubuntu):
  ```
  sudo ufw allow 80
  sudo ufw allow 443
  ```
- Для firewalld (CentOS/RHEL):
  ```
  sudo firewall-cmd --permanent --add-port=80/tcp
  sudo firewall-cmd --permanent --add-port=443/tcp
  sudo firewall-cmd --reload
  ```
- Для iptables:
  ```
  sudo iptables -A INPUT -p tcp --dport 80 -j ACCEPT
  sudo iptables -A INPUT -p tcp --dport 443 -j ACCEPT
  ```