# Мониторинг стенда в Zabbix

Сервер Zabbix 7.0, веб-интерфейс и PostgreSQL работают в Docker Compose на узле `mon` (192.168.56.30). Агенты на lb1, lb2, web1, web2 ставит роль Ansible `zabbix_agent`.

## 1. Сервер

На `mon`:

```bash
cd ~/zabbix
cp .env.example .env
nano .env                 # задать свой пароль
docker compose up -d
docker compose ps         # все три сервиса в состоянии running
docker compose logs -f zabbix-server
```

Первый запуск занимает 1–2 минуты: сервер создаёт схему базы.

Веб-интерфейс: `http://192.168.56.30:8080`, логин `Admin`, пароль по умолчанию `zabbix`. Сразу смените его: *User settings → Profile → Change password*.

> Пароль базы хранится в `.env`, который не коммитится. В репозитории лежит только `.env.example`.

## 2. Агенты

На управляющем узле (lb1) из каталога `ansible/`:

```bash
ansible-playbook site.yml -K
```

Проверка своих метрик на балансировщике. Запуск от пользователя `zabbix` нужен, чтобы сразу увидеть возможные проблемы с правами:

```bash
sudo -u zabbix /usr/sbin/zabbix_agentd -t stand.vip.present
sudo -u zabbix /usr/sbin/zabbix_agentd -t stand.haproxy.active
```

Доступность агента с сервера (на `mon`):

```bash
nc -zv 192.168.56.21 10050
```

> Сервер работает в контейнере, но агенты видят его подключения с адреса `mon` (192.168.56.30): исходящий трафик контейнеров проходит через NAT хоста. Поэтому в конфиге агента `Server=192.168.56.30`.

## 3. Импорт шаблона

*Data collection → Templates → Import* → файл `zabbix/templates/stand_ha_lb.yaml`.

Шаблон **Stand HA LB** (группа *Templates/Stand*) содержит:

| Метрика | Ключ | Триггер |
|---|---|---|
| HAProxy: служба активна | `stand.haproxy.active` | HAProxy не запущен на узле (High) |
| Keepalived: VIP на этом узле | `stand.vip.present` | — (триггер на два хоста создаётся отдельно, шаг 5) |

## 4. Хосты

*Data collection → Host groups → Create host group*: `Stand`.

Для каждого узла: *Data collection → Hosts → Create host*.

| Поле | Значение |
|---|---|
| Host name | `lb1` / `lb2` / `web1` / `web2`, точно как `Hostname=` в конфиге агента |
| Host groups | `Stand` |
| Interfaces | Agent, IP узла, порт 10050 |
| Templates | `Linux by Zabbix agent` у всех; плюс `Stand HA LB` у lb1 и lb2 |

> `Host name` должен совпадать с `Hostname` агента. Иначе пассивные проверки работают, а активные — нет: в логе агента будет сообщение, что хост не найден.

Через 1–2 минуты в *Data collection → Hosts* значок **ZBX** у всех четырёх должен стать зелёным, а в *Monitoring → Latest data* появятся значения.

## 5. Триггер split-brain

VIP должен быть ровно на одном балансировщике. Триггер сравнивает метрики двух хостов, поэтому в шаблон его не положить: он создаётся на хосте.

*Data collection → Hosts → lb1 → Triggers → Create trigger*:

| Поле | Значение |
|---|---|
| Name | `VIP: split-brain или VIP отсутствует` |
| Severity | Disaster |
| Expression | `last(/lb1/stand.vip.present)+last(/lb2/stand.vip.present)<>1` |

Сумма 2 означает split-brain (VIP на обоих), сумма 0 — VIP нет нигде, сервис недоступен.

## 6. Проверка алертов

Смотреть в *Monitoring → Problems*. После каждого сценария вернуть всё как было и убедиться, что проблема закрылась.

| Сценарий | Как сломать | Ожидаемый триггер |
|---|---|---|
| HAProxy упал | на lb1: `sudo systemctl stop haproxy` | HAProxy не запущен на lb1; VIP уходит на lb2, сумма остаётся 1 |
| Split-brain | на lb2: блокировка VRRP через nftables (см. ниже) | VIP: split-brain |
| Агент недоступен | на web2: `sudo systemctl stop zabbix-agent` | Zabbix agent is not available (через несколько минут) |
| Кончается место | на web1: `df -h /`, затем `sudo fallocate -l <размер> /var/tmp/big`, чтобы занять больше 90 % | Space is critically low |

Блокировка и снятие блокировки VRRP на lb2:

```bash
sudo nft add table inet test
sudo nft add chain inet test input '{ type filter hook input priority 0; }'
sudo nft add rule inet test input ip protocol vrrp drop
# ...проверка...
sudo nft delete table inet test
```

Не забыть удалить тестовый файл: `sudo rm /var/tmp/big`.

## Что можно добавить

- Уведомления в Telegram: встроенный тип оповещения *Telegram* и действие (*Alerts → Actions*) на проблемы группы `Stand`
- Автоматическая регистрация агентов (`HostMetadata` + действие *Autoregistration*) вместо ручного создания хостов
- Метрики HAProxy через stats-сокет: состояние бэкендов, число сессий
