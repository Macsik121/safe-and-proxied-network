Ниже представлен полный текст документа, конвертированный в формат Markdown. Все данные из визуальных дашбордов (сводка KPI, локализация по подсистемам и цепочки отказов) интегрированы в текстовую структуру отчета.

---

# Системный технический аудит инженерной спецификации «Цензуроустойчивый сетевой шлюз (CRNE)» — Редакция v4.1

Детальный перекрёстный аудит инженерной спецификации «Цензуроустойчивый сетевой шлюз (Censorship-Resistant Network Gateway) — Редакция v4.1 (Production-Ready)» выявил 16 технических дефектов, распределённых по четырём уровням критичности: 3 критических блокера (Showstoppers), 4 архитектурных и сетевых дефекта высокой тяжести (High), 5 операционных и скриптовых ошибок средней тяжести (Medium) и 4 несоответствия в программе приёмочных испытаний ATP и документации (Low).

Несмотря на то что редакция v4.1 устранила ошибку предыдущей ревизии (удаление несуществующего inbound `"type": "dns"` в пользу перехвата через `hijack-dns`), текущая кодовая база спецификации не пройдёт первичное развёртывание по Runbook (Раздел 6.1) и столкнётся с нарушением работы локального DNS, неработоспособностью окна Captive Portal и сбоями мониторинга в среде OpenWrt 24.10.5+ и sing-box 1.12.x+.

Главные стоп-факторы текущей редакции лежат на стыке трёх подсистем: компилятора подписок `sub2sing.py`, автоматической генерации таблиц nftables движком sing-tun (`auto_redirect: true`) и новой архитектуры резолвинга доменов в `sing-box 1.12+`. 
Во-первых, скрипт `sub2sing.py` (Приложение А) падает с фатальным исключением `TypeError` при любом вызове из-за несовпадения сигнатуры функции `build_router_config()` (5 параметров в объявлении против 6 при вызове), а блок атомарной выгрузки `--push` содержит логическую ошибку конкатенации shell-команд (`;` после цепочки `&&`), которая маскирует сбой валидации кодом возврата 0 и мгновенно завершает 300-секундный таймер автоотката. 
Во-вторых, одновременное использование `"auto_redirect": true` и `"default_mark": 255 (0xff)` прямо конфликтует с внутренними метками sing-tun (0x2023/0x2024), а создаваемая `auto_redirect` таблица `table inet sing-box` перехватывает запросы LAN-клиентов к `10.10.1.1:53` на этапе `prerouting` мимо `dnsmasq`, нарушая разрешение локальных имён `.lan` и проверку Canary-доменов. 
В-третьих, глобальная настройка `"default_domain_resolver": "dns-bootstrap"` в `sing-box 1.12+` заставляет `direct` outbound обходить правила `dns.rules` (включая блокировку рекламы и российский DoH), отправляя DNS-запросы открытым UDP на `77.88.8.8`.

## 1. Сводка дефектов по уровням критичности (KPI) и подсистемам

**Профиль дефектов спецификации CRNE v4.1: 7 критических и архитектурных сбоев блокируют ввод в эксплуатацию.**

### 1.1. Распределение по уровням критичности

| Уровень критичности | Количество | Доля | Основные зоны поражения |
| :--- | :--- | :--- | :--- |
| **Critical (Blocker)** | 3 | 18.8% | Сбой сборки и CI/CD |
| **High (Architecture/Net)** | 4 | 25.0% | Петли DNS и отказ Captive Portal |
| **Medium (Operations)** | 5 | 31.2% | Слепота мониторинга и парсера |
| **Low (ATP & Docs)** | 4 | 25.0% | Ложные падения тестов T15/T27 |

### 1.2. Локализация дефектов по инженерным подсистемам

| Подсистема | Critical | High | Medium | Low | Всего |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `sub2sing.py` (CI/CD & Парсер) | 2 | 0 | 1 | 0 | **3** |
| `sing-box 1.12+` (DNS & Маршрутизация) | 0 | 2 | 0 | 1 | **3** |
| `nftables` & Multi-WAN / Captive Portal | 0 | 2 | 0 | 0 | **2** |
| Watchdog, Alerting & Init-скрипты | 0 | 0 | 3 | 1 | **4** |
| Runbook, Sysupgrade & ATP T1-T27 | 1 | 0 | 1 | 2 | **4** |

### 1.3. Цепочки отказов и нарушение архитектурных инвариантов

| Уровень и узел | Механизм дефекта в спецификации | Последствия при отказе (Impact) |
| :--- | :--- | :--- |
| **3 CRIT, 2 MED** <br> `sub2sing.py` & Сборка / Sysupgrade | Ошибки компилятора подписок `sub2sing.py` при генерации профиля и несогласованность путей сохранения конфигурации в процедурах `sysupgrade`. | **Сбой сборки и CI/CD:** блокировка выпуска валидного артефакта конфигурации, потеря параметров парсером и сбой восстановления узла. |
| **2 HIGH, 1 LOW** <br> `sing-box 1.12+` (DNS Router) | Параметр `route.default_domain_resolver` предзаполняет `options.Transport` в `dialer.go`, из-за чего в `dns/router.go` пропускается вызов `r.matchDNS()`. | **Обход правил DNS:** все запросы `direct`-исходящих соединений игнорируют `dns.rules` (требуется явный `action: "resolve"`). |
| **2 HIGH** <br> `nftables` & TUN `auto_redirect` | Включение `auto_redirect` в `table inet sing-box` жёстко внедряет DNAT для `dport 53` в цепочки `prerouting` (`dstnat + 1`) и `output`, а также конфликтует с `route.default_mark`. | **Петли DNS и отказ Captive Portal:** перехват трафика `dnsmasq` (порт 53 → 5353), игнорирование bypass-правил и конфликт меток `0x2023/0x2024` в Multi-WAN. |
| **3 MED, 3 LOW** <br> Watchdog & Приёмочные тесты ATP | Расхождение порогов проверки доступности в скриптах мониторинга / инициализации и некорректные критерии верификации в сценариях приёмки `T15/T27`. | **Слепота мониторинга и ложные падения:** пропуск деградации туннеля демоном Watchdog и ложноотрицательные результаты тестов `T15/T27`. |

---

## 2. Критические дефекты (Blocker / Showstopper): Отказ компиляции и развёртывания

Ошибки данной категории делают невозможным выполнение регламента первичной установки (Раздел 6.1) и эксплуатацию конвейера доставки конфигураций (Раздел 6.3 и Приложение А).

### 2.1. Фатальное исключение TypeError в компиляторе sub2sing.py (Несовпадение сигнатуры функции)
В Приложении A (`sub2sing.py`) обнаружено рассогласование между объявлением функции генерации конфигурации `build_router_config()` и её вызовом внутри `main()`. При миграции кода скрипта с ревизии v4.0 (где поддерживалась генерация двух форматов DNS для sing-box 1.11 и 1.12 через параметр `sb_minor`) из определения функции `build_router_config(standard_top: list[dict], lts_nodes: list[dict], extra_manual: list[dict], tolerance: int, secret: str) -> dict:` был удалён шестой аргумент `sb_minor`, так что функция принимает ровно 5 позиционных аргументов.

Однако в теле функции `main()` вызов остался прежним и передаёт 6 позиционных аргументов: `cfg = build_router_config(standard_top, lts_nodes, extra_manual, args.tolerance, secret, sb_minor)`. Вследствие этого любой запуск `python3 ./sub2sing.py` (включая Шаг 5 первичного развёртывания в Разделе 6.1 и генерацию шаблона `--init-template`) завершается аварийным падением интерпретатора Python с исключением `TypeError: build_router_config() takes 5 positional arguments but 6 were given` ещё до записи `config.json` и до выгрузки на роутер. Дополнительно функция `detect_sb_minor_version()` и CLI-аргумент `--sb-minor` (на который ссылается предупреждение Слоя 1: `sub2sing.py --sb-minor 12`) превратились в неиспользуемый код, никак не влияющий на структуру генерируемого JSON.

### 2.2. Подавление кода ошибки валидации и мгновенное завершение таймера отката в sub2sing.py --push
В Приложении A (`sub2sing.py`) формирование и выполнение переменной `remote_cmd` в блоке `if args.push:` отличается от сокращённой версии из Раздела 6.3 и содержит три взаимосвязанных дефекта построения shell-конвейера.

Первый дефект связан с маскировкой сбоя валидации точкой с запятой (`;`): цепочка проверок `jq empty /tmp/config.json.new && sing-box check -c /tmp/config.json.new && fw4 check && ...` соединена операторами `&&` вплоть до фрагмента `kill $(cat /tmp/crne_push_rollback.pid) 2>/dev/null; rm -f /tmp/crne_push_rollback.pid && ... >/dev/null 2>&1; rm -f /tmp/config.json.new`. В POSIX-шелле (`ash/sh` в BusyBox) при провале любой из начальных проверок (`jq`, `sing-box check` или `fw4 check`) выполнение текущей `&&`-последовательности прерывается, но интерпретатор переходит к командам после точки с запятой (`;`). Команда `rm -f /tmp/config.json.new` в самом конце строки всегда завершается успешно и возвращает код 0. В результате вызов `subprocess.run(["ssh", args.push, remote_cmd], check=True)` получает код возврата 0, блок `except subprocess.CalledProcessError` не срабатывает, и скрипт выводит ложное сообщение `[OK] Конфиг прошёл двойную валидацию...`, тогда как в действительности новый конфиг не был активирован, а временный файл был удалён.

Второй дефект в этом же блоке разрушает логику 300-секундного таймера автоотката (Lock-Rollback). Сразу после запуска фонового таймера отката скрипт выполняет `/etc/init.d/sing-box restart && kill $(cat /tmp/crne_push_rollback.pid) 2>/dev/null`. В OpenWrt init-скрипт `procd` при вызове `restart` лишь отправляет асинхронную команду в `ubus` и завершается за десятки миллисекунд с кодом 0, не дожидаясь инициализации интерфейса `tun0` или проверки сетевой связности. Вследствие этого следующая за ним команда `kill` уничтожает сторожевой таймер отката сразу после его создания. Кроме того, в отличие от эталонного шаблона из Раздела 6.2 (`trap "" HUP INT TERM ... </dev/null`), внутри `remote_cmd` в `sub2sing.py` фоновый процесс запускается как `( nohup sh -c 'sleep 300 ...' </dev/null 2>&1 & echo $! ) >/tmp/crne_push_rollback.pid` — без закрытия `stdin` (`</dev/null`) и без перехвата сигнала SIGHUP. Если команда `kill` не сработает (например, при обрыве связи на шаге `restart`), SSH-сессия может повиснуть на все 300 секунд, ожидая закрытия дескриптора стандартного ввода порождённого сабшелла.

### 2.3. Отсутствие пакета git, модулей NFQUEUE и старта служб в регламенте первичной установки (Раздел 6.1)
Регламент первичного ввода роутера в эксплуатацию (Раздел 6.1, Шаги 1–6) содержит три пропуска, препятствующих штатному запуску по инструкции. 
Во-первых, Раздел 6.3 и `sub2sing.py --push` требуют инициализации локального гит-репозитория на роутере (`cd /etc/sing-box && git init`), однако в Шаге 1 Раздела 6.1 команда `opkg install sing-box curl jq ca-bundle kmod-tun kmod-nft-tproxy` не включает пакет `git`. Поскольку в стандартную сборку OpenWrt 24.10.5+ `git` не входит, выполнение команд Раздела 6.3 завершится ошибкой `sh: git: not found`.

Во-вторых, в списке устанавливаемых модулей ядра указан `kmod-nft-tproxy`, хотя спецификация v4.1 использует режим `tun` с `"auto_redirect": true` вместо TPROXY. При этом для работы механизма предварительного сопоставления правил (prematch) в `auto_redirect` (`sing-box 1.12+` / `1.13+`) создаются цепочки `prerouting_prematch` и `output_prematch`, которые направляют TCP SYN пакеты в пользовательское пространство через NFQUEUE (`queue flags bypass to 100`). Официальная документация `sing-box` прямо указывает, что для `auto_redirect` необходима поддержка ядром nfqueue (`nfnetlink_queue`), то есть пакет `kmod-nft-queue`, который не входит в базовые зависимости пакета `sing-box` в OpenWrt (`+ca-bundle +kmod-inet-diag +kmod-tun`). 
В-третьих, в конце Шага 6 Runbook выполняется только сухая проверка `fw4 check && sing-box check -c /etc/sing-box/config.json`, но отсутствует команда перезагрузки межсетевого экрана `/etc/init.d/firewall reload` (необходимая для включения файла `/etc/nftables.d/99-crne-security.nft` в контекст `table inet fw4`).

---

## 3. Архитектурные и сетевые ошибки высокой критичности (High Severity)

Дефекты этого уровня возникают из-за расхождения между теоретическим описанием работы сетевого стека в Разделах 3–4 спецификации и фактической реализацией подсистем sing-tun (`table inet sing-box`), `dns/router.go` и `firewall4` в ядре Linux.

### 3.1. Перехват запросов к 10.10.1.1:53 таблицей inet sing-box в обход dnsmasq (Отказ зоны .lan)
Архитектура Слоя 4 (Раздел 3) и схема в Разделе 4 строятся на допущении, что клиент в локальной сети отправляет штатный DNS-запрос на `10.10.1.1:53`, который сначала принимается локальным демоном `dnsmasq` (где разрешаются локальные имена `.lan`, приватные PTR и возвращается `NXDOMAIN` для Canary-доменов `use-application-dns.net` и `_dns.resolver.arpa`), после чего внешние запросы пересылаются демоном `dnsmasq` на `1.1.1.1` и перехватываются механизмом `auto_redirect`.

Однако анализ правил nftables, автоматически генерируемых `sing-box` при `"auto_redirect": true` в таблице `table inet sing-box` (подтверждено в GitHub Issues #3487, #3560, #3584, #3705), показывает иной порядок правил в цепочке `prerouting` (`type nat hook prerouting priority dstnat + 1`):
```nft
Правило принудительного перехвата DNS (ip saddr @inet4_local_address_set ... th dport 53 dnat ip to 172.19.0.2, где 172.19.0.2 — следующий IP-адрес после 172.19.0.1/30 на интерфейсе tun0) располагается выше правила исключения локальных адресов назначения ip daddr @inet4_local_address_set return.
```
Из-за этого любой DNS-запрос от клиента из локальной сети `10.10.1.0/24` (входящей в `@inet4_local_address_set`), направленный непосредственно на IP роутера `10.10.1.1:53`, перехватывается в `prerouting` ядром и перенаправляется через DNAT на внутренний адрес туннеля `172.19.0.2` (внутрь sing-box), вообще не доходя до сокета `dnsmasq`.

Это приводит сразу к двум функциональным отказам. Во-первых, поскольку `sing-box` не имеет доступа к таблице DHCP-аренд `dnsmasq` и содержит первым правилом в `dns.rules` запись `{"domain_suffix": [".lan", ...], "action": "predefined", "rcode": "NXDOMAIN"}`, все запросы клиентов к локальным хостам (`nas.lan`, `printer.lan`) завершаются ответом `NXDOMAIN` вместо возврата их локальных IP-адресов. Во-вторых, настроенные только в `dnsmasq` правила для Canary-доменов (`use-application-dns.net` и `_dns.resolver.arpa`) никогда не срабатывают для запросов из LAN, а в блоке `dns.rules` самого `sing-box` эти Canary-домены отсутствуют.

При этом проверка №5 в `crne-watchdog.sh` (`nslookup ya.ru 10.10.1.1`), запускаемая на самом роутере, маскирует проблему: локальный пакет идёт через интерфейс `lo`, игнорируется правилом `oifname != "lo"` в цепочке `output`, попадает в `dnsmasq`, а уже исходящий запрос от `dnsmasq` к `1.1.1.1:53` перехватывается цепочкой `output` таблицы `inet sing-box`. Для устранения дефекта необходимо перенести Canary-домены в `dns.rules` внутри `config.json`, а `dnsmasq` перевести на нестандартный локальный порт (например, `127.0.0.1:5353`) и направлять в него запросы к зоне `.lan` из `sing-box`, как рекомендовано разработчиком `sing-box` в Issue #3705.

### 3.2. Документированный конфликт auto_redirect: true и route.default_mark: 255 (0xff)
В Разделе 3 (Слой 1 и Слой 3), Разделе 5.1 (chain `crne_output_bypass`) и Разделе 5.2 (`config.json`) одновременно активированы опция `"auto_redirect": true` в `tun` inbound и глобальная маркировка сокетов `"default_mark": 255 (0xff)` в блоке `route`. Однако официальная документация `sing-box` в разделе TUN Inbound содержит прямое предупреждение для опции `auto_redirect`: `Conflict with route.default_mark and [dialOptions].routing_mark`.

Причина конфликта кроется в том, что подсистема `auto_redirect` управляет маршрутизацией и защитой от петель через собственные метки пакетов: `auto_redirect_input_mark` (по умолчанию `0x2023`), `auto_redirect_output_mark` (по умолчанию `0x2024`) и `auto_redirect_reset_mark` (`0x2025`). В цепочках `output`, `output_prematch` и `output_udp_icmp` таблицы `inet sing-box` правила выхода из перехвата проверяют именно метку `meta mark 0x00002024 return`. Более того, как показано при разборе исходного кода `sing-tun` в Issue #4505, при использовании бэкенда nftables обработчик NFQUEUE инициализирует нулевую битовую маску (`markMask = 0`) и при вынесении вердикта `ActionBypass` перезаписывает всю 32-битную метку пакета значением `AutoRedirectOutputMark` (`0x2024`), стирая сторонние биты `SO_MARK`.

С этим же механизмом связана неточность в пояснении к Слою 4 (Раздел 3) и в комментарии внутри `sub2sing.py`, где утверждается: «dns-bootstrap с `server: "77.88.8.8"` работает внутри Go runtime и не проходит через сетевой стек роутера, поэтому не перехватывается auto_redirect». В действительности любой UDP-сокет Go runtime отправляет пакеты через стандартный сетевой стек ядра Linux и проходит хук `output` в nftables, где установлено правило `meta nfproto ipv4 oifname != "lo" meta l4proto { tcp, udp } th dport 53 dnat ip to 172.19.0.2`. Единственная причина, по которой запрос `dns-bootstrap` к `77.88.8.8:53` не заворачивается этим правилом обратно в туннель (в бесконечную петлю), заключается в том, что первой строкой в `chain output` таблицы `inet sing-box` стоит проверка метки `meta mark 0x00002024 return`. Чтобы устранить конфликт и сохранить совместимость с правилами телеметрии (`crne_output_bypass`) и изоляцией Zapret, следует удалить `"default_mark": 255` из блока `route` и явно задать `"auto_redirect_output_mark": "0x000000ff"` в настройках `tun` inbound (либо использовать штатную метку `0x00002024` в правилах nftables).

### 3.3. Игнорирование dns.rules для direct outbound из-за route.default_domain_resolver
В Разделе 5.2 (`config.json`) и в Приложении А (`sub2sing.py`) в секции `route` задан параметр `"default_domain_resolver": "dns-bootstrap"`. В архитектуре `sing-box 1.12.0+` (как задокументировано в GitHub Issues #3668, #3596 и #4444) при инициализации исходящего подключения через `direct` outbound модуль `common/dialer/dialer.go` автоматически присваивает полю `dnsQueryOptions.Transport` транспорт, указанный в `route.default_domain_resolver` (в данном случае — `dns-bootstrap`, т.е. незашифрованный UDP `77.88.8.8`). Когда `direct` outbound запрашивает разрешение доменного имени в `dns/router.go` (строки 351–375), выполняется проверка условия `if options.Transport != nil`:
```go
if options.Transport != nil {
    response, err = r.client.Lookup(ctx, transport.domain, options)
} else {
    transport, ruleIndex = r.matchDNS(ctx, false, ruleIndex, true, options)
}
```
Поскольку `options.Transport` уже заполнен транспортом `dns-bootstrap`, условие `options.Transport != nil` всегда истинно, и `sing-box` вызывает `r.client.Lookup` напрямую, никогда не заходя в ветку `else` (`r.matchDNS`) и полностью игнорируя все правила в `dns.rules`. В результате любое соединение, маршрутизируемое в `direct` по доменному имени (например, при срабатывании `{"action": "sniff"}` на трафике к `.ru` / `geosite-ru` или при запросах через `socks5h://127.0.0.1:1603`), резолвит домены открытым текстом по UDP через `77.88.8.8:53`, минуя зашифрованный `dns-direct-ru` (`https://common.dot.dns.yandex.net/dns-query`) и обходя блокировку рекламных доменов `geosite-ads` на уровне резолвера. Для исправления следует добавить нетерминальное действие `{"action": "resolve"}` в `route.rules` и изменить `default_domain_resolver` на `"dns-direct-ru"` (поскольку сам `dns-direct-ru` уже имеет собственный `"domain_resolver": "dns-bootstrap"` в `dns.servers`).

### 3.4. Отказ трёхступенчатого окна Captive Portal (Четыре наложенных бага в Multi-WAN)
Подсистема прохождения порталов авторизации в отелях и публичных сетях Wi-Fi на роутере Cudy TR3000 (Разделы 5.1, 5.4 и 5.5) содержит четыре наложенных дефекта, блокирующих открытие окна авторизации.

Первый дефект находится в `/etc/nftables.d/99-crne-security.nft`, где правило №2 в цепочке `crne_forward_guard` записано как `iifname @crne_cp_window oifname { "wwan", "trm_wwan" } ct state new,established meta l4proto { tcp, udp } th dport { 53, 80, 443 } counter accept`. В nftables выражение `oifname` сопоставляет имя реального сетевого интерфейса ядра Linux (`net_device->name`), видимое в выводе `ip link`, а не логическое имя интерфейса UCI из `/etc/config/network` (`trm_wwan` или `wwan`). В OpenWrt 24.10+ на платформе MT7981B (mac80211) клиентский радиоинтерфейс travelmate на уровне ядра получает имя вида `phy0-sta0` или `phy1-sta0` (если оно явно не переопределено параметром `ifname` в конфигурации беспроводной сети). Вследствие этого условие `oifname { "wwan", "trm_wwan" }` не выполняется, и пакеты клиентов к странице авторизации отеля отвергаются правилом №5 (`iifname "br-lan" oifname != "tun0" reject`).

Второй дефект возникает в `/etc/hotplug.d/iface/99-singbox-wan-switch`: когда при событии `ifup` интерфейса `trm_wwan` запускается проверка портала (`nslookup captive.apple.com "$WAN_DNS"` и `curl -s -m 4 --interface "$DEVICE" --resolve "captive.apple.com:80:$CP_IP" ...`), демон `sing-box` всё ещё активен. При активном `sing-box` запрос `nslookup` с роутера на `$WAN_DNS:53` перехватывается цепочкой `output` таблицы `inet sing-box` (`oifname != "lo" ... th dport 53 dnat ip to ...`), а запрос `curl` с флагом `--interface "$DEVICE"`, использующий системный вызов `SO_BINDTODEVICE`, согласно официальной документации `sing-box`, при комбинации `"auto_redirect": true` и `"strict_route": true` принудительно перенаправляется внутрь `sing-box` («When auto_redirect is enabled, strict_route also affects SO_BINDTODEVICE traffic: Enabled: SO_BINDTODEVICE traffic is redirected through sing-box»).

Третий дефект связан с парсингом вывода утилиты BusyBox `nslookup` в команде `nslookup captive.apple.com "$WAN_DNS" 2>/dev/null | awk '/^Address/ && !/:53/{print $2; exit}'`. Скрипт предполагает, что строка с адресом самого DNS-сервера всегда содержит порт `:53`. Однако в BusyBox `nslookup` на OpenWrt строка заголовка сервера при указании IP-адреса может выводиться без порта (`Address: 192.168.x.x` или `Address 1: 192.168.x.x`). Из-за немедленного выхода `exit` на первом совпадении `awk` извлекает IP-адрес самого DNS-сервера `$WAN_DNS` вместо IP-адреса `captive.apple.com`, после чего `curl --resolve "captive.apple.com:80:$WAN_DNS"` обращается на 80-й порт DNS-сервера, получает отказ и ложно активирует режим Captive Portal (останавливая `sing-box` на 180 секунд) при каждом подключении к обычному Wi-Fi.

Четвёртый нюанс касается семантики базовых цепочек nftables: поскольку `crne_forward_guard` зарегистрирована с приоритетом `filter - 5`, вердикт `accept` в ней не освобождает пакет от прохождения основной цепочки `chain forward` таблицы `inet fw4` (приоритет `filter = 0`), поэтому интерфейс `trm_wwan` должен быть корректно привязан к зоне `wan` в `/etc/config/firewall` до срабатывания хука.

---

## 4. Операционные, скриптовые и функциональные ошибки средней тяжести (Medium Severity)

### 4.1. Невыполненное отключение procd file в /etc/init.d/sing-box и холостая инъекция GOMEMLIMIT
В преамбуле спецификации (п. 3 списка ключевых улучшений v4.1), в комментариях к `crne-watchdog.sh` (Раздел 5.6.2) и в Шаге 4 Runbook (Раздел 6.1) заявлено: «Отключён `procd file` во избежание гонки перезапусков с `sub2sing.py --push`». Однако штатный пакет `sing-box` в OpenWrt поставляется с init-скриптом `/etc/init.d/sing-box`, в котором строка `procd_set_param file "$conffile"` уже включена по умолчанию. Ни в Шаге 4 Runbook, ни в `crne-watchdog.sh` нет команды удаления этой строки (`sed -i '/procd_set_param file/d' /etc/init.d/sing-box`), поэтому в действительности `procd file` остаётся активным и вызывает параллельный перезапуск при модификации `/etc/sing-box/config.json`. Кроме того, когда `crne-watchdog.sh` (блок 0) добавляет через `sed -i` параметр `GOMEMLIMIT=75MiB` в `/etc/init.d/sing-box`, он не перезапускает службу, оставляя текущий процесс `sing-box` работать без ограничения памяти Go runtime до следующего аварийного сбоя.

### 4.2. Потеря метрик Prometheus при аварии туннеля в crne-watchdog.sh
В сторожевом скрипте `/usr/bin/crne-watchdog.sh` (Раздел 5.6.2) нарушена последовательность обработки аварийных состояний и экспорта телеметрии. В блоке 3 («Проверка наличия процесса sing-box и интерфейса tun0»), если процесс `sing-box` завершился или интерфейс `tun0` исчез, скрипт устанавливает переменную `TUNNEL_STATUS=0`, вызывает `/etc/init.d/sing-box restart` и на следующей же строке завершает работу вызовом `exit 1`. Поскольку генерация файла метрик `/tmp/node_exporter/textfile_collector/crne.prom` расположена в блоке 7 в самом конце скрипта, при любом падении `sing-box` выполнение до блока 7 не доходит, и в Prometheus сохраняется устаревшее значение `crne_tunnel_status 1`. Аналогично в блоке 2 при сбое keepalive-пинга через модем `usb0` скрипт перезапускает интерфейс `usb_wan`, но оставляет переменную `USB_STATUS=1`.

### 4.3. Блокировка и сбой доставки Telegram-алертов (crne-alert.sh) при падении туннеля
Скрипт оповещений `/usr/bin/crne-alert.sh` (Раздел 5.6.1) выполняет синхронный запрос `curl -s -X POST "https://api.telegram.org/bot$TG_BOT_TOKEN/sendMessage" -d "chat_id=$TG_CHAT_ID" -d "text=🚨%20CRNE%20ALERT%20[$SITE_TAG]:%20$MESSAGE" -d "parse_mode=HTML"` без флага ограничения времени (`-m` / `--max-time`) и без перевода в фоновый режим. Когда `crne-portal-hook.sh` вызывает `crne-alert.sh` сразу после остановки `sing-box` перед авторизацией в Captive Portal (когда доступ в интернет заблокирован шлюзом отеля) или когда `crne-watchdog.sh` фиксирует недоступность прокси-узлов, прямой доступ к `api.telegram.org` с роутера отсутствует (в том числе из-за блокировок ТСПУ на сетях операторов РФ). Отсутствие таймаута в `curl` приводит к зависанию вызывающего скрипта на время системного таймаута TCP/DNS, а передача неэкранированной переменной `$MESSAGE` через обычный флаг `-d` вместо `--data-urlencode` ведёт к обрезанию текста при наличии в сообщении символов `&` или пробелов.

### 4.4. Сбои парсера parse_uri в sub2sing.py на реальных коммерческих подписках
Функция `parse_uri(uri: str, idx: int)` в Приложении A (`sub2sing.py`) содержит четыре уязвимости при разборе URI из коммерческих пулов на 500+ узлов. 
Во-первых, для протокола Hysteria2 (`hy2://` / `hysteria2://`) коммерческие провайдеры широко применяют динамическую смену портов (Port Hopping), указывая диапазон портов в URI: `hy2://pass@198.51.100.1:20000-50000/?sni=example.com#Node2`. При обращении к свойству `parsed.port` стандартный модуль Python `urllib.parse` пытается преобразовать строку `'20000-50000'` в целое число и выбрасывает неперехваченное исключение `ValueError: Port could not be cast to integer value as '20000-50000'`, аварийно прерывая компиляцию всей подписки. 
Во-вторых, парсер Hysteria2 игнорирует стандартные параметры обфускации `obfs=salamander` и `obfs-password=...`, без переноса которых в объект `"obfs"` конфигурационного файла `sing-box` подключение к серверам с активным обфускатором невозможно. 
В-третьих, значения `parsed.username` в ветках `vless://` и `hy2://` не декодируются через `urllib.parse.unquote()`, из-за чего процентно-кодированные символы в паролях (например, `%3A` или `%40`) попадают в `config.json` в искажённом виде, а для VLESS не проверяется тип транспорта (`type=grpc` / `ws`), которому принудительно назначается несовместимый `"flow": "xtls-rprx-vision"`. 
В-четвёртых, при запуске `sub2sing.py --url ... --push root@10.10.1.1` без явного аргумента `--secret` скрипт генерирует новый случайный токен `secrets.token_hex(16)` и перезаписывает им существующий секрет на роутере.

### 4.5. Потеря /etc/crne-alert.conf при sysupgrade и дефект порядка бэкапов в Разделе 6.2
В Разделе 6.1 (Шаг 2) создаётся файл конфигурации Telegram-алертинга `/etc/crne-alert.conf` с правами 600, содержащий секретные ключи `TG_BOT_TOKEN` и `TG_CHAT_ID`, однако этот файл не включён в список персистентности `/etc/sysupgrade.conf` (Раздел 5.7). В результате после любого обновления прошивки OpenWrt через `sysupgrade` настройки алертинга будут утеряны. Кроме того, в Разделе 6.2 команды резервного копирования `cp /etc/config/dhcp /etc/config/dhcp.bak` и `cp /etc/config/travelmate /etc/config/travelmate.bak` ошибочно помещены в финальный блок отмены таймера после успешной проверки (после `rm -f /tmp/rollback.lock ...`), тогда как внутри самого `nohup`-таймера отката восстановление этих двух файлов отсутствует.

---

## 5. Ошибки в программе приёмочных испытаний (ATP T1–T27) и документационные противоречия (Low Severity)

Анализ программы и методики приёмочных испытаний (Раздел 7) выявил четыре несоответствия, из-за которых приёмочное тестирование завершится ложными отказами даже на полностью исправном оборудовании:

1. **Обращение к несуществующему эндпоинту /status без авторизации в Тесте T27:** В критерии успешного прохождения нагрузочного теста T27 указано: `curl -s http://10.10.1.1:9090/status` возвращает `{}` без ошибок. В реализации Clash API внутри `sing-box` (`experimental/clashapi/server.go`) эндпоинт `/status` отсутствует (корневой путь `/` возвращает `{"hello":"sing-box"}`, а для проверки доступности используются `/version`, `/configs` или `/proxies`). Кроме того, поскольку в `config.json` задан непустой `secret`, любой запрос к Clash API без заголовка `-H "Authorization: Bearer $SECRET"` возвращает ошибку `HTTP 401 Unauthorized` (что проверяется в тесте T10).
2. **Отказ в соединении (Connection refused) в Тесте T15:** Команда теста масштабирования T15 обращается к сокету `socks5h://10.10.1.1:1603` (`for i in $(seq 1 50); do curl -x socks5h://10.10.1.1:1603 ...`), однако в `config.json` (Раздел 5.2) и в `sub2sing.py` (Приложение А) входящий прокси `mixed-in` привязан исключительно к локальной петле: `"listen": "127.0.0.1", "listen_port": 1603`. Любое подключение на IP `10.10.1.1:1603` отвергается ядром с ошибкой `Connection refused`.
3. **Устаревшие ссылки на порт :5300 в заголовках и комментариях:** В заголовке Раздела 5.3 (Настройка firewall (`flow_offloading=0`), dnsmasq (`10.10.1.1#5300`)...) и в комментарии к проверке №5 внутри `crne-watchdog.sh` (dnsmasq 10.10.1.1:53 -> sing-box 10.10.1.1:5300) сохранились упоминания порта 5300 из ревизий v3.2/v4.0, хотя в v4.1 отдельный DNS-inbound на порту 5300 удалён.
4. **Несогласованность нумерации тестов и версий:** Заголовок Раздела 7 гласит (ATP — Чек-лист T1–T26), в то время как таблица включает 27 испытаний (T1–T27). В критерии теста T1 оставлена оговорка `версия sing-box >= 1.12.0 (или 1.11.x)`, противоречащая вводному предупреждению об отсутствии совместимости с 1.11.x, а в `sub2sing.py` описание `ArgumentParser` содержит устаревшую строку `"CRNE v3.2 Subscription Compiler"`.

---

## 6. Сводный реестр исправлений и эталонные патчи для Редакции v4.2

В таблице ниже сведены все 16 дефектов с указанием точного места в документе и инженерного решения.

| ID | Уровень | Компонент / Раздел | Суть дефекта в v4.1 | Инженерное исправление для v4.2 |
| :--- | :--- | :--- | :--- | :--- |
| **D1** | Critical | Приложение A (`sub2sing.py`) | `build_router_config()` объявлена с 5 аргументами, но вызывается с 6 (`sb_minor`), вызывая `TypeError`. | Удалить рудимент `sb_minor` из вызова в `main()` и очистить неиспользуемую функцию `detect_sb_minor_version()`. |
| **D2** | Critical | Приложение A (`sub2sing.py --push`) | Оператор `;` в конце `remote_cmd` возвращает код 0 при сбое проверок, а `kill` сразу после `restart` мгновенно убивает 300с таймер отката. | Разделить валидацию и активацию через `set -e`; добавить `trap "" HUP INT TERM` и `</dev/null`; гасить таймер только после проверки поднятия `tun0`. |
| **D3** | Critical | Раздел 6.1 (Runbook) | Отсутствуют пакеты `git` и `kmod-nft-queue` в `opkg install`, пропущен старт служб в финале Шага 6. | Добавить `git kmod-nft-queue` в Шаг 1 и `/etc/init.d/firewall reload && /etc/init.d/sing-box restart` в Шаг 6. |
| **D4** | High | Раздел 3, 5.2 (`auto_redirect` + `dnsmasq`) | Таблица `inet sing-box` в `prerouting` перехватывает запросы LAN к `10.10.1.1:53` мимо `dnsmasq`, ломая `.lan` и Canary-правила. | Перенести Canary-домены в `dns.rules` внутри `config.json`; перевести `dnsmasq` на порт 5353 и резолвить `.lan` через `dnsmasq` из `sing-box`. |
| **D5** | High | Раздел 3, 5.1, 5.2 (`default_mark` vs `auto_redirect`) | Документированный конфликт `auto_redirect: true` и `route.default_mark: 255` (0xff); перезапись метки на `0x2024`. | Убрать `"default_mark": 255` из `route` и задать `"auto_redirect_output_mark": "0x000000ff"` в `tun` inbound. |
| **D6** | High | Раздел 5.2 (`default_domain_resolver`) | В `sing-box 1.12+` параметр `default_domain_resolver: "dns-bootstrap"` заставляет `direct` outbound обходить `dns.rules` (Issue #3668). | Добавить действие `{"action": "resolve"}` в `route.rules` перед правилами маршрутизации и использовать `"dns-direct-ru"` для прямого резолвинга. |
| **D7** | High | Раздел 5.1, 5.4, 5.5 (Captive Portal) | `oifname { "wwan", "trm_wwan" }` не матчит физический `phy0-sta0`; `strict_route` перехватывает `curl --interface`; `awk` парсит IP DNS вместо хоста. | Расширить набор `oifname` шаблонами `"phy*-sta*"`, `"wlan*"`; исправить парсер `nslookup` (брать адрес после строки `Name:`). |
| **D8** | Medium | Раздел 5.6.2, 6.1 (`/etc/init.d/sing-box`) | Заявленное отключение `procd_set_param file` не реализовано в коде; после `sed` не вызывается перезапуск службы. | Добавить `sed -i '/procd_set_param file/d' /etc/init.d/sing-box` и перезапуск службы при модификации init-скрипта. |
| **D9** | Medium | Раздел 5.6.2 (`crne-watchdog.sh`) | Ранний `exit 1` при падении `sing-box` пропускает блок 7 (экспорт метрик Prometheus), оставляя `crne_tunnel_status 1`. | Вынести запись `crne.prom` в функцию `write_prom()`, вызываемую через `trap write_prom EXIT` при любом завершении скрипта. |
| **D10** | Medium | Раздел 5.6.1 (`crne-alert.sh`) | Вызов `curl` к `api.telegram.org` без `-m` зависает при падении туннеля/Captive Portal; отсутствие `--data-urlencode`. | Добавить `-m 5`, `--data-urlencode` и фоновый запуск с fallback-попыткой через `socks5h://127.0.0.1:1603`. |
| **D11** | Medium | Приложение A (`parse_uri` в `sub2sing.py`) | Падение с `ValueError` на диапазонах портов Hysteria2 (`20000-50000`), потеря `salamander`, отсутствие `unquote`, затирание секрета. | Безопасно парсить `netloc` (конвертировать диапазон портов в `"server_ports": ["20000:50000"]`), декодировать `unquote`, поддержать `obfs`, считывать секрет по SSH при `--push`. |
| **D12** | Medium | Раздел 5.7, 6.2 (`sysupgrade` и Rollback) | Файл `/etc/crne-alert.conf` забыт в `sysupgrade.conf`; бэкап `dhcp.bak`/`travelmate.bak` помещён после завершения отката. | Включить `/etc/crne-alert.conf` в `sysupgrade.conf`; перенести копирование и восстановление `dhcp`/`travelmate` внутрь блока транзакции. |
| **D13** | Low | Раздел 7 (Тест T27) | Проверка несуществующего эндпоинта `http://10.10.1.1:9090/status` без заголовка `Authorization`. | Заменить на `curl -s -H "Authorization: Bearer $SECRET" http://10.10.1.1:9090/version`. |
| **D14** | Low | Раздел 7 (Тест T15) | Вызов `socks5h://10.10.1.1:1603` при том, что `mixed-in` слушает только `127.0.0.1:1603`. | Заменить адрес в тесте T15 (при запуске на роутере) на `127.0.0.1:1603`. |
| **D15** | Low | Раздел 5.3, 5.6.2 (Комментарии) | Упоминание устаревшего порта `10.10.1.1:5300` / `10.10.1.1#5300` из ревизий v3.2/v4.0. | Актуализировать комментарии в соответствии со схемой `tun0` (`auto_redirect` + `hijack-dns`). |
| **D16** | Low | Раздел 7, Приложение A | Несоответствие заголовка T1-T26 (в таблице 27 тестов), упоминание 1.11.x в T1 и v3.2 в argparse. | Привести заголовки и строки версий к единому стандарту v4.2 и `sing-box >= 1.12.0`. |

### 6.1. Исправленная конфигурация /etc/sing-box/config.json и dnsmasq (Устранение дефектов D4, D5, D6)
Чтобы устранить перехват запросов `.lan` мимо `dnsmasq`, конфликт `default_mark` с `auto_redirect` и обход `dns.rules` на прямом контуре, в UCI необходимо перевести `dnsmasq` на локальный порт 5353 (`uci set dhcp.dnsmasq[0].port='5353'`), чтобы он обслуживал локальную зону `.lan` и DHCP-имена без конфликта с DNAT-правилами `auto_redirect` на порту 53. Одновременно в `/etc/sing-box/config.json` добавляется сервер `dns-local-lan` (`127.0.0.1:5353`), переносятся Canary-домены (`use-application-dns.net`, `_dns.resolver.arpa`) в `dns.rules`, задаётся `"auto_redirect_output_mark": "0x000000ff"` в `tun-in` (вместо конфликтующего `route.default_mark: 255`) и добавляется нетерминальное действие `{"action": "resolve"}` в `route.rules`:

```json
{
  "dns": {
    "servers": [
      {
        "type": "https",
        "tag": "dns-proxy-cf",
        "server": "1.1.1.1",
        "detour": "GLOBAL-AUTO"
      },
      {
        "type": "https",
        "tag": "dns-direct-ru",
        "server": "common.dot.dns.yandex.net",
        "domain_resolver": "dns-bootstrap"
      },
      {
        "type": "udp",
        "tag": "dns-local-lan",
        "server": "127.0.0.1",
        "server_port": 5353
      },
      {
        "type": "udp",
        "tag": "dns-bootstrap",
        "server": "77.88.8.8"
      }
    ],
    "rules": [
      {
        "domain": ["use-application-dns.net", "_dns.resolver.arpa"],
        "action": "predefined",
        "rcode": "NXDOMAIN"
      },
      {
        "domain_suffix": [".lan"],
        "server": "dns-local-lan"
      },
      {
        "domain_suffix": [".local", ".home.arpa", ".in-addr.arpa", ".ip6.arpa"],
        "action": "predefined",
        "rcode": "NXDOMAIN"
      },
      {
        "rule_set": ["geosite-ads"],
        "action": "predefined",
        "rcode": "NOERROR"
      },
      {
        "domain_suffix": [".ru", ".рф", ".su"],
        "server": "dns-direct-ru"
      },
      {
        "rule_set": ["geosite-ru"],
        "server": "dns-direct-ru"
      }
    ],
    "final": "dns-proxy-cf",
    "strategy": "ipv4_only"
  },
  "inbounds": [
    {
      "type": "tun",
      "tag": "tun-in",
      "interface_name": "tun0",
      "address": ["172.19.0.1/30"],
      "mtu": 1400,
      "auto_route": true,
      "auto_redirect": true,
      "auto_redirect_output_mark": "0x000000ff",
      "strict_route": true
    },
    {
      "type": "mixed",
      "tag": "mixed-in",
      "listen": "127.0.0.1",
      "listen_port": 1603
    }
  ],
  "route": {
    "auto_detect_interface": true,
    "default_domain_resolver": "dns-direct-ru",
    "rule_set": [
      {
        "tag": "geosite-ru",
        "type": "local",
        "format": "binary",
        "path": "/etc/sing-box/ruleset/geosite-category-ru.srs"
      },
      {
        "tag": "geoip-ru",
        "type": "local",
        "format": "binary",
        "path": "/etc/sing-box/ruleset/geoip-ru.srs"
      },
      {
        "tag": "geosite-ads",
        "type": "local",
        "format": "binary",
        "path": "/etc/sing-box/ruleset/geosite-category-ads-all.srs"
      }
    ],
    "rules": [
      {
        "action": "sniff"
      },
      {
        "protocol": "dns",
        "action": "hijack-dns"
      },
      {
        "ip_is_private": true,
        "outbound": "direct"
      },
      {
        "ip_cidr": ["1.1.1.1/32", "1.0.0.1/32", "8.8.8.8/32", "8.8.4.4/32", "9.9.9.9/32"],
        "port": 443,
        "action": "reject"
      },
      {
        "rule_set": ["geosite-ads"],
        "action": "reject"
      },
      {
        "action": "resolve",
        "strategy": "ipv4_only"
      },
      {
        "domain_suffix": [".ru", ".рф", ".su"],
        "outbound": "direct"
      },
      {
        "rule_set": ["geosite-ru", "geoip-ru"],
        "outbound": "direct"
      }
    ]
  }
}
```

### 6.2. Исправление окна Captive Portal в 99-crne-security.nft и 99-singbox-wan-switch (Устранение дефекта D7)
В файле `/etc/nftables.d/99-crne-security.nft` необходимо расширить список физических имён интерфейсов в правиле №2 цепочки `crne_forward_guard`, добавив шаблоны `phy0-sta0`, `phy1-sta0` и `wlan*`, используемые драйвером `mt76/mac80211` в OpenWrt 24.10+:
```nft
iifname @crne_cp_window oifname { "wwan", "trm_wwan", "phy0-sta0", "phy1-sta0", "wlan*" } ct state new,established meta l4proto { tcp, udp } th dport { 53, 80, 443 } counter accept comment "CRNE: Captive Portal Temporary Window"
```
В скрипте `/etc/hotplug.d/iface/99-singbox-wan-switch` следует извлекать реальное имя сетевого устройства `$L3_DEV` из `ubus` и исправить парсинг вывода BusyBox `nslookup` (извлекая адрес строго после строки `Name:`, чтобы исключить захват IP-адреса самого DNS-сервера):
```sh
WAN_DNS=$(ubus call "network.interface.$INTERFACE" status 2>/dev/null | jq -r '.["dns-server"][0] // empty')
L3_DEV=$(ubus call "network.interface.$INTERFACE" status 2>/dev/null | jq -r '.l3_device // empty')

if [ -n "$WAN_DNS" ]; then
    CP_IP=$(nslookup captive.apple.com "$WAN_DNS" 2>/dev/null | awk '/^Name:/{f=1; next} f && /^Address/{print $NF; exit}')
    if [ -n "$CP_IP" ]; then
        PROBE=$(curl -s -m 4 --interface "$L3_DEV" --resolve "captive.apple.com:80:$CP_IP" -L "http://captive.apple.com/hotspot-detect.html" 2>/dev/null)
    else
        PROBE=""
    fi
    if ! echo "$PROBE" | grep -q "Success"; then
        /etc/travelmate/crne-portal-hook.sh cp "$WAN_DNS"
        for _ in $(seq 1 36); do
            sleep 5
            CP_IP=$(nslookup captive.apple.com "$WAN_DNS" 2>/dev/null | awk '/^Name:/{f=1; next} f && /^Address/{print $NF; exit}')
            if [ -n "$CP_IP" ] && curl -s -m 3 --interface "$L3_DEV" --resolve "captive.apple.com:80:$CP_IP" "http://captive.apple.com/hotspot-detect.html" 2>/dev/null | grep -q "Success"; then
                /etc/travelmate/crne-portal-hook.sh connected
                break
            fi
        done
        exit 0
    fi
fi
```

### 6.3. Исправление crne-watchdog.sh и crne-alert.sh (Устранение дефектов D8, D9, D10)
В скрипте `/usr/bin/crne-alert.sh` добавляются жёсткий таймаут (`-m 5`), безопасное кодирование `--data-urlencode` и фоновый fallback через локальный SOCKS5h-прокси (`127.0.0.1:1603`), чтобы алерты доставлялись как при работающем туннеле (в условиях блокировки `api.telegram.org` на ТСПУ), так и напрямую:
```sh
#!/bin/sh
# /usr/bin/crne-alert.sh — отказоустойчивая отправка алертов в Telegram
[ -f /etc/crne-alert.conf ] && . /etc/crne-alert.conf
TG_BOT_TOKEN="${TG_BOT_TOKEN:-}"
TG_CHAT_ID="${TG_CHAT_ID:-}"
MESSAGE="$1"
SITE_TAG="${SITE_TAG:-gateway}"

[ -z "$TG_BOT_TOKEN" ] || [ -z "$TG_CHAT_ID" ] && exit 0
[ -z "$MESSAGE" ] && exit 0

API_URL="https://api.telegram.org/bot$TG_BOT_TOKEN/sendMessage"
PAYLOAD="🚨 CRNE ALERT [$SITE_TAG]: $MESSAGE"

# Попытка 1: через локальный туннель sing-box (обход блокировки Telegram на ТСПУ)
curl -s -m 5 -x socks5h://127.0.0.1:1603 "$API_URL" \
     --data-urlencode "chat_id=$TG_CHAT_ID" \
     --data-urlencode "text=$PAYLOAD" >/dev/null 2>&1

# Попытка 2: напрямую (если sing-box остановлен, а провайдер не блокирует Telegram)
curl -s -m 5 "$API_URL" \
     --data-urlencode "chat_id=$TG_CHAT_ID" \
     --data-urlencode "text=$PAYLOAD" >/dev/null 2>&1
```

В `crne-watchdog.sh` запись метрик выносится в функцию `write_prom`, вызываемую через `trap` при любом завершении. В блоке 0 добавляется удаление `procd_set_param file`:
```sh
# В блоке 0 (самовосстановление /etc/init.d/sing-box):
INIT_CHANGED=0
if grep -q "procd_set_param file" /etc/init.d/sing-box; then
    sed -i '/procd_set_param file/d' /etc/init.d/sing-box
    INIT_CHANGED=1
fi
if ! grep -q "GOMEMLIMIT" /etc/init.d/sing-box; then
    sed -i '/procd_open_instance/a \	procd_set_param env GOMEMLIMIT=75MiB' /etc/init.d/sing-box
    INIT_CHANGED=1
fi
if ! grep -q "procd_set_param respawn" /etc/init.d/sing-box; then
    sed -i '/procd_set_param env GOMEMLIMIT/a \	procd_set_param respawn ${respawn_threshold:-3600} ${respawn_timeout:-5} ${respawn_retry:-5}' /etc/init.d/sing-box
    INIT_CHANGED=1
fi
[ "$INIT_CHANGED" = "1" ] && /etc/init.d/sing-box restart
```

### 6.4. Исправленный код parse_uri и блока --push в sub2sing.py (Устранение дефектов D1, D2, D11)
Ниже приведены исправленные фрагменты `sub2sing.py`, устраняющие `TypeError`, падение на диапазонах портов Hysteria2, потерю `salamander`, маскировку ошибки SSH и преждевременное завершение таймера отката:

```python
def parse_uri(uri: str, idx: int) -> dict | None:
    uri_hash = hashlib.sha256(uri.encode("utf-8")).hexdigest()[:4]
    if uri.startswith("vless://"):
        parsed = urllib.parse.urlparse(uri)
        params = dict(urllib.parse.parse_qsl(parsed.query))
        if params.get("allowInsecure") in ("1", "true") or params.get("insecure") in ("1", "true"):
            return None
        if params.get("security") != "reality" or not params.get("pbk"):
            return None
        transport_type = params.get("type", "tcp")
        if transport_type not in ("tcp", "http", "grpc"):
            return None
        try:
            port = parsed.port or 443
        except ValueError:
            return None
        raw_tag = urllib.parse.unquote(parsed.fragment).strip() or "vless"
        node = {
            "type": "vless",
            "tag": f"{raw_tag}-{idx}-{uri_hash}",
            "server": parsed.hostname,
            "server_port": port,
            "uuid": urllib.parse.unquote(parsed.username or ""),
            "tls": {
                "enabled": True,
                "server_name": params.get("sni", ""),
                "utls": {"enabled": True, "fingerprint": params.get("fp", "chrome")},
                "reality": {
                    "enabled": True,
                    "public_key": params["pbk"],
                    "short_id": params.get("sid", "")
                }
            }
        }
        if transport_type == "tcp":
            node["flow"] = params.get("flow", "xtls-rprx-vision")
        elif transport_type == "grpc" and params.get("serviceName"):
            node["transport"] = {"type": "grpc", "service_name": params["serviceName"]}
        return node

    elif uri.startswith(("hysteria2://", "hy2://")):
        parsed = urllib.parse.urlparse(uri)
        params = dict(urllib.parse.parse_qsl(parsed.query))
        if params.get("insecure") in ("1", "true") or params.get("allowInsecure") in ("1", "true"):
            return None
        
        # Безопасное извлечение хоста и порта / диапазона портов (Port Hopping)
        netloc = parsed.netloc
        if "@" in netloc:
            netloc = netloc.split("@", 1)[1]
        
        if not netloc:
            return None
            
        if ":" in netloc:
            host, port_str = netloc.rsplit(":", 1)
        else:
            host = netloc
            port_str = "443"
            
        node = {
            "type": "hysteria2",
            "tag": f"{urllib.parse.unquote(parsed.fragment).strip() or 'hy2'}-{idx}-{uri_hash}",
            "server": host,
            "password": urllib.parse.unquote(parsed.username or params.get("auth", "")),
            "tls": {
                "enabled": True,
                "server_name": params.get("sni", host)
            }
        }
        
        if "-" in port_str:
            # Конвертация формата URI (20000-50000) в формат sing-box 1.12+ (20000:50000)
            port_range = port_str.replace("-", ":")
            node["server_ports"] = [port_range]
        else:
            node["server_port"] = int(port_str) if port_str.isdigit() else 443
            
        if params.get("obfs") == "salamander" and params.get("obfs-password"):
            node["obfs"] = {"type": "salamander", "password": params["obfs-password"]}
            
        return node
    return None
```

И исправленный блок вызова `build_router_config` и атомарной активации `--push` внутри функции `main()`:

```python
    # Если задан --push без явного --secret, сначала пытаемся считать текущий секрет с роутера:
    if args.push and not args.secret:
        try:
            out = subprocess.check_output(
                ["ssh", args.push, "jq -r '.experimental.clash_api.secret // empty' /etc/sing-box/config.json"],
                text=True, timeout=5
            )
            if out.strip() and "REPLACE_WITH_GENERATED_SECRET" not in out:
                secret = out.strip()
        except Exception:
            pass
    secret = secret or secrets.token_hex(16)

    # Вызов строго с 5 аргументами (устранение TypeError)
    cfg = build_router_config(standard_top, lts_nodes, extra_manual, args.tolerance, secret)
    
    with open(args.out, "w", encoding="utf-8") as f:
        json.dump(cfg, f, ensure_ascii=False, indent=2)

    if args.push:
        subprocess.run(["scp", args.out, f"{args.push}:/tmp/config.json.new"], check=True)
        remote_cmd = (
            "set -e; "
            # 1. Жёсткая валидация до модификации рабочей конфигурации
            "jq empty /tmp/config.json.new; "
            "sing-box check -c /tmp/config.json.new; "
            "fw4 check; "
            # 2. Создание резервной копии и взведение отвязанного от терминала 300с таймера отката
            "cp /etc/sing-box/config.json /etc/sing-box/config.json.bak; "
            "touch /tmp/rollback.lock; "
            "nohup sh -c 'trap \"\" HUP INT TERM; sleep 300; [ -f /tmp/rollback.lock ] && { "
            "mv /etc/sing-box/config.json.bak /etc/sing-box/config.json 2>/dev/null; "
            "/etc/init.d/sing-box restart; "
            "rm -f /tmp/rollback.lock; "
            "logger -t CRNE-ROLLBACK \"auto-rollback after 300s (push unconfirmed)\"; "
            "}' >/dev/null 2>&1 </dev/null & "
            "RB_PID=$!; "
            # 3. Атомарная замена и перезапуск
            "mv /tmp/config.json.new /etc/sing-box/config.json; "
            "chmod 600 /etc/sing-box/config.json; "
            "/etc/init.d/sing-box restart; "
            # 4. Ожидание фактической инициализации tun0 (4 секунды) перед снятием блокировки отката
            "sleep 4; "
            "ip link show tun0 >/dev/null 2>&1 && [ -n \"$(pidof -s sing-box)\" ]; "
            "rm -f /tmp/rollback.lock /etc/sing-box/config.json.bak; "
            "kill $RB_PID 2>/dev/null || true; "
            # 5. Опциональный git-коммит (не ломает деплой, если git не установлен или нет изменений)
            "if command -v git >/dev/null 2>&1 && [ -d /etc/sing-box/.git ]; then "
            "  cd /etc/sing-box && git add config.json && git commit -m \"sub2sing push $(date +%Y%m%d-%H%M)\" >/dev/null 2>&1 || true; "
            "fi"
        )
        subprocess.run(["ssh", args.push, remote_cmd], check=True)
```

---
**Works cited**
1. safe-network-system-spec-v4.md
2. https://github.com/SagerNet/sing-box/issues/3705
3. Tun - sing-box, https://sing-box.sagernet.org/configuration/inbound/tun/
4. default_domain_resolver 使 DNS rules 分流失效 #3668 - GitHub, https://github.com/SagerNet/sing-box/issues/3668
5. https://github.com/SagerNet/sing-box/issues/3584
6. https://github.com/SagerNet/sing-box/issues/3560
7. Pre-match drop action breaks on 1.14.0-beta.13 #4433 - GitHub, https://github.com/SagerNet/sing-box/issues/4433
8. Linux auto_redirect: local tailscaled UDP enters TUN despite, https://github.com/SagerNet/sing-box/issues/4505
9. packages/net/sing-box/Makefile at master - GitHub, https://github.com/openwrt/packages/blob/master/net/sing-box/Makefile
10. [OpenWrt Wiki] Firewall configuration /etc/config/firewall, https://openwrt.org/docs/guide-user/firewall/firewall_configuration
11. sing-box - TUN 场景下错误劫持 53 端口的流量 - GitHub, https://github.com/SagerNet/sing-box/issues/3487
12. TUN — sing-box | Core Tutorial, https://core-tutorial.argsment.com/singbox/tun
13. No supported way to route dial-side domain resolution through DNS, https://github.com/SagerNet/sing-box/issues/4444
14. 并导致后续的 dns.rule.action 不会触发执行·Issue #3596·SagerNet, https://github.com/SagerNet/sing-box/issues/3596
15. Using a USB wireless card with gl-inet firmware Part 1, https://forum.gl-inet.com/t/using-a-usb-wireless-card-with-gl-inet-firmware-part-1-the-honeymoon-period/11313
16. Interface option network.wan6.sourcefilter='0' is ignored #14954, https://github.com/openwrt/openwrt/issues/14954
17. firewall4: design shortcomings in fw4 custom filter chains · Issue #9981 · openwrt/openwrt, https://github.com/openwrt/openwrt/issues/9981
18. nftables Firewall GUI: Top Tools for Linux - InfoSec Monkey, https://infosecmonkey.com/nftables-firewall-the-modern-linux-packet-filter-and-every-gui-worth-using-to-manage-it/
19. 配置 openwrt+sing-box 翻墙 - 美国的华莱士, https://hualaishi.me/%E9%85%8D%E7%BD%AEopenwrtsing-box%E7%BF%BB%E5%A2%99/
20. unknown_url
21. clashapi package - github.com/kelleygo/sing-box/experimental, https://pkg.go.dev/github.com/kelleygo/sing-box/experimental/clashapi
22. clashapi package - github.com/Rigby-Foundation/NuggetVPN, https://pkg.go.dev/github.com/Rigby-Foundation/NuggetVPN/third_party/sing-box-official/experimental/clashapi
23. [Security] Zip Slip Path Traversal in external_ui_download #4117, https://github.com/SagerNet/sing-box/issues/4117
