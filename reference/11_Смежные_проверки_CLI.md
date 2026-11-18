# Команды для смежных требований ДКБ-С

Каждый маршрут проверяют отдельно. Записать источник, назначение и ожидаемое решение до испытания. Команды с `read` предназначены для Bash. Учетные данные берутся из подготовленных профилей с правами только на испытание. Значения ключей, пароли и токены в протокол не выводить.

<a id="cli-tls"></a>

## T1. С-01/02 и К1. TLS, доверенный УЦ, отзыв, права

**1. На разрешённом клиенте** выбрать HTTPS-операцию чтения. Для mTLS нужен вход, где обязательна проверка клиентского сертификата. Подготовить выделенный действующий сертификат, его цепочку, ключ, сертификат чужого УЦ и ключ к нему. Остальные условия запроса сохранять одинаковыми. Сертификаты клиента в PEM при необходимости содержат промежуточный УЦ. Ввести полный URL с DNS-именем сервера, чтобы проверить имя и SNI.

```bash
read -rp 'HTTPS URL проверяемого чтения: ' URL
read -rp 'Файл доверенного CA сервера: ' CA
read -rp 'Цепочка разрешённого клиента: ' CERT
read -rp 'Ключ разрешённого клиента: ' KEY
read -rp 'Цепочка тестового клиента постороннего УЦ: ' OTHER
read -rp 'Его ключ: ' OTHER_KEY
openssl x509 -in "$CERT" -noout -subject -issuer -serial -dates
openssl x509 -in "$OTHER" -noout -subject -issuer -serial
```

**2. На том же клиенте** выполнить три новых запроса. Если выбранная операция требует прикладного токена, одинаковые параметры передать через защищённый локальный `curl --config` во всех трёх вызовах; содержимое файла в протокол не включать. Успех — ожидаемый ответ авторизованного чтения. Для двух отрицательных случаев зафиксировать TLS alert или документированное отклонение клиентского сертификата на TLS-терминаторе и запись этого узла. Один HTTP 401 без установления причины недостаточен.

```bash
mtls() { curl -sS --connect-timeout 4 --max-time 12 --cacert "$CA" -o /dev/null -w 'HTTP %{http_code}\n' "$@" "$URL"; }
mtls --cert "$CERT" --key "$KEY"
mtls
mtls --cert "$OTHER" --key "$OTHER_KEY"
```

**3. Отзыв.** Повторить успешную пробу с выделенным одноразовым сертификатом и зафиксировать его серийный номер. На машине оператора испытательного Vault PKI выбрать mount нужного издателя и передать серийный номер из его учётной записи (формат Vault). Отзыв необратим для данного сертификата. Здесь используется только сертификат, выпущенный специально для этого опыта. Для иного УЦ использовать уже зарегистрированный тестовый отзыв с временем и номером, либо его собственную процедуру обслуживания.

```bash
date -u
openssl x509 -in "$CERT" -noout -serial -issuer
mtls --cert "$CERT" --key "$KEY"
```

```bash
read -rp 'Mount испытательного Vault PKI: ' PKI
read -rp 'Серийный номер одноразового сертификата в Vault: ' SERIAL
vault write "$PKI/revoke" serial_number="$SERIAL"
```

**4. На клиенте** после штатного обновления CRL/OCSP на принимающем сервисе повторить тот же запрос и отдельный контроль с ещё действующим клиентом. На принимающем узле выбрать сервисный журнал TLS-терминатора за этот интервал, записать причину и время отказа. Критерий — отозванный отклонён, второй действующий принят, посторонняя цепочка также отклонена. Проверка CRL только командой openssl на рабочем месте не подтверждает применение CRL сервисом.

```bash
date -u
mtls --cert "$CERT" --key "$KEY"
read -rp 'Второй действующий клиент, цепочка: ' CONTROL_CERT
read -rp 'Его ключ: ' CONTROL_KEY
mtls --cert "$CONTROL_CERT" --key "$CONTROL_KEY"
```

```bash
read -rp 'Файл журнала принимающего сервиса: ' TLS_LOG
sudo tail -n 150 "$TLS_LOG" | grep -Ei 'certificate|revoked|unknown ca|verify|handshake'
```

**5. Авторизация отдельно. На А** выбрать два профиля с действующими удостоверениями и разными полномочиями. Для примера ниже в RBAC стенда одна роль может просматривать сервисы Nova, другая должна получить HTTP 403. Если в стенде иная матрица ролей, выбрать заранее утверждённое чтение с такой парой решений и записать фактическую команду. Проверка токена выводит только project ID.

```bash
read -rp 'Профиль с разрешённым чтением: ' ALLOW
read -rp 'Профиль с ограниченной ролью: ' DENY
openstack --os-cloud "$ALLOW" token issue -f value -c project_id
openstack --os-cloud "$DENY" token issue -f value -c project_id
openstack --os-cloud "$ALLOW" compute service list
openstack --os-cloud "$DENY" compute service list
```

## T2. С-01. Брокер и оба участка БД

**1. На принимающем узле RabbitMQ** выбрать фактический контейнер Kolla и получить сведения о служебных соединениях. Для Docker пример ниже; при Podman заменить имя клиента контейнеров. Успех — нужный сервисный пользователь, TLS, согласованная версия и субъект сертификата именно этой сессии.

```bash
sudo docker ps --format '{{.Names}}'
read -rp 'Контейнер RabbitMQ: ' RABBIT
sudo docker exec "$RABBIT" rabbitmqctl list_connections user peer_host peer_port ssl ssl_protocol peer_cert_subject
```

**2. На разрешённом AMQP-клиенте** выполнить одну и ту же пробу c действующим сертификатом и без него. Команда передаёт только стандартный заголовок AMQP 0-9-1. Ответ Connection.Start подтверждает, что сервер принял TLS и перешёл к AMQP. Для обязательного mTLS у второй попытки ожидается ошибка клиентского сертификата и отсутствует AMQP-ответ. На брокере подтвердить причину. Сопоставить разрешённые прикладные права со штатным сервисным соединением.

```bash
read -rp 'DNS-имя брокера: ' MQ_NAME
read -rp 'AMQPS-порт: ' MQ_TLS
read -rp 'CA брокера: ' CA
read -rp 'Действующий сертификат клиента брокера: ' CERT
read -rp 'Ключ этого клиента: ' KEY
amqp_probe() {
  printf 'AMQP\000\000\011\001' | timeout 6 openssl s_client -quiet -connect "$MQ_NAME:$MQ_TLS" -servername "$MQ_NAME" -verify_hostname "$MQ_NAME" -verify_return_error -CAfile "$CA" "$@" | od -An -tx1 -N 16
}
amqp_probe -cert "$CERT" -key "$KEY"
amqp_probe
```

**3. На разрешённом клиенте БД** подготовить два файла настроек MariaDB клиента с одним и тем же пользователем, паролем и доверенным CA. Первый содержит клиентские `ssl-cert` и `ssl-key`, второй их не содержит. Файлы с правами 0600. Указать FQDN и порт реального входа ProxySQL, затем повторить для FQDN/порта принимающей MariaDB **из разрешённой точки на стороне ProxySQL** с соответствующей тестовой учётной записью. Нужен отдельный положительный и отрицательный контроль на каждом участке.

```bash
read -rp 'Файл клиента БД с сертификатом: ' WITH_CERT
read -rp 'Файл того же клиента без сертификата: ' WITHOUT_CERT
read -rp 'DNS-имя принимающей стороны: ' SQL_HOST
read -rp 'Порт этой стороны: ' SQL_PORT
mariadb --defaults-file="$WITH_CERT" --protocol=tcp --host="$SQL_HOST" --port="$SQL_PORT" --ssl-verify-server-cert -e 'SELECT 1'
mariadb --defaults-file="$WITHOUT_CERT" --protocol=tcp --host="$SQL_HOST" --port="$SQL_PORT" --ssl-verify-server-cert -e 'SELECT 1'
```

**4. На клиенте ProxySQL** запросить состояние текущего клиентского TLS через статус команды клиента. **На стороне MariaDB** с теми же параметрами прямого тестового подключения запросить cipher. Для фактической связи ProxySQL → MariaDB дополнительно на админ-входе ProxySQL проверить соединение с нужным backend и на стороне MariaDB увидеть сессию с IP ProxySQL. Если сервер поддерживает таблицу `performance_schema.status_by_thread`, последняя команда на MariaDB связывает TLS cipher именно с этой сессией. Если таблицы нет, потребовать эквивалентное свидетельство от принимающего сервера. Одна успешная ручная сессия к backend не доказывает свойства действующего пула ProxySQL.

```bash
mariadb --defaults-file="$WITH_CERT" --protocol=tcp --host="$SQL_HOST" --port="$SQL_PORT" --ssl-verify-server-cert -e '\s'
mariadb --defaults-file="$WITH_CERT" --protocol=tcp --host="$SQL_HOST" --port="$SQL_PORT" --ssl-verify-server-cert -e "SHOW SESSION STATUS LIKE 'Ssl_cipher'"
```

```bash
read -rp 'Защищённый клиентский файл для admin ProxySQL: ' PROXY_ADMIN
mariadb --defaults-file="$PROXY_ADMIN" -e 'SELECT hostgroup,srv_host,srv_port,ConnUsed,ConnFree FROM stats_mysql_connection_pool'
read -rp 'Защищённый файл наблюдателя MariaDB: ' DB_AUDIT
mariadb --defaults-file="$DB_AUDIT" -e "SELECT t.PROCESSLIST_USER,t.PROCESSLIST_HOST,s.VARIABLE_VALUE AS cipher FROM performance_schema.threads t JOIN performance_schema.status_by_thread s USING (THREAD_ID) WHERE s.VARIABLE_NAME='Ssl_cipher' AND t.PROCESSLIST_HOST IS NOT NULL"
```

## T3. К6 и К9. Доставка почты и канал СХД

**1. К6, испытательный отправитель.** Использовать подготовленное правило Grafana, отслеживающее доступность выделенного HTTP-пробника. Его маршрут должен вести в тестовый email contact point Grafana. На согласованном хосте запустить временный HTTP на выделенном адресе. В точке мониторинга проверить HTTP 200. Дождаться нормального состояния правила.

```bash
read -rp 'IP тестового пробника: ' PROBE_IP
python3 -m http.server 28083 --bind "$PROBE_IP" --directory "$(mktemp -d)"
```

```bash
read -rp 'URL этого пробника из сети мониторинга: ' PROBE_URL
curl -sS --max-time 4 -o /dev/null -w '%{http_code}\n' "$PROBE_URL"
date -u
```

**2. На хосте пробника** остановить этот процесс Ctrl+C. Выждать период проверки, for и group_wait настроенного правила. На узле Grafana найти запись отправки. На тестовом почтовом шлюзе найти ту же доставку в журнале. Ниже пример для Postfix с журналом в journald. Сопоставить по времени, соединению/IP Grafana, queue ID, получателю и ID письма на тестовом ящике. В журнале SMTP должно быть подтверждение TLS именно этой сессии. Если политика требует IMAPS для получения, выполнить последнюю команду к реальному IMAP входу и подтвердить получение того же письма на ящике.

```bash
read -rp 'Файл журнала Grafana: ' GRAFANA_LOG
sudo tail -n 200 "$GRAFANA_LOG" | grep -Ei 'notify|email|smtp|alert'
```

```bash
sudo journalctl --since '20 min ago' --no-pager -t postfix/smtpd -t postfix/cleanup -t postfix/qmgr -t postfix/local
```

```bash
read -rp 'IMAPS URL тестового ящика, например imaps://mail.example/INBOX: ' IMAP_URL
read -rp 'Файл netrc тестового IMAP: ' IMAP_AUTH
read -rp 'CA IMAP-сервера: ' MAIL_CA
read -rp 'Message-ID тестового письма: ' MSG
curl -sS --netrc-file "$IMAP_AUTH" --cacert "$MAIL_CA" "$IMAP_URL" -X "UID SEARCH HEADER Message-ID \"$MSG\""
```

Запустить пробник снова той же командой и проверить HTTP 200. Критерий К6 требует фактической защищённой доставки сообщения от Grafana. Изолированное TLS-подключение оператора к SMTP проверяет только доступность сервера.

**3. К9, compute.** Выбрать выделенный диск тестовой ВМ с известным Cinder volume ID и путём до реальной СХД. На А установить привязку тома. На compute сопоставить диск и backend с документацией стенда. Для варианта с IPsec выполнить `swanctl --list-sas` на оконечном устройстве до и после чтения. Сверить селекторы compute/СХД, negotiated algorithms и изменение счётчиков нужной SA. Для FC, Ceph msgr2 или иного транспорта у владельца указывают его механизм, CLI и свидетельство текущей сессии. Если транспорт или средство защиты неизвестны, результат К9 остаётся открытым.

```bash
read -rp 'UUID тестовой ВМ: ' SERVER
openstack server volume list "$SERVER"
read -rp 'UUID проверяемого тестового тома: ' VOL
openstack volume show "$VOL" -c id -c attachments -c type
```

```bash
sudo swanctl --list-sas
```

**4. Внутри тестовой ВМ** сверить диск с назначенным тестовым томом и прочитать 8 МиБ без записи. Повторить команду просмотра SA на оконечном устройстве. Для общего туннеля требуется также привязать поток к адресу/интервалу операции. Компенсирующее решение оценивается по его согласованному критерию.

```bash
lsblk -o NAME,SIZE,TYPE,SERIAL,MOUNTPOINT
read -rp 'Путь к выделенному тестовому диску: ' DISK
date -u
sudo dd if="$DISK" of=/dev/null bs=1M count=8 iflag=direct status=none
date -u
```

<a id="cli-audit"></a>

## A1. С-28 и С-30. Изменение политики и две записи во внешней SIEM

**1. На А** в рабочем каталоге E01 указать путь к фактическому репозиторию, ревизию согласованного изменения и путь к каталогу относительно корня Git. Сопоставить содержимое из Git с файлом, переданным Kolla в E02. При разном форматировании сравнить нормализованные структуры YAML/JSON. Выполнить выбранный этап E02, записать номер заявки и ID задания из используемой системы Банка. Повторить для возврата. Если применялся только локально сгенерированный файл, так и записать; это частичная проверка С-28.

```bash
read -rp 'Каталог checkout: ' REPO
read -rp 'Ревизия политики: ' REV
read -rp 'Путь файла политики внутри git: ' FILE
git -C "$REPO" show -s --format='%H %cI %s' "$REV"
git -C "$REPO" show "$REV:$FILE" > reviewed.yml
norm_policy() { python3 -c 'import json,sys,yaml; print(json.dumps(yaml.safe_load(sys.stdin),sort_keys=True,indent=2))' < "$1"; }
diff -u <(norm_policy reviewed.yml) <(norm_policy "$PROFILE")
date -u
```

Если система Банка использует GitLab CI, на А получить сведения конкретного задания через настроенный профиль glab. Команда не выводит переменные CI. Сопоставить SHA задания с ревизией. Успешный статус задания и журнал применения должны относиться к одному прогону.

```bash
read -rp 'Домен используемого GitLab: ' GITLAB_HOST
read -rp 'Числовой ID проекта: ' PROJECT_ID
read -rp 'ID задания применения: ' JOB_ID
glab api --hostname "$GITLAB_HOST" "projects/$PROJECT_ID/jobs/$JOB_ID" | jq '{id,status,sha:.commit.id,started_at,finished_at,web_url}'
```

**2. На изменённом H** выполнить `hostname`, `date -u`, `tx` (E01.3). **На П** после этапа 2 E02 записать время и попытку, которая должна быть закрыта; **на В** в тот же интервал проверить тот же адрес и порт. Указать реальный фильтр, принявший решение. У завершающего drop в каталоге отсутствует log; для С-30 понадобится действующий источник событий фильтра.

```bash
date -u
nc -vz -w 3 "$IP1" 28081
```

**3. У оператора внешней SIEM** выполнить два поиска. Ниже готовый пример для внешнего приёмника с OpenSearch-совместимым Search API. Ввести URL реального индекса с окончанием `/_search`, путь к CA, read-only netrc и запросы для схемы приёмника. Для изменения искать по имени узла, идентификатору транзакции или версии и времени; для отказа — по source/destination IP, порту, решению и времени. Примеры q для ECS — `host.name:"h1" AND event.action:"policy-change"` и `source.ip:"192.0.2.20" AND destination.ip:"192.0.2.30" AND destination.port:28081 AND event.action:"drop"`. Дату события сверить по UTC явно.

```bash
read -rp 'HTTPS URL внешнего индекса с /_search: ' SEARCH
read -rp 'CA внешней SIEM: ' SIEM_CA
read -rp 'Путь к netrc оператора SIEM: ' SIEM_AUTH
read -rp 'Поиск операции изменения по полям этого приёмника: ' Q_CHANGE
read -rp 'Поиск события отказа по полям этого приёмника: ' Q_DROP
siem_find() {
  curl --fail -sS --netrc-file "$SIEM_AUTH" --cacert "$SIEM_CA" --get "$SEARCH" --data-urlencode "q=$1" --data-urlencode 'sort=@timestamp:desc' --data-urlencode 'size=10' --data-urlencode 'filter_path=hits.hits._id,hits.hits._source.@timestamp,hits.hits._source.event,hits.hits._source.host.name,hits.hits._source.user.name,hits.hits._source.source.ip,hits.hits._source.destination'
}
siem_find "$Q_CHANGE" | jq .
siem_find "$Q_DROP" | jq .
```

Сохранить ID двух событий конечного приёмника и факты из записи. Если SIEM использует другой API, оператор выполняет два эквивалентных запроса штатным CLI и прилагает их текст и ответы. До определения продукта/схемы и получения записи именно на внешнем приёмнике результат доставки — «Не проверено». Ноль найденных записей при исправной цепочке поиска фиксируется отдельно.

<a id="host-v6"></a>

## V1. IPv6 на узлах платформы

**1. На узле с подготовленным IPv6-профилем** проверить адреса, listener и политику. Взять IPv6 действующего сервиса. Наличие tenant-подсети из N1 и результат этой проверки записываются раздельно.

```bash
ip -6 -br addr
sudo ss -6 -lnt
sudo firewall-cmd --policy=kolla-host-input --list-rich-rules | grep ipv6
```

**2. На К и З** указать один и тот же IPv6 опубликованного входа и порт, сопоставить успех из К и запрет из З с фактическим правилом. Для БД и очереди повторить nc с В и К, для прямого backend — с К, для SSH и ICMPv6 — с З согласно F05–F08. Указать адрес и порт каждого пути в протоколе.

```bash
read -rp 'IPv6 проверяемого сервиса или узла: ' V6
read -rp 'Порт проверяемого сервиса: ' VP
ip -6 route get "$V6"
nc -6 -vz -w 3 "$V6" "$VP"
ping -6 -c 2 -W 2 "$V6"
```

<a id="cli-deps"></a>

## D1. С-31/32. VLAN управления, публикация и зависимости

**1. На А** получить опубликованные интерфейсы. **На узле** получить адреса, VLAN и listeners. Если VLAN заканчивается на коммутаторе, дополнить согласованным выводом его CLI по фактическому порту. Для Linux-моста с VLAN filtering приложить также вывод bridge. Разнести в протоколе management, публичный frontend, прямой backend, storage, BMC и сети ВМ.

```bash
openstack endpoint list -c 'Service Name' -c Interface -c URL
```

```bash
ip -br addr
ip -d link show
bridge vlan show
sudo ss -lntup
sudo firewall-cmd --policy=kolla-host-input --list-rich-rules
```

**2. На А, затем на К** ввести один и тот же управляющий IP и порт из матрицы. Выполнить две команды. Из А ожидаем доступ. Из К при требуемой изоляции ожидаем запрет. Затем на К ввести публичный адрес/порт и повторить, ожидаем успех. Повторить для каждого согласованного входа (SSH, API, Horizon, Grafana, BMC). Проверить и зафиксировать разные роли frontend/backend, внешний ACL и исключение SSH в текущей политике.

```bash
read -rp 'IPv4 конкретного входа: ' DEST
read -rp 'TCP-порт: ' DPORT
ip route get "$DEST"
nc -vz -w 3 "$DEST" "$DPORT"
```

**3. На узле-клиенте каждой зависимости** для TCP повторить предыдущий шаг к фактическому адресу и порту. Для UDP нужен ответ протокола. Примеры ниже для DNS и синхронизации времени. Подставить имя из обслуживаемой зоны; для NTP сопоставить адрес и достижимость реально настроенного peer. Для LDAP/Vault/registry/SIEM/storage/BMC с TCP записать действующий адрес, штатного клиента и обратный контроль из запрещённого источника. Входящая policy узла сама по себе не устанавливает ограничения egress.

```bash
read -rp 'IP штатного DNS: ' DNS
read -rp 'Известная запись обслуживаемой зоны: ' FQDN
dig +short +time=2 +tries=1 @"$DNS" "$FQDN" A
chronyc -n sources -v
chronyc tracking
```

**4. На проверяемом H** сопоставить конкретный порт со слушателем и правилом. Вручную учесть также диапазоны, в которые входит порт, и VIP на HAProxy. Для сервиса в отдельном network namespace выполнить ss в соответствующем контейнере, если утилита в нём установлена. Повторить соединения из положительной и отрицательной точек как в шаге 2. Записать каждую пару «правило без listener» и «listener без строки реестра» для разбора.

```bash
read -rp 'Проверяемый TCP-порт: ' DPORT
sudo ss -lntp "sport = :$DPORT"
sudo firewall-cmd --policy=kolla-host-input --list-rich-rules
sudo docker ps --format '{{.Names}}'
read -rp 'Контейнер сервиса с отдельным netns: ' SERVICE
sudo docker exec "$SERVICE" ss -lntup
```
