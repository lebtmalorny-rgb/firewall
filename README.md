# Как проверить firewall в OpenStack PVS

**Порядок показа:** сначала подключитесь к одному брокеру с двух разных машин. Затем откройте и снова закройте один порт на control. В конце откройте один порт внутри ВМ. В каждом шаге ниже указано, в каком терминале вводить команду.

## Куда идёт трафик

```text
Клиент        → публичный VIP / HAProxy → Keystone API на control
Nova API      → внутренний адрес БД     → ProxySQL (если включён) → MariaDB
Nova Compute  → RabbitMQ на control     → обмен сообщениями между службами Nova
Прокси VNC    → порт консоли на compute → экран выбранной ВМ

ВМ-А → исходящие правила порта А → сеть Neutron → входящие правила порта Б → ВМ-Б
```

**Host-firewall** проверяет подключения к самому серверу control или compute. **Security Group (SG)** проверяет трафик порта ВМ. Для сайта внутри ВМ открывают порт в её SG. В типовом inventory публичный VIP находится на группе `network`. На совмещённом стенде это может быть тот же сервер, что и control.

## Что открыто на control и compute

| Куда | Основные порты по умолчанию | Откуда разрешено |
|---|---|---|
| **VIP** | TCP 443 при едином HTTPS-входе или собственный порт API | Разрешённая клиентская сеть |
| **Control — API** | Keystone 5000, Nova 8774, Cinder 8776, Glance 9292, Neutron 9696, Placement 8780, Horizon 80/443. Все TCP | HAProxy. Для этих опубликованных сервисов форк также разрешает клиентам **прямой путь** на IP узла |
| **Control — внутренние службы** | RabbitMQ TCP **5671 с TLS / 5672 без TLS**; БД TCP **3306** | Брокер — с узлов группы `common`, включая compute. MariaDB — с `common` и `loadbalancer`, прямой вход ProxySQL — с `loadbalancer`. Обычному клиенту закрыто |
| **Compute — libvirt и консоли** | TCP **16509**, при TLS также **16514**; VNC **5900–5999** | libvirt — с control и compute. VNC — с узлов `nova-novncproxy`. Клиенту прямой вход закрыт |
| **Compute — миграция и туннель** | Миграция TCP 8022 и 49152–49261; VXLAN **UDP 4789**, Geneve **UDP 6081** | Миграция — с других compute. Туннели — между узлами группы `openvswitch`; в правиле сверяют IP концов туннеля |
| **Оба узла — SSH и ping** | Обычно TCP 22; ICMP/ICMPv6 без порта | Политика узла разрешает с любого источника. Внешний межсетевой экран может сузить доступ |

<details>
<summary><strong>Все сервисные порты по ролям в примере inventory из архива</strong></summary>

В колонках собраны порты потоков, у которых узел `control01` или `compute01` входит в группу назначения в `ansible/inventory/multinode`. Число означает разрешение при включённом сервисе и выполненных условиях. Прочерк означает, что в этом примере роль размещена в другом месте. Общие правила SSH и ICMP приведены выше. При совмещении ролей набор входов изменится.

| Служба | Control | Compute |
|---|---|---|
| keystone | TCP 5000, TCP 8023 | — |
| glance | TCP 9292, TCP 9293 | — |
| cinder | TCP 8776 | — |
| nova | TCP 8774, TCP 8775, TCP 6080, TCP 6083 | TCP 5900-5999, TCP 10000–20000 |
| neutron | TCP 9696, TCP 9697 | — |
| neutron-tunnel | — | UDP 4789, UDP 6081 |
| placement | TCP 8780 | — |
| horizon | TCP 80 или 443 по backend TLS, TCP 443 | — |
| watcher | TCP 9322 | — |
| masakari | TCP 15868 | — |
| mistral | TCP 8989 | — |
| ironic | TCP 6385, TCP 8089 | — |
| mariadb | TCP 3306, TCP 4567, TCP 4568, TCP 4444 | — |
| rabbitmq | TCP 5671 TLS / 5672 без TLS, TCP 4369, TCP 25672, TCP 6000-6500, TCP 15672 | — |
| memcached | TCP 11211 | — |
| etcd | TCP 2379, TCP 2380 | — |
| proxysql | TCP 6032, TCP 3306 | — |
| nova-libvirt | — | TCP 16509, TCP 16514, TCP 49152-49261 |
| nova-ssh | — | TCP 8022 |
| hacluster | UDP 5405 | TCP 3121 |
| fluentd | UDP 5140 | UDP 5140 |
| prometheus-node-exporter | TCP 9100 | TCP 9100 |
| prometheus-mysqld-exporter | TCP 9104 | — |
| prometheus-memcached-exporter | TCP 9150 | — |
| prometheus-libvirt-exporter | — | TCP 9177 |
| prometheus-cadvisor | TCP 18080 | TCP 18080 |
| prometheus-rabbitmq-exporter | TCP 15692 | — |
| prometheus-proxysql-exporter | TCP 6070 | — |
| prometheus-fluentd-integration | TCP 24231 | TCP 24231 |
| ironic-prometheus-exporter | TCP 9608 | — |
| opensearch | TCP 9200 по умолчанию, TCP 9300 | — |
| opensearch-dashboards | TCP 5601 | — |
| barbican | TCP 9311 | — |
| security-audit-webhook | TCP 19890 | TCP 19890 |
| prometheus-elasticsearch-exporter | TCP 9108 | — |

</details>

**Как узнать, что открыто именно у вас.** На **control**, затем на **compute** выполните:

```bash
hostname
ip -br -4 addr
sudo ss -lntup
sudo firewall-cmd --info-policy=kolla-host-input
```

В `ss` найдите локальный **IP:порт и процесс**. В политике найдите тот же порт, адрес источника (`source`) и решение (`accept` или завершающий `drop`). Разрешение в таблице зависит от включённой роли; служба может не слушать порт. Для проверки «снаружи» нужен обычный клиент вне операторских и внутренних групп. Адресам из `host_firewall_operator_sources` форк разрешает больше соединений.

**Подставьте адреса один раз.** Замените в трёх строках примеры на API-IP ваших узлов и порт слушающего брокера. Вставьте этот блок в терминалы **control, compute и клиента**. Если откроете второй терминал control, вставьте туда тоже.

```bash
CONTROL=192.0.2.10
COMPUTE=192.0.2.20
MQ=5671
```

## Показ 1. К API клиент проходит, к брокеру — только разрешённый узел

| Где вводить | Команда | Что увидеть |
|---|---|---|
| **Администратор**, OpenStack CLI | `openstack catalog show identity` | Скопируйте URL из строки `public` |
| **Клиент**, Linux | `read -rp 'Вставьте public URL Keystone: ' URL` | Вставьте URL и нажмите Enter |
| **Клиент** | `curl --noproxy '*' --connect-timeout 3 --max-time 8 "$URL"` | Ответ Keystone с версией API. Для внутреннего УЦ добавьте `--cacert /путь/к/ca.pem` |
| **Compute** | `nc -vz -w 3 "$CONTROL" "$MQ"` | **Открыто** — `succeeded` или `open` |
| **Клиент** | `nc -vz -w 3 "$CONTROL" "$MQ"` | **Закрыто** для клиента при работающем брокере |

Чтобы показать конкретное разрешение в большом списке правил, **на control** выполните:

```bash
sudo firewall-cmd --policy=kolla-host-input --list-rich-rules | grep -w "$MQ"
```

**Покажите вход на compute.** Если в `ss` на compute слушает 16514, выполните `nc -vz -w 3 "$COMPUTE" 16514` **сначала на control, потом на клиенте**. Ожидаем успех только с control. Если слушает 16509, подставьте его. Если оба порта не слушают, запишите «libvirt не проверен».

<details>
<summary>Как увидеть реальные соединения сервисов и проверить прямой доступ в обход VIP</summary>

| Терминал | Команда | Что искать |
|---|---|---|
| **Control с RabbitMQ**, Kolla с Docker | `sudo docker exec rabbitmq rabbitmqctl list_connections peer_host port user` | IP compute, порт брокера и служебная учётная запись. Это реальные соединения приложений |
| **Узел с Nova API** | `sudo ss -ntp 'dport = :3306'` | Сессия процесса к внутреннему адресу БД. При ProxySQL на его узле той же командой виден второй участок до MariaDB |
| **Узел HAProxy** | `sudo ss -ntp 'dport = :5000'` | Во время запроса клиента виден backend Keystone. У короткого запроса сессия может успеть закрыться |
| **Клиент** | `nc -vz -w 3 "$CONTROL" 5000` | Если на control слушает Keystone и проба успешна, клиент достигает backend напрямую |

</details>

## Показ 2. Порт на control — закрыто → открыто → закрыто

Используйте свободный TCP **28081** на испытательном control. Во время этих четырёх шагов Kolla не должна применять другую политику.

**1. Control, терминал 1.** Первая команда должна показать пустой список слушателей. Вторая запускает тестовый сервис. Оставьте его работающим.

```bash
ss -lnt 'sport = :28081'
python3 -m http.server 28081 --bind "$CONTROL" --directory "$(mktemp -d)"
```

**2. Control, терминал 2.** Проверьте `curl --noproxy '*' -s -o /dev/null -w '%{http_code}\n' "http://$CONTROL:28081/"` — ожидаем **200**. Затем **на compute** выполните `nc -vz -w 3 "$CONTROL" 28081` — ожидаем **тайм-аут**.

**3. Control, терминал 2.** Добавьте временное правило только для IP compute. Сначала запросите правило: ожидается `no`. Затем добавьте его в действующую политику. Если первая проба с compute уже прошла или правило уже есть, для опыта нужен другой свободный порт.

```bash
RULE="rule family=ipv4 priority=-20000 source address=$COMPUTE/32 destination address=$CONTROL/32 port port=28081 protocol=tcp accept"
sudo firewall-cmd --policy=kolla-host-input --query-rich-rule="$RULE"
sudo firewall-cmd --policy=kolla-host-input --add-rich-rule="$RULE"
```

**4. Проверьте открытие и верните исходное состояние.** Оставьте сервис включённым до последней пробы.

| Где вводить | Что ввести | Ожидание |
|---|---|---|
| **Compute** | `nc -vz -w 3 "$CONTROL" 28081` | **Открыто** |
| **Control, терминал 2** | `sudo firewall-cmd --policy=kolla-host-input --remove-rich-rule="$RULE"` | `success` |
| **Compute** | `nc -vz -w 3 "$CONTROL" 28081` | **Снова закрыто** |
| **Control, терминал 2** | Повторите команду curl из шага 2 | Сервис всё ещё отвечает **200** |

Теперь остановите HTTP-сервис клавишами Ctrl+C в первом терминале. Если показ прервался после добавления правила, выполните строку `--remove-rich-rule` на control. В новом терминале сначала заново задайте адреса и `RULE`.

Три пробы проверяют одно соединение с одной и той же работающей службой. Изменилось только правило host-firewall. Если положительная проба не прошла, результат остаётся «Не проверено». На compute команда `ip route get "$CONTROL"` показывает исходный IP в поле `src`. При NAT в правило нужно подставить адрес, который видит control. Для проверки управления через Kolla в полной ПМИ есть отдельный сценарий E02.

## Показ 3. Порт внутри ВМ — закрыто → открыто → закрыто

Нужны **ВМ-А и ВМ-Б в одной подсети**. ВМ-А входит в SG `pmi-src` с разрешённым исходящим TCP. На порту ВМ-Б — SG `pmi-dst`, исходно без входящего разрешения TCP 8080.

<details>
<summary><strong>Если ВМ ещё нет — создать сеть и обе ВМ через CLI</strong></summary>

**На машине администратора** откройте Bash с доступом к OpenStack. Используйте отдельный тестовый проект, в котором имена `pmi-*` свободны. Выберите свободную подсеть и образ Linux с Python 3, утилитой nc и подготовленным входом через консоль. Для примера взята 192.168.240.0/24. Обе ВМ в одной подсети, внешний шлюз им не требуется.

```bash
openstack token issue -f value -c project_id
openstack network create --enable-port-security pmi-net
openstack subnet create --network pmi-net --subnet-range 192.168.240.0/24 pmi-v4
openstack security group create pmi-src
openstack security group create pmi-dst
openstack security group rule list pmi-src
openstack security group rule list pmi-dst
```

**Там же** введите имя доступного образа и flavor. Создайте ВМ и запишите фактический IPv4 ВМ-Б из последней команды.

```bash
openstack image list
openstack flavor list
read -rp 'Имя образа: ' IMAGE
read -rp 'Имя flavor: ' FLAVOR
openstack server create --wait --image "$IMAGE" --flavor "$FLAVOR" --network pmi-net --security-group pmi-src pmi-a
openstack server create --wait --image "$IMAGE" --flavor "$FLAVOR" --network pmi-net --security-group pmi-dst pmi-b
openstack server list --name pmi-
```

**Там же** получите ссылки на консоли. Откройте каждую и войдите в гостевую ОС. Ссылки на консоли содержат временные токены. В протокол внесите имена ВМ.

```bash
openstack console url show pmi-a
openstack console url show pmi-b
```

</details>

**На ВМ-А** введите адрес ВМ-Б из `openstack server list`. Команды OpenStack ниже выполняйте в одном терминале администратора.

```bash
read -rp 'IPv4 ВМ-Б: ' VM_B
```

| Шаг | Где вводить | Что ввести | Ожидание |
|---|---|---|---|
| 1 | **ВМ-Б**, консоль гостя | `python3 -m http.server 8080 --bind 0.0.0.0 --directory "$(mktemp -d)"` | HTTP-сервис запущен. Оставьте его работать |
| 2 | **ВМ-А**, консоль гостя | `nc -vz -w 3 "$VM_B" 8080` | Закрыто |
| 3 | **Администратор**, OpenStack CLI | `RULE=$(openstack security group rule create --ingress --ethertype IPv4 --protocol tcp --dst-port 8080 --remote-group pmi-src pmi-dst -f value -c id)` | Разрешён трафик от группы ВМ-А к порту 8080 группы ВМ-Б |
| 4 | **ВМ-А** | `nc -vz -w 3 "$VM_B" 8080` | Открыто |
| 5 | **Администратор**, OpenStack CLI | `openstack security group rule delete "$RULE"` | Удалено одно тестовое правило |
| 6 | **ВМ-А** | `nc -vz -w 3 "$VM_B" 8080` | Снова закрыто. Теперь на ВМ-Б остановите сервис клавишами Ctrl+C |

После изменения SG дайте Neutron несколько секунд и каждый раз открывайте **новое** соединение. Если шаг 4 не прошёл, изоляция пока не доказана: проверьте IP ВМ-Б, слушатель, её гостевой firewall и привязку SG командой `openstack port list --server pmi-b`, затем `openstack port show UUID_ПОРТА -c fixed_ips -c security_group_ids -c port_security_enabled`.

<details>
<summary><strong>Как увидеть путь между двумя compute</strong></summary>

**Администратор:** выполните две команды с правом просмотра размещения. При одном имени compute пакеты этой пары проходят локально через виртуальную сеть. При разных compute и сети VXLAN/Geneve внутренний пакет ВМ передаётся между узлами в UDP-туннеле.

```bash
openstack server show pmi-a -c OS-EXT-SRV-ATTR:host
openstack server show pmi-b -c OS-EXT-SRV-ATTR:host
```

**На принимающем compute**, при открытом правиле шага 4 начните захват на tunnel-интерфейсе. Замените `tun0` на его фактическое имя и `192.0.2.51` на tunnel-IP отправляющего compute. **На ВМ-А** сразу повторите пробу к 8080. На узле виден внешний UDP-пакет между tunnel-IP, внутри ВМ проверяется TCP 8080. Привяжите наблюдение ко времени пробы и паре узлов: в общем туннеле есть трафик других ВМ. В данном форке каталог строит разрешения туннелей по API-IP. Когда tunnel-IP отличается, сверяйте адрес назначения в действующем правиле с фактическим tunnel-IP.

```bash
sudo timeout 12 tcpdump -nni tun0 'src host 192.0.2.51 and (udp dst port 4789 or udp dst port 6081)'
```

</details>

<details>
<summary><strong>Как убрать тестовые ВМ и сеть</strong></summary>

**На машине администратора** выполните после показа, если создавали эти ресурсы в блоке подготовки. Сначала сверьте проект и имена.

```bash
openstack token issue -f value -c project_id
openstack server list --name pmi-
openstack server delete --wait pmi-a pmi-b
openstack subnet delete pmi-v4
openstack network delete pmi-net
openstack security group delete pmi-dst pmi-src
```

Справка по командам: [firewall-cmd](https://firewalld.org/documentation/man-pages/firewall-cmd.html) · [правила Security Groups OpenStack](https://docs.openstack.org/python-openstackclient/2025.1/cli/command-objects/security-group-rule.html).

</details>

Испытания на вашем стенде ещё предстоят. Для протокола запишите **источник → IP назначения → порт → результат до / после / после возврата**. Если видите `Connection refused`, проверьте, слушает ли сервис. Тайм-аут без успешной контрольной попытки запишите как «Не проверено».

[Приложение для полной приёмки ДКБ-С: методика, каталог портов и протокол](reference/Методика_firewall.md)
