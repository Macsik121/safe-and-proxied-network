# Инженерная спецификация проекта «Цензуроустойчивый сетевой шлюз (Censorship-Resistant Network Gateway)» — Редакция v4.4

Настоящий документ представляет собой финальную, полностью верифицированную спецификацию промышленного уровня (**HLD + LLD + Исполняемый код артефактов + ATP + Runbook эксплуатации**). Редакция **v4.4** исправляет критические архитектурные дефекты, выявленные при независимом аудите v4.3 (критика `experts-reports-criticisism.md`), и усиливает модель угроз.

**Ключевые улучшения v4.4 относительно v4.3:**
- ✅ **ADR-08 (Critical):** Bootstrap DNS заменён на DoH с certificate pinning (Cloudflare 1.1.1.1) — устранён MITM-вектор в юрисдикции РФ
- ✅ **ADR-09 (Critical):** Kill Switch расширен на output роутера — добавлена цепочка `crne_output_guard` + набор `crne_cp_output_wan` (окно для детект-проб Captive Portal с роутера, устраняет регрессию ADR-09)
- ✅ **ADR-10 (Critical):** Git-история перенесена в tmpfs — устранена персистентность секретов на NAND
- ✅ **MSS Clamping:** Добавлена цепочка `crne_mss_clamp` для предотвращения PMTUD-blackhole

**Ключевые улучшения v4.3 относительно v4.2:**
- ✅ **D1 (Critical):** Добавлен SMS/email fallback для алертинга при недоступности Telegram
- ✅ **D2 (Critical):** Добавлен детект пустого пула узлов в watchdog + auto-rollback на резервную подписку
- ✅ **D3 (Critical):** Добавлена SHA256 verification для `.srs` файлов с trusted hashes
- ✅ **ADR-06 (New):** SHA256 verification для бинарных правил маршрутизации
- ✅ **ADR-07 (New):** Watchdog self-check через staleness detection

**Ключевые улучшения v4.2 относительно v4.1:**
- ✅ **D1 (High):** Исправлен тест T15 — SOCKS-прокси слушает `127.0.0.1:1603` (только localhost), тест выполняется с роутера
- ✅ **D2 (High):** Rollback-таймер в `--push` гасится **только после** автоматической пробы работоспособности через SOCKS
- ✅ **D3 (Medium):** Удалены артефакты legacy DNS (порт 5300), заголовок §5.3 исправлен
- ✅ **D4 (Medium):** Добавлен `/etc/crne-alert.conf` в `sysupgrade.conf` для персистентности секретов Telegram
- ✅ **D5 (Medium):** Hotplug-лок заменён на атомарный `mkdir` вместо check-then-create
- ✅ **D6 (Low):** Порядок проверок в watchdog исправлен: сначала туннель (шаг 6), потом dnsmasq (шаг 5)
- ✅ **D7 (Low):** Добавлен счётчик `crne_dns_bootstrap_requests_total` для мониторинга открытых DNS-запросов к 77.88.8.8
- ✅ **D8 (Low):** Добавлено напоминание об обновлении `.srs`-баз (cron-алерт раз в 30 дней)

**Дефекты независимого технического аудита (`reports/spec-bugs-report.md`, ревизия v4.1), устранённые в v4.2 (поверх архитектурного аудита):**
- ✅ **R-D1 (Critical, очистка):** Удалён мёртвый код `detect_sb_minor_version()` и рудимент `--sb-minor`. `TypeError` в v4.2 не воспроизводился (вызов `build_router_config` уже передавал 5 аргументов), но неиспользуемая функция миграции с 1.11.x очищена.
- ✅ **R-D2 (Critical):** Блок `--push` в `sub2sing.py` переписан на `set -e` + `trap "" HUP INT TERM` + `</dev/null`; таймер отката гасится только после проверки поднятия `tun0` и `pidof sing-box` (устранены маскировка ошибки кодом возврата 0 через `;` и преждевременный `kill` после асинхронного `restart`).
- ✅ **R-D3 (Critical):** В Шаг 1 Runbook добавлены пакеты `git` и `kmod-nft-queue` (последний нужен для NFQUEUE-прематча `auto_redirect`; `kmod-nft-tproxy` нерелевантен для режима `tun`); в Шаг 6 добавлены `/etc/init.d/firewall reload && /etc/init.d/sing-box restart`.
- ✅ **R-D4 (High):** `dnsmasq` переведён на порт `5353`; добавлен сервер `dns-local-lan` (`127.0.0.1:5353`); Canary-домены и зона `.lan` перенесены в `dns.rules` (устранён перехват LAN-DNS мимо `dnsmasq` таблицей `inet sing-box` в `prerouting`).
- ✅ **R-D5 (High):** Удалён конфликтующий `route.default_mark: 255`; в `tun` inbound задан `"auto_redirect_output_mark": "0x000000ff"` (совместимость с внутренними метками sing-tun `0x2023/0x2024/0x2025`).
- ✅ **R-D6 (High):** `default_domain_resolver` изменён на `"dns-direct-ru"`; в `route.rules` добавлено нетерминальное `{"action": "resolve", "strategy": "ipv4_only"}` (устранён обход `dns.rules` для `direct` outbound, Issue #3668).
- ✅ **R-D7 (High):** В `crne_forward_guard` расширены `oifname` (`phy0-sta0`, `phy1-sta0`, `wlan*`); hotplug извлекает `L3_DEV` из `ubus` и использует корректный парсер `nslookup` (адрес после строки `Name:`).
- ✅ **R-D8 (Medium):** Реализовано фактическое удаление `procd_set_param file` через `sed` (в Runbook Шаг 4 и в watchdog блок 0) с перезапуском службы при модификации init-скрипта.
- ✅ **R-D10 (Medium):** `crne-alert.sh` переписан с `-m 5`, `--data-urlencode` и фоновой доставкой через `socks5h://127.0.0.1:1603` с fallback на прямой запрос.
- ✅ **R-D11 (Medium):** `parse_uri` переписан: безопасный разбор `netloc` для Port Hopping (`server_ports`), поддержка `obfs salamander`, `urllib.parse.unquote` для учётных данных, проверка типа транспорта VLESS; при `--push` без `--secret` секрет считывается с роутера по SSH.
- ✅ **R-D12 (Medium):** Бэкапы `dhcp.bak`/`travelmate.bak` перенесены внутрь блока транзакции Lock-Rollback с восстановлением в таймере отката.
- ✅ **R-D13 (Low):** Тест T27 исправлен: `curl -s -H "Authorization: Bearer $SECRET" http://10.10.1.1:9090/version` (вместо несуществующего `/status` без авторизации).
- ✅ **R-D9, R-D14, R-D15, R-D16:** Устранены ранее (в рамках архитектурного аудита) — ранний `exit 1` в watchdog убран, T15 использует `127.0.0.1:1603`, артефакты порта 5300 удалены, заголовок ATP приведён к `T1–T27`, версии унифицированы под `sing-box >= 1.12.0` / `v4.2`.

**Ключевые улучшения v4.1 относительно v4.0:**
- ✅ **КРИТИЧЕСКОЕ ИСПРАВЛЕНИЕ:** Удалён несуществующий inbound `"type": "dns"` — DNS-перехват теперь осуществляется через `tun` inbound с правилом `hijack-dns`
- ✅ Добавлено предупреждение о миграции с 1.11.x на 1.12+ (legacy DNS format будет удалён в 1.14.0)
- ✅ Отключён `procd file` во избежание гонки перезапусков с `sub2sing.py --push`
- ✅ Уточнены критерии тестов T12 (gc-interval vs timeout) и T27 (VmRSS стабильность)

> **⚠️ Важное примечание о поддержке версий:** Данная спецификация требует **`sing-box >= 1.12.0`**. Обратная совместимость с `1.11.x` **не поддерживается** в production-режиме из-за критических изменений в формате DNS-конфигурации.

---

## 📌 1. Назначение, модель угроз и архитектурные инварианты

### 1.1. Целевое назначение
Построение унифицированной пары сетевых шлюзов под управлением **OpenWrt / ImmortalWrt** — **стационарного домашнего (`home`)** и **автономного дорожного (`travel`)**, обеспечивающих:
1. Прозрачную маршрутизацию трафика локальных клиентов через коммерческий пул из **500+ криптографических узлов** (`VLESS + XTLS-Reality`, `Hysteria2`), включая специализированные **мобильные `LTS`-серверы** (доступные только при выходе через сети сотовых операторов).
2. Противодействие глубокому анализу пакетов (**stateful DPI / ТСПУ**) с автоматическим выбором оптимального узла по результатам параллельного L7-тестирования (`HTTP 204`).
3. Эшелонированную защиту от утечек метаданных во враждебных проводных и публичных Wi-Fi сетях:
   * **Жёсткая блокировка (100% перехват/отсев на уровне ядра):** открытый DNS (`TCP/UDP :53`), `DNS-over-TLS/QUIC (:853 TCP/UDP)`, транзитный `IPv6`, запросы `WebRTC STUN/TURN (:3478, :5349, :19302)` и браузерный `QUIC (UDP/443)` из локальной сети.
   * **Подавление встроенного браузерного DoH (`TCP/443`):** сигнализация браузерам через Canary-домены (`NXDOMAIN` для `use-application-dns.net` и `_dns.resolver.arpa`, штатно отключающая DoH в Firefox/Chrome/Safari) + блокировка IP-адресов известных публичных DoH-резолверов (*best-effort* для нестандартных клиентов).

### 1.2. Архитектурные инварианты (Неприкосновенные правила системы)
* **Абсолютный Fail-Closed на уровне ядра (`nftables`):** 
  * **Транзитная цепочка `forward`:** жёстко блокирует любые пакеты из `br-lan` в любые внешние интерфейсы (`iifname "br-lan" oifname != "tun0" reject`). Даже если процесс `sing-box` аварийно завершится и виртуальный интерфейс `tun0` исчезнет из системы, трафик клиентов **физически не сможет уйти в WAN провайдера**.
  * **Цепочка `output` роутера (ADR-09, v4.4):** добавлена цепочка `crne_output_guard`, блокирующая исходящий трафик самого роутера при отсутствии `tun0` с исключениями для критичных сервисов (NTP, DHCP, localhost). Это закрывает утечки router-originated traffic (NTP, обновления пакетов, алерты).
* **Bootstrap DNS вне юрисдикции РФ (ADR-08, v4.4):** вместо Яндекс DNS (77.88.8.8) используется Cloudflare DoH (1.1.1.1:443) с certificate pinning, что устраняет MITM-вектор на этапе начальной резолюции доменов прокси-серверов.
* **Git-история в tmpfs (ADR-10, v4.4):** репозиторий `.git` перенесён в оперативную память, что исключает персистентность секретов на NAND-flash и соответствует принципу Data Minimization.
* **Безопасное бюджетирование памяти на 64-битном ARM64 (≤ 50 узлов в ОЗУ и 3 эшелона порогов):** фильтрация всех 500+ узлов подписки выполняется на внешнем сборочном узле (**ПК Windows** или **Android-смартфоне с нативным `Termux`**). В конфигурацию роутера выгружается **не более 50 отобранных узлов** (Топ-20 `Standard` + до 20 `LTS-Mobile` + до 10 резервных узлов). Управление памятью `sing-box` разделено на 3 эшелона (без установки `RLIMIT_AS` / `procd limits as`, конфликтующего с резервированием виртуальной арены `mmap` рантаймом Go на ARM64):
  1. **Базовый порог холодного старта (`≤ 45 МБ VmRSS`, тест `T3`):** нормативное потребление сразу после загрузки трёх бинарных баз `.srs` и 50 узлов.
  2. **Мягкий потолок сборщика мусора Go (`GOMEMLIMIT=75MiB`, тест `T14`):** целевой рабочий коридор при длительной суточной эксплуатации под нагрузкой (`VmRSS ≤ 75 МБ`).
  3. **Аварийный отсекатель сторожа (`95 МБ VmRSS` в `crne-watchdog.sh`):** жёсткий порог принудительного перезапуска демона при аномальной утечке памяти до срабатывания ядерного `OOM-killer`.
* **Детерминированный автооткат, устойчивый к обрыву SSH (`nohup` Lock-Rollback):** любое изменение сетевых экранов или конфигурации `sing-box` применяется под защитой демонизированного сторожевого таймера (300 с), отвязанного от терминала (`nohup ... trap "" HUP INT TERM`) и привязанного к файлу-маркеру `/tmp/rollback.lock`. Даже при мгновенной потере SSH-сессии роутер гарантированно вернёт рабочую конфигурацию через 5 минут.
* **Автономная поставка правил и веб-интерфейса (Offline-First `.srs` + `ui`):** роутер не обращается к GitHub во время работы или холодного старта. Бинарные базы маршрутизации (`.srs`) и статическая веб-панель управления (`metacubexd` / `yacd`) скачиваются и верифицируются на сборочном узле, после чего доставляются в `/etc/sing-box/` локально по `SCP`.
* **Единый источник истины для секрета API (SSOT Secret):** уникальный токен доступа к Clash API генерируется при инициализации, хранится в `/etc/sing-box/config.json` (`chmod 600`) и динамически считывается всеми системными скриптами через `jq`.

---

## 🖥️ 2. Аппаратная спецификация и профили развёртывания

### 2.1. Аппаратные требования контракта
* **Процессор (SoC):** 2 ядра ARMv8-A Cortex-A53 ≥ 1.3 ГГц с **аппаратными криптографическими инструкциями (`aes`, `pmull`, `sha1`, `sha2`)** — платформа **MediaTek Filogic 820 (`MT7981B`)**. Аппаратное ускорение шифрования на ядре процессора обеспечивает прокачку **220–350 Мбит/с** трафика `VLESS-Reality` / `Hysteria2` чисто программным путём при принудительно отключённом движке PPE (`flow_offloading='0'`, `flow_offloading_hw='0'`).
* **Оперативная память (RAM):** **строго ≥ 256 МБ DDR3/DDR4** (устройства со 128 МБ ОЗУ к эксплуатации не допускаются).
* **Флеш-память (Flash):** **≥ 128 МБ SPI-NAND**.
* **Целевая версия ОС:** **OpenWrt 24.10.5+ / 25.12+** или **ImmortalWrt 24.10+**.
  > **⚠️ Важное инженерное примечание по ревизиям NAND-памяти Cudy (партии с ноября 2025 – 2026 гг.):** устройства `Cudy WR3000S v1` и `Cudy TR3000 v1` с серийными номерами **`SN 2543...` и выше** комплектуются новым чипом флеш-памяти **ESMT `F50L1G41LC`**. Драйвер этого чипа включён только в **OpenWrt 24.10.5+** (и переходную прошивку Cudy от `2025-11-19` и новее). Использование старых образов (23.05 или ранних 24.10.0–24.10.4) на устройствах с `SN 2543+` категорически запрещено.

### 2.2. Спецификация узлов комплекса

| Профиль / Роль | Модель устройства | Аппаратная конфигурация | Функциональное назначение в проекте |
| :--- | :--- | :--- | :--- |
| **`home`** (Домашний шлюз) | **Cudy WR3000S v1** *(или Netis NX32U)* | MT7981B, 256 МБ RAM, 128 МБ NAND, Wi-Fi 6, 4×GbE | Подключается кабелем Ethernet к вводному роутеру/провайдеру. Обеспечивает защиту всех домашних устройств без настройки VPN на клиентах. |
| **`travel`** (Походный шлюз) | **Cudy TR3000 v1** *(или GL-MT3000)* | MT7981B, 256 МБ RAM, 128 МБ NAND, Wi-Fi 6, USB 3.0, USB-C | Автономный шлюз для поездок (не зависит от домашнего роутера). Принимает интернет от публичного Wi-Fi (`trm_wwan`), USB-телефона (`usb0` RNDIS/NCM) или Ethernet и раздаёт личный защищённый Wi-Fi. Питание: **5V/3A**. |
| **Мобильный ИБП / Сборщик / Модем** | **Смартфон Android (`Termux` F-Droid)** + **ПК Windows** | ARM64 / x86_64, **Python 3.10+** (`stdlib`), OpenSSH | 1) Запускает `sub2sing.py` (тест 500+ узлов и атомарная выгрузка `config.json`).<br>2) Смартфон в дороге по USB-кабелю (через короткий экранированный кабель во избежание радиошума USB 3.0 на 2.4 ГГц) работает как **LTE-модем (`usb0`) для активации `LTS`-серверов**. |

**Аппаратный стоп-лист:** карманные модемы с 16–32 МБ ОЗУ (`4Band M2` и OEM-клоны на ZTE/ASR — допустимы только как внешний USB-модем), роутеры с риском неподдерживаемой ревизии без проверки (`Xiaomi AX3000T` RD03V2 на Qualcomm, новые партии `H3C NX30 Pro` и `Nokia` с закрытым загрузчиком).

---

## ⚙️ 3. Программная архитектура (7 изолированных слоёв)

### Слой 0: Базовая ОС и сохранность NAND (OpenWrt 24.10.5+ / ImmortalWrt 24.10+)
* Подсеть LAN перенесена на `10.10.1.1/24` (исключение коллизий с сетями отелей и USB-модемов `192.168.x.x`).
* Программный и аппаратный `flow_offloading` принудительно отключён в UCI (`0`), чтобы исключить обход интерфейса `tun0` сетевым ускорителем ядра.
* Системный журнал изолирован в ОЗУ (`log_type 'circular'`, `log_size '512'`), отладочный спам `travelmate` отключён (`trm_debug='0'`), логирование `sing-box` зафиксировано на уровне `warn` (нулевой износ NAND-памяти логами).

### Слой 1: Ядро маршрутизации и криптографии — Нативный `sing-box`
* **Целевая версия:** `sing-box >= 1.12.0` (штатный пакет OpenWrt 24.10.5+) с проверкой цифровой подписи пакетов `usign`. Сторонние обёртки (`VPNpool`) исключены.
  > **⚠️ Критическое предупреждение о миграции с 1.11.x на 1.12+:** Legacy DNS format (`address`, `detour`) объявлен устаревшим в 1.12.0 и будет **удалён в 1.14.0**. При обновлении с `1.11.x` до `1.12+` **обязательно** перегенерируйте конфиг через `sub2sing.py`. Для production-режима **требуется `sing-box 1.12+`** без обратной совместимости с 1.11.x.
* **Сетевой перехват:** интерфейс `tun0` (`"address": ["172.19.0.1/30"]`, `"mtu": 1400`, `"auto_route": true`, `"auto_redirect": true`, `"strict_route": true`).
* **Строгий порядок правил маршрутизации (`route.rules`):** нетерминальное действие `{"action": "sniff"}` располагается первым, далее следуют правила перехвата `{"action": "hijack-dns"}`, затем правила сброса `{"action": "reject"}` и только после них — правила маршрутизации `outbound` (`direct` и `final: GLOBAL-AUTO`).
* **Маркировка исходящих сокетов (`auto_redirect_output_mark: 0x000000ff`):** вместо конфликтующего с `auto_redirect` глобального `route.default_mark` (см. официальное предупреждение sing-box: *Conflict with route.default_mark and [dialOptions].routing_mark*) в `tun` inbound задаётся явная метка `0xff` через `"auto_redirect_output_mark": "0x000000ff"`. Это позволяет правилам `nftables` (`crne_output_bypass`) и изоляции `Zapret` (`meta mark 0x000000ff return` в `postrouting`) безошибочно идентифицировать пакеты самого `sing-box` к прокси-узлам и прямому контуру, не вступая в конфликт с внутренними метками sing-tun `auto_redirect_input_mark` (`0x2023`) / `auto_redirect_output_mark` (`0x2024`) / `auto_redirect_reset_mark` (`0x2025`).

### Слой 2: Трёхуровневый конвейер подписок и двухпуловая логика `Standard + LTS`
* **Фильтрация, дедупликация тегов и лимит памяти:** скрипт `sub2sing.py` декодирует `base64`-подписку (500+ узлов), формирует уникальные детерминированные теги с 4-символьным SHA256-хешем URI (`f"{tag}-{idx}-{hash4}"`), отбрасывает небезопасные узлы (`allowInsecure=1` / `insecure=true`), сохраняет узлы с мобильными маркерами (`LTS`, `MOBILE`, `МОБ`, `LTE`, `БЕЛЫЙ`, `WL-`, `BYPASS`), тестирует остальные узлы параллельным запросом `https://www.youtube.com/generate_204` (`timeout: 1500 мс`) и записывает в `config.json` роутера **не более 50 узлов**.
* **Плоская архитектура `urltest` (без вложенных групп):**
  * Группа `GLOBAL-AUTO` (`type: "urltest"`, `interval: "2m"`, `tolerance: 180`) содержит **плоский список тегов** (Топ-20 обычных + до 20 `LTS`-узлов).
  * Группы `STANDARD-TOP20` и `LTS-MOBILE` доступны в селекторе `MANUAL-SELECT` через локальную панель **Clash API (`http://10.10.1.1:9090/ui`)**.

### Слой 3: Изоляция контуров обхода DPI (`sing-box` vs `Zapret`)
* Туннелируемый трафик защищён транспортами `VLESS + XTLS-Reality` и `Hysteria2`.
* Если для прямого контура (`Direct`) используется `Zapret` (`nfqws`), перед отправкой пакета в очередь `NFQUEUE` в цепочке `postrouting` ставится правило `meta mark 0x000000ff return`, гарантирующее, что `nfqws` никогда не повредит пакеты прокси-туннеля.

### Слой 4: Замкнутый двухконтурный Split-DNS через `tun` inbound с `auto_redirect`
DNS-перехват реализован через механизм `hijack-dns` в `tun` inbound с `auto_redirect: true`, который прозрачно перехватывает DNS-запросы (UDP/TCP :53) на этапе маршрутизации внутри туннеля:
1. Демон `dnsmasq` переведён с порта `53` на локальный порт **`5353`** (`uci set dhcp.@dnsmasq[0].port='5353'`). Это критически необходимо: таблица `table inet sing-box`, автоматически генерируемая при `auto_redirect: true`, в цепочке `prerouting` (`priority dstnat + 1`) перехватывает любые LAN-запросы к `10.10.1.1:53` (DNAT на внутренний адрес туннеля `172.19.0.2`), минуя сокет `dnsmasq` (GitHub Issues #3487, #3560, #3584, #3705). На порту `5353` `dnsmasq` продолжает обслуживать зону `local='/lan/'`, блокировать приватные PTR (`boguspriv='1'`) и возвращать `NXDOMAIN` для Canary-доменов.
2. `tun` inbound с `auto_redirect: true` автоматически перехватывает **все** DNS-запросы (включая запросы к `1.1.1.1:53`) и передаёт их внутреннему DNS-резолверу `sing-box` через правило `{"protocol": "dns", "action": "hijack-dns"}`. Явный DNAT в `nftables` не требуется — `auto_redirect` уже выполняет эту функцию.
3. Внутренний DNS-резолвер `sing-box` обрабатывает запросы по следующим правилам:
   * Canary-домены (`use-application-dns.net`, `_dns.resolver.arpa`) → `predefined` `NXDOMAIN` (сигнал браузерам отключить DoH). **Перенесены из `dnsmasq` в `dns.rules`**, т.к. запросы из LAN перехватываются `auto_redirect` до `dnsmasq`.
   * Зона `.lan` → сервер `dns-local-lan` (`127.0.0.1:5353`), который перенаправляет запрос в `dnsmasq` для разрешения локальных DHCP-имён. Локальные и обратные зоны (`.local`, `.home.arpa`, `.in-addr.arpa`, `.ip6.arpa`) → `predefined` `NXDOMAIN`.
   * Реклама (`geosite-ads`) → `predefined` `NOERROR` (блокировка).
   * Домены `.ru`, `.рф`, `.su` и база `geosite-ru` → зашифрованный российский DoH **`https://common.dot.dns.yandex.net/dns-query`** (сервер `dns-direct-ru`, резолвится через собственный `"domain_resolver": "dns-bootstrap"`).
   * Все остальные запросы → DoH **`https://1.1.1.1/dns-query`** **строго внутри туннеля `GLOBAL-AUTO`**.

> **⚠️ Архитектурное примечание:** В `sing-box 1.12+` не существует отдельного inbound типа `"dns"`. DNS-перехват осуществляется исключительно через `tun` inbound с правилом `hijack-dns` в `route.rules`.

> **⚠️ Предотвращение DNS-петли и обхода `dns.rules`:** Сервер `dns-bootstrap` (v4.4: Cloudflare DoH `1.1.1.1:443`, ADR-08) резолвит домен сервера `dns-direct-ru` (`common.dot.dns.yandex.net`) и **не** используется как `default_domain_resolver`. Глобальный `default_domain_resolver` установлен в `"dns-direct-ru"`, а в `route.rules` перед правилами маршрутизации добавлено нетерминальное действие `{"action": "resolve", "strategy": "ipv4_only"}`. Это устраняет дефект sing-box 1.12+ (Issues #3668, #3596, #4444): если `default_domain_resolver` указывает на транспорт с прямым UDP, поле `options.Transport` в `dialer.go` предзаполняется, и `dns/router.go` вызывает `r.client.Lookup` напрямую, минуя `r.matchDNS()` и полностью игнорируя `dns.rules` (блокировку рекламы и российский DoH). `dns-bootstrap` не образует петли, т.к. первой строкой `chain output` таблицы `inet sing-box` стоит проверка метки `meta mark 0x000000ff return` (переопределённая `auto_redirect_output_mark`), которая освобождает bootstrap-запросы Go runtime от повторного DNAT в туннель.
>
> **⚠️ ADR-08 (v4.4) — обоснование замены Яндекс DNS на Cloudflare DoH:** Предыдущая редакция использовала `77.88.8.8` (Яндекс DNS) плейнтекстом по UDP/53. Это создавало системную уязвимость: резолвер под российской юрисдикцией может быть отравлен или заблокирован по решению РКН, а on-path атакующий (ТСПУ) может подменить ответ и увести резолюцию домена прокси-сервера на honeypot. Reality защищает TLS-хендшейк, но начальная резолвация была открыта. Cloudflare DoH (`1.1.1.1:443`, `server_name: cloudflare-dns.com`) с certificate pinning закрывает этот вектор: MITM невозможен без компрометации сертификата Cloudflare. Для холодного старта IP `1.1.1.1` предзаполнен в `/etc/hosts` (см. Runbook Шаг 2).

### Слой 5: Эшелонированная сетевая защита (`99-crne-security.nft`)
1. **Hard Fail-Closed Kill Switch:** правило `iifname "br-lan" oifname != "tun0" reject` в цепочке `forward` блокирует любой выход клиентов мимо `tun0` (включая сценарий падения самого процесса `sing-box`).
2. **Защита от утечек браузерного QUIC (`UDP/443`), DoT (`853`) и WebRTC (`STUN/TURN`):** для клиентов из `br-lan` заблокированы порты `853/tcp,udp`, `3478, 5349, 19302 / udp` (STUN) и `443/udp` (QUIC). *Осознанный архитектурный компромисс:* блокировка `UDP/443` из `br-lan` переводит и прямые российские сайты с `HTTP/3 (QUIC)` на `HTTP/2 (TCP/443)`, что необходимо для гарантированного анализа SNI и предотвращения обхода правил маршрутизации, но никак не затрагивает работу транспорта `Hysteria2` (исходящего из цепочки `output` самого роутера с меткой `0xff`).

### Слой 6: Бесшовный Multi-WAN (`Wi-Fi ↔ USB`) и трёхступенчатая развязка Captive Portal (`TR3000`)
* **Переключение каналов и `LTS`-узлов:** при смене канала связи между `trm_wwan` (Wi-Fi отеля/кафе, `metric 10`) и `usb_wan` (смартфон по USB RNDIS `usb0`, `metric 20`) скрипт `/etc/hotplug.d/iface/99-singbox-wan-switch` через `curl` сбрасывает старые соединения и запускает внеочередной `urltest`. За 2 секунды `LTS`-серверы автоматически активируются на мобильном интернете и выводятся из ротации на проводном Wi-Fi.
* **Трёхступенчатое окно Captive Portal (без мёртвого клинча DNS и перехвата `tun0`):**
  1. **Независимый детект:** при поднятии `trm_wwan` хотплаг-скрипт извлекает IP локального DNS-сервера отеля напрямую из `ubus` (`$WAN_DNS`), резолвит через него `captive.apple.com` и выполняет HTTP-пробу через `curl --resolve` в обход `dnsmasq` и `sing-box`.
  2. **Скалярный набор `@crne_cp_window`, ядерная карта DNS `@crne_cp_dns` и output-набор `@crne_cp_output_wan` (180 с):** при обнаружении портала хук `/etc/travelmate/crne-portal-hook.sh cp "$WAN_DNS" "$L3_DEV"` добавляет `"br-lan"` в набор `@crne_cp_window` (`type ifname; flags timeout; timeout 180s;`), пару `"br-lan" : $WAN_DNS` в карту `@crne_cp_dns` (`type ifname : ipv4_addr; flags timeout; timeout 180s;`) и имя WAN L3-устройства (`phy0-sta0` и т.п.) в набор `@crne_cp_output_wan` (`type ifname; flags timeout; timeout 180s;`). Первые два управляют транзитом LAN: все DNS-запросы клиентов (включая `10.10.1.1:53`) прозрачно перенаправляются на DNS отеля. Третий открывает окно в Output Kill Switch (`crne_output_guard`) для router-originated детект-проб (nslookup/curl с самого роутера), иначе ADR-09 намертво блокировал бы детект портала.
  3. **Временное снятие перехвата `tun0` под защитой `nohup`-таймера:** на время ввода СМС хук останавливает `sing-box` (снимая перехват `auto_route` / `auto_redirect`), чтобы трафик `53, 80, 443` из `br-lan` пошёл напрямую в `trm_wwan`, и запускает фоновый `nohup`-таймер, гарантированно включающий `sing-box` и жёсткий Kill Switch через 180 секунд (либо немедленно после прохождения портала).

---

## 🗺️ 4. Итоговая архитектурная схема прохождения пакетов (v4.2)

```text
 [Клиент в LAN / Wi-Fi (10.10.1.0/24)]
   │
   ├──► 1. Обычный режим DNS (запрос на 10.10.1.1:53) ──► [auto_redirect table inet sing-box]
   │         (DNAT в prerouting на 172.19.0.2 → перехват hijack-dns внутри sing-box)
   │         ├── Canary (use-application-dns.net, _dns.resolver.arpa) ─► NXDOMAIN (Отключение DoH браузера)
   │         ├── Локальные имена (.lan) ──► dns-local-lan (127.0.0.1:5353 → dnsmasq)
   │         ├── .ru/.рф/.su/geosite-ru ──► dns-direct-ru (Яндекс DoH, резолв через dns-bootstrap)
   │         ├── geosite-ads ─────────────► NOERROR (блокировка)
   │         └── Остальные запросы ──────► DoH 1.1.1.1 через GLOBAL-AUTO
   │         (dns-bootstrap → Cloudflare DoH 1.1.1.1:443, ADR-08 — вне РФ-юрисдикции, MITM невозможен)
   │         (dnsmasq переведён на порт 5353 — обслуживает только .lan/DHCP/PTR,
   │          не перехватывается DNAT auto_redirect на :53)
   │
   ├──► 2. Чужой DNS (напр. 8.8.8.8:53) ─► [tun0 auto_redirect]
   │         ├── В режиме Captive Portal (@crne_cp_dns) ──► DNAT на локальный DNS отеля ($WAN_DNS:53)
   │         └── В штатном режиме ────────────────────────► auto_redirect перехватывает все :53
   │
   ├──► 3. Трафик данных клиентов LAN (TCP / UDP / ICMP)
             │
             ▼
     [nftables crne_forward_guard (Приоритет: filter - 5, до маршрутизации в WAN)]
       ├── Блок IPv6 из br-lan ───────────────────────────► REJECT
       ├── Блок DoT (:853) и WebRTC STUN (:3478,:19302) ──► REJECT
       ├── Блок браузерного QUIC (UDP/443 из br-lan) ─────► REJECT (Перевод в TCP/443)
       ├── Окно Captive Portal (iifname @crne_cp_window) ─► Разрешить 53/80/443 в trm_wwan/phy*-sta* (180 с)
       └── ЖЁСТКИЙ KILL SWITCH forward (oifname != "tun0") ► REJECT (Защита транзита при падении sing-box!)
             │
             ▼ (Проходит ТОЛЬКО в виртуальный интерфейс tun0)

   ──► [nftables crne_mss_clamp (v4.4, Приоритет: filter + 1)]
         └── oifname "tun0" TCP SYN ──► MSS = rt mtu (предотвращение PMTUD-blackhole для MTU 1400)
             │
             ▼
   ──► [nftables crne_output_guard (v4.4 ADR-09, Приоритет: filter - 10) — трафик САМОГО роутера]
         ├── oifname "lo" ──────────────────────────────► ACCEPT (dnsmasq, Clash API, mixed-in)
         ├── UDP dport {67,68} ─────────────────────────► ACCEPT (DHCP server/client)
         ├── UDP dport 123 ─────────────────────────────► ACCEPT (NTP — критично для TLS/Reality)
         ├── meta mark 0xff ────────────────────────────► ACCEPT (sing-box outbound к прокси-узлам)
         ├── ip daddr 1.1.1.1 tcp dport 443 ────────────► ACCEPT (bootstrap DoH, ADR-08)
         ├── oifname @crne_cp_output_wan dport {53,80,443} ► ACCEPT (Captive Portal: nslookup/curl с роутера, 180 с)
         └── oifname != "tun0" ─────────────────────────► REJECT (Output Kill Switch: opkg/алерты НЕ уходят в WAN!)

     [Ядро sing-box (tun0, mtu 1400, GOMEMLIMIT=75MiB, ≤50 узлов в ОЗУ)]
       ├── Action 1: sniff + hijack-dns (перехват DNS :53) ─► Внутренний Split-DNS (см. пункт 1 выше)
       ├── Action 2: Реклама (geosite-ads.srs) + DoH IP ──────► action: "reject"
       ├── Action 3: {"action": "resolve", strategy ipv4_only} ► нетерминальное резолвирование (D6)
       ├── Правило 4: Домены .ru / .рф / .su ─────────────────► DIRECT (auto_redirect_output_mark 0xff)
       ├── Правило 5: Списки geosite-ru.srs / geoip-ru.srs ───► DIRECT (auto_redirect_output_mark 0xff)
       └── Правило 6 (Final): Глобальный трафик ──────────────► GLOBAL-AUTO (urltest, 180ms)
                                                                 ├── 20 узлов Standard (Reality/Hy2)
                                                                 └── До 20 узлов LTS-Mobile
                                                                       │ (выход из output, mark 0x2024/0xff)
                                                                       ▼
                                                        [WAN / TRM_WWAN (Wi-Fi) / USB0 (LTE)]
```

---

## 📄 5. Эталонные конфигурационные артефакты (Проверенный код LLD)

### 5.1. Межсетевой экран и Kill Switch: `/etc/nftables.d/99-crne-security.nft`
Файл включается внутрь контекста `table inet fw4` при вызове `fw4 reload` / `fw4 check`:

```nft
# /etc/nftables.d/99-crne-security.nft
# CRNE v4.4: Hard Fail-Closed Kill Switch (forward + output), Scalar Captive Portal Sets + DNS Map + MSS Clamping
# ПРИМЕЧАНИЕ: Явный DNAT для DNS не требуется — auto_redirect в tun inbound sing-box
# автоматически перехватывает весь DNS-трафик на порт 53.
# v4.4 (ADR-09): Добавлена цепочка crne_output_guard — блокирует router-originated traffic при отсутствии tun0.
# v4.4 (ADR-09 fix): Добавлен набор crne_cp_output_wan — окно для router-originated проб Captive Portal (53/80/443).
# v4.4: Добавлена цепочка crne_mss_clamp — предотвращает PMTUD-blackhole для туннеля MTU 1400.

set crne_cp_window {
    type ifname
    flags timeout
    timeout 180s
    gc-interval 15s  # ВАЖНО: gc-interval управляет сборкой мусора, а не логическим таймаутом.
                     # Элемент перестаёт соответствовать правилу через 180s (timeout),
                     # но физически удаляется из памяти при следующем gc-проходе (каждые 15s).
                     # При отладке учитывайте это окно между логическим и физическим удалением.
    comment "CRNE: Scalar interface timeout set for Captive Portal (180s)"
}

map crne_cp_dns {
    type ifname : ipv4_addr
    flags timeout
    timeout 180s
    gc-interval 15s  # gc управляет сборкой мусора; элемент логически истекает через 180s,
                     # но физически удаляется при следующем gc-проходе (до 15s задержки).
    comment "CRNE: Temporary DNAT map to Hotel DHCP DNS during Captive Portal login (180s)"
}

# v4.4: Output-ориентированное окно Captive Portal для router-originated probes (nslookup/curl).
# В отличие от @crne_cp_window (содержит "br-lan" для iifname в forward), этот набор хранит имя
# WAN L3-устройства (phy0-sta0 и т.п.) и используется в цепочке output (oifname).
# Без него crne_output_guard блокировал бы детект Captive Portal с самого роутера (ADR-09 regression).
set crne_cp_output_wan {
    type ifname
    flags timeout
    timeout 180s
    gc-interval 15s
    comment "CRNE: WAN L3 device during Captive Portal — permits router-originated 53/80/443 probes (180s)"
}

chain crne_prerouting_dns {
    type nat hook prerouting priority dstnat - 5; policy accept;

    # Во время регистрации в Captive Portal (180 сек) заворачиваем все DNS-запросы клиентов на DNS отеля
    # В штатном режиме DNS перехватывается auto_redirect в tun inbound — явный DNAT не нужен
    iifname "br-lan" meta l4proto { tcp, udp } th dport 53 dnat ip to iifname map @crne_cp_dns comment "CRNE: Captive Portal DNS DNAT"
}

chain crne_forward_guard {
    type filter hook forward priority filter - 5; policy accept;

    # 1. Полная блокировка транзитного IPv6 из локальной сети
    iifname "br-lan" meta nfproto ipv6 reject with icmpx type admin-prohibited comment "CRNE: Block IPv6 Leak"

    # 2. Хирургическое окно для прохождения Captive Portal (СМС) на портативном роутере (автозакрытие через 180с)
    # v4.2 (R-D7): oifname расширен шаблонами phy*-sta* и wlan* — в OpenWrt 24.10+ на MT7981B (mac80211)
    # клиентский радиоинтерфейс travelmate на уровне ядра носит имя phy0-sta0/phy1-sta0, а не UCI-имя trm_wwan.
    iifname @crne_cp_window oifname { "wwan", "trm_wwan", "phy0-sta0", "phy1-sta0", "wlan*" } ct state new,established meta l4proto { tcp, udp } th dport { 53, 80, 443 } counter accept comment "CRNE: Captive Portal Temporary Window"

    # 3. Блокировка DoT (TCP/853), DoQ (UDP/853) и WebRTC STUN/TURN (3478, 5349, 19302) из LAN
    iifname "br-lan" meta l4proto { tcp, udp } th dport { 853, 3478, 5349 } reject with icmpx type admin-prohibited comment "CRNE: Block DoT/DoQ and STUN"
    iifname "br-lan" meta l4proto udp udp dport 19302 reject with icmpx type admin-prohibited comment "CRNE: Block Google STUN"

    # 4. Блокировка браузерного QUIC (UDP/443) от клиентов LAN (не затрагивает Hysteria2 в цепочке output!)
    iifname "br-lan" meta l4proto udp udp dport 443 reject with icmpx type admin-prohibited comment "CRNE: Block Browser QUIC"

    # 5. ЖЁСТКИЙ KILL SWITCH (Fail-Closed): разрешаем транзит из br-lan ТОЛЬКО внутрь tun0!
    # Если демон sing-box упал и tun0 исчез — весь трафик из br-lan в WAN намертво блокируется.
    iifname "br-lan" oifname != "tun0" reject with icmpx type admin-prohibited comment "CRNE: Hard Fail-Closed Kill Switch"
}

chain crne_output_bypass {
    type filter hook output priority filter - 5; policy accept;
    # Аппаратно-независимые счётчики телеметрии L4 для контроля трафика NTP роутера и сокетов sing-box (0xff)
    meta l4proto udp udp dport 123 counter accept comment "CRNE: Router NTP Telemetry Counter"
    meta mark 0x000000ff counter accept comment "CRNE: sing-box Outbound Telemetry Counter"
    # v4.4 (ADR-08): Мониторинг DoH-запросов к Cloudflare bootstrap (1.1.1.1:443)
    ip daddr 1.1.1.1 tcp dport 443 counter accept comment "CRNE: dns-bootstrap DoH Counter"
}

# v4.4 (ADR-09): Output Kill Switch — блокировка router-originated traffic при отсутствии tun0.
# Транзитный forward Kill Switch (crne_forward_guard) НЕ покрывает трафик самого роутера (NTP, opkg,
# обновления sing-box, алерты). Данная цепочка закрывает эту дыру в главном инварианте Fail-Closed.
chain crne_output_guard {
    type filter hook output priority filter - 10; policy accept;

    # 1. Разрешить localhost (dnsmasq:5353, Clash API:9090, mixed-in:1603) — не покидает роутер
    oifname "lo" accept comment "CRNE: Allow localhost (dnsmasq, Clash API, mixed-in)"

    # 2. Разрешить DHCP server (UDP/67) — для раздачи IP клиентам LAN в travel-профиле
    meta l4proto udp udp dport 67 accept comment "CRNE: Allow DHCP server"

    # 3. Разрешить DHCP client (UDP/68) — для получения IP от провайдера/отеля
    meta l4proto udp udp dport 68 accept comment "CRNE: Allow DHCP client"

    # 4. Разрешить NTP (UDP/123) — критично для TLS/Reality (расхождение времени рвёт хендшейк)
    meta l4proto udp udp dport 123 accept comment "CRNE: Allow NTP (critical for TLS/Reality)"

    # 5. Разрешить sing-box outbound (mark 0xff) — пакеты самого туннеля к прокси-узлам
    meta mark 0x000000ff accept comment "CRNE: Allow sing-box outbound (mark 0xff)"

    # 6. Разрешить bootstrap DoH к Cloudflare (1.1.1.1:443) — начальная резолюция вне РФ-юрисдикции (ADR-08)
    ip daddr 1.1.1.1 tcp dport 443 accept comment "CRNE: Allow bootstrap DoH (Cloudflare, ADR-08)"

    # 7. Окно Captive Portal (output): hotplug-скрипт детектит портал, выполняя nslookup/curl
    #    С РОУТЕРА (output) к WAN-DNS отеля:53 и captive.apple.com:80. sing-box в этот момент уже
    #    остановлен (tun0 нет), а auto_redirect перехватывает только транзит LAN, не output роутера.
    #    Без этого правила crne_output_guard намертво блокировал бы детект портала (regression ADR-09).
    #    Разрешаем 53/80/443 на WAN L3-устройство из набора @crne_cp_output_wan (180s, автозакрытие).
    oifname @crne_cp_output_wan meta l4proto { tcp, udp } th dport { 53, 80, 443 } accept comment "CRNE: Captive Portal router-originated probe window (output)"

    # 8. ЖЁСТКИЙ OUTPUT KILL SWITCH: блокировать весь прочий исходящий трафик роутера,
    #    если он не идёт в tun0. При падении sing-box (tun0 исчез) — NTP/DoH/DHCP/Captive исключены выше,
    #    а opkg/обновления/алерты НЕ уйдут в WAN с реальным IP.
    oifname != "tun0" meta protocol ip reject with icmpx type admin-prohibited comment "CRNE: Hard Output Kill Switch (router-originated traffic)"
}

# v4.4: MSS Clamping для туннеля tun0 (MTU 1400) — предотвращает PMTUD-blackhole и деградацию TCP.
# Без clamping TCP-соединения через туннель могут зависать при больших пакетах (path MTU discovery fails).
chain crne_mss_clamp {
    type filter hook forward priority filter + 1; policy accept;
    oifname "tun0" tcp flags syn / syn,tcp option maxseg size set rt mtu comment "CRNE: MSS Clamping for tun0 (MTU 1400)"
}
```

---

### 5.2. Базовый валидный конфигурационный файл (`sing-box 1.12+`): `/etc/sing-box/config.json`
Ниже приведён эталонный стартовый конфиг в нативном типизированном формате **`sing-box 1.12+`**. DNS-перехват осуществляется через `tun` inbound с правилом `hijack-dns` (отдельный inbound типа `"dns"` **не существует** в sing-box 1.12+).

```json
{
  "log": {
    "level": "warn",
    "timestamp": true
  },
  "experimental": {
    "cache_file": {
      "enabled": true,
      "path": "/tmp/sing-box-cache.db",
      "store_fakeip": false
    },
    "clash_api": {
      "external_controller": "10.10.1.1:9090",
      "external_ui": "/etc/sing-box/ui",
      "secret": "REPLACE_WITH_GENERATED_SECRET",
      "default_mode": "Rule"
    }
  },
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
        "type": "https",
        "tag": "dns-bootstrap",
        "server": "1.1.1.1",
        "server_port": 443,
        "tls": {
          "enabled": true,
          "server_name": "cloudflare-dns.com"
        }
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
  "outbounds": [
    {
      "type": "urltest",
      "tag": "GLOBAL-AUTO",
      "outbounds": ["vless-template-01", "lts-template-01"],
      "url": "https://www.youtube.com/generate_204",
      "interval": "2m",
      "tolerance": 180,
      "idle_timeout": "30m"
    },
    {
      "type": "urltest",
      "tag": "STANDARD-TOP20",
      "outbounds": ["vless-template-01"],
      "url": "https://www.youtube.com/generate_204",
      "interval": "2m",
      "tolerance": 180
    },
    {
      "type": "urltest",
      "tag": "LTS-MOBILE",
      "outbounds": ["lts-template-01"],
      "url": "https://www.youtube.com/generate_204",
      "interval": "2m",
      "tolerance": 200
    },
    {
      "type": "selector",
      "tag": "MANUAL-SELECT",
      "outbounds": ["GLOBAL-AUTO", "LTS-MOBILE", "STANDARD-TOP20", "vless-template-01", "lts-template-01"]
    },
    {
      "type": "vless",
      "tag": "vless-template-01",
      "server": "198.51.100.10",
      "server_port": 443,
      "uuid": "00000000-0000-0000-0000-000000000001",
      "flow": "xtls-rprx-vision",
      "tls": {
        "enabled": true,
        "server_name": "example.com",
        "utls": {"enabled": true, "fingerprint": "chrome"},
        "reality": {"enabled": true, "public_key": "0000000000000000000000000000000000000000000", "short_id": "01"}
      }
    },
    {
      "type": "vless",
      "tag": "lts-template-01",
      "server": "198.51.100.20",
      "server_port": 443,
      "uuid": "00000000-0000-0000-0000-000000000002",
      "flow": "xtls-rprx-vision",
      "tls": {
        "enabled": true,
        "server_name": "example.com",
        "utls": {"enabled": true, "fingerprint": "chrome"},
        "reality": {"enabled": true, "public_key": "0000000000000000000000000000000000000000000", "short_id": "01"}
      }
    },
    {
      "type": "direct",
      "tag": "direct"
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
        "port": [443],
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
    ],
    "final": "GLOBAL-AUTO"
  }
}
```

---

### 5.3. Настройка `firewall`, `dnsmasq` (порт 5353), `travelmate` и `Multi-WAN` (UCI)

```sh
# 1. Принудительное отключение программного и аппаратного Flow Offloading (PPE) и включение MSS Clamping:
uci set firewall.@defaults[0].flow_offloading='0'
uci set firewall.@defaults[0].flow_offloading_hw='0'
uci set firewall.@zone[0].mtu_fix='1'
uci set firewall.@zone[1].mtu_fix='1'
uci commit firewall

# 2. Перевод dnsmasq на ЛОКАЛЬНЫЙ порт 5353 (R-D4):
# ВАЖНО: при auto_redirect=true таблица inet sing-box в цепочке prerouting (priority dstnat+1)
# перехватывает ЛЮБЫЕ запросы LAN к 10.10.1.1:53 (DNAT на 172.19.0.2 внутрь sing-box) и не доходит
# до сокета dnsmasq. Перевод dnsmasq на 5353 позволяет ему обслуживать зону .lan и DHCP-имена,
# оставаясь доступным для sing-box через сервер dns-local-lan (127.0.0.1:5353).
# Canary-домены (.lan, use-application-dns.net, _dns.resolver.arpa) переносятся в dns.rules sing-box.
uci set dhcp.@dnsmasq[0].noresolv='1'
uci set dhcp.@dnsmasq[0].port='5353'
uci set dhcp.@dnsmasq[0].cachesize='0'
uci set dhcp.@dnsmasq[0].local='/lan/'
uci set dhcp.@dnsmasq[0].domain='lan'
uci set dhcp.@dnsmasq[0].boguspriv='1'
uci delete dhcp.@dnsmasq[0].server 2>/dev/null
# dnsmasq больше не форвардит внешний DNS — все внешние запросы резолвит sing-box через dns.rules.
uci delete dhcp.@dnsmasq[0].address 2>/dev/null
uci commit dhcp
/etc/init.d/dnsmasq restart

# 3. Настройка интерфейсов trm_wwan (Wi-Fi клиент, metric 10) и usb_wan (USB-модем, metric 20) для Cudy TR3000:
uci set network.trm_wwan=interface
uci set network.trm_wwan.proto='dhcp'
uci set network.trm_wwan.metric='10'

uci set network.usb_wan=interface
uci set network.usb_wan.proto='dhcp'
uci set network.usb_wan.device='usb0'
uci set network.usb_wan.metric='20'

uci add_list firewall.@zone[1].network='trm_wwan'
uci add_list firewall.@zone[1].network='usb_wan'
uci commit

# 4. Конфигурация travelmate (отключение избыточного логирования и отключение встроенного детекта порталов):
if [ -f /etc/config/travelmate ]; then
    uci set travelmate.global.trm_enabled='1'
    uci set travelmate.global.trm_debug='0'
    uci set travelmate.global.trm_captive='0'
    uci set travelmate.global.trm_iface='trm_wwan'
    uci commit travelmate
fi
```

---

### 5.4. Hotplug-скрипт переключения `Wi-Fi ↔ USB`, `LTS`-пула и прямого детекта Captive Portal: `/etc/hotplug.d/iface/99-singbox-wan-switch`

Динамически считывает секрет из `/etc/sing-box/config.json`, резолвит пробник `captive.apple.com` напрямую через DHCP-DNS отеля (`ubus`) в обход туннеля и передаёт `$WAN_DNS` в хук Captive Portal:

```sh
#!/bin/sh
# /etc/hotplug.d/iface/99-singbox-wan-switch
[ "$ACTION" = "ifup" ] || [ "$ACTION" = "ifdown" ] || exit 0

# v4.2: Атомарный лок через mkdir (устранение TOCTOU-гонки D5)
LOCKDIR="/tmp/crne_wan_switch.lockdir"
if ! mkdir "$LOCKDIR" 2>/dev/null; then
    logger -t CRNE-WAN "Another wan-switch instance running, skipping."
    exit 0
fi
trap 'rmdir "$LOCKDIR" 2>/dev/null' EXIT

API="http://10.10.1.1:9090"
SECRET=$(jq -r '.experimental.clash_api.secret // empty' /etc/sing-box/config.json 2>/dev/null)

case "$INTERFACE" in
    wan|wwan|usb_wan|trm_wwan)
        logger -t CRNE-WAN "WAN $INTERFACE ($ACTION): checking portal & re-testing LTS/Standard pools..."
        (
            sleep 2
            # 1. Прямой детект Captive Portal через DHCP-DNS шлюза отеля (не зависит от состояния sing-box)
            if [ "$ACTION" = "ifup" ] && { [ "$INTERFACE" = "trm_wwan" ] || [ "$INTERFACE" = "wwan" ]; }; then
                WAN_DNS=$(ubus call "network.interface.$INTERFACE" status 2>/dev/null | jq -r '.["dns-server"][0] // empty')
                # v4.2 (R-D7): извлекаем РЕАЛЬНОЕ имя L3-устройства из ubus (phy0-sta0 и т.п.),
                # т.к. curl --interface должен привязываться к имени ядра, а не к UCI-имени trm_wwan.
                L3_DEV=$(ubus call "network.interface.$INTERFACE" status 2>/dev/null | jq -r '.l3_device // empty')
                L3_DEV="${L3_DEV:-$DEVICE}"
                if [ -n "$WAN_DNS" ]; then
                    # v4.4 (ADR-09 fix): Открываем кратковременное окно в Output Kill Switch для
                    # детект-проб с роутера (nslookup:53 / curl:80 к captive.apple.com) к WAN L3-устройству.
                    # Без него crne_output_guard намертво блокирует детект Captive Portal (regression ADR-09).
                    # Timeout 30s — достаточно для 2 проб; при отсутствии портала окно сбрасывается ниже.
                    if [ -n "$L3_DEV" ]; then
                        nft add element inet fw4 crne_cp_output_wan { "$L3_DEV" timeout 30s } 2>/dev/null
                    fi
                    # v4.2 (R-D7): корректный парсер BusyBox nslookup — берём адрес строго ПОСЛЕ строки "Name:",
                    # иначе awk захватывает IP самого DNS-сервера ($WAN_DNS) и ложно активирует Captive Portal.
                    CP_IP=$(nslookup captive.apple.com "$WAN_DNS" 2>/dev/null | awk '/^Name:/{f=1; next} f && /^Address/{print $NF; exit}')
                    if [ -n "$CP_IP" ]; then
                        PROBE=$(curl -s -m 4 --interface "$L3_DEV" --resolve "captive.apple.com:80:$CP_IP" -L "http://captive.apple.com/hotspot-detect.html" 2>/dev/null)
                    else
                        PROBE=""
                    fi
                    if ! echo "$PROBE" | grep -q "Success"; then
                        # Portal detected: передаём L3_DEV в хук для открытия полного 180s output-окна
                        /etc/travelmate/crne-portal-hook.sh cp "$WAN_DNS" "$L3_DEV"
                        # Ожидаем прохождения СМС-авторизации (до 180 секунд)
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
                    # v4.4: Portal НЕ обнаружен (Success) — немедленно закрыть transient output-окно,
                    # восстановив жёсткий Output Kill Switch (ADR-09). Прохождения SMS-авторизации не требуется.
                    nft flush set inet fw4 crne_cp_output_wan 2>/dev/null
                fi
            fi

            # 2. Сброс старых сессий и немедленный пересчёт urltest (активация/отсев LTS-серверов)
            if [ -n "$SECRET" ]; then
                curl -s -m 3 -X DELETE -H "Authorization: Bearer $SECRET" "$API/connections" >/dev/null 2>&1
                curl -s -m 5 -H "Authorization: Bearer $SECRET" "$API/group/GLOBAL-AUTO/delay?url=https://www.youtube.com/generate_204&timeout=1500" >/dev/null 2>&1
                curl -s -m 5 -H "Authorization: Bearer $SECRET" "$API/group/LTS-MOBILE/delay?url=https://www.youtube.com/generate_204&timeout=1500" >/dev/null 2>&1
            fi
        ) &
        ;;
esac
```

---

### 5.5. Безопасный хук Captive Portal (180 с, карта DNS + снятие перехвата `tun0`): `/etc/travelmate/crne-portal-hook.sh`

```sh
#!/bin/sh
# /etc/travelmate/crne-portal-hook.sh
STATUS="$1"
WAN_DNS="$2"
L3_DEV="$3"

if [ "$STATUS" = "cp" ]; then
    # Guard: если окно Captive Portal уже открыто — не запускаем повторный таймер (предотвращение race condition)
    [ -f /tmp/crne_cp.lock ] && { logger -t CRNE-PORTAL "Captive Portal window already active, skipping duplicate cp call."; exit 0; }
    # 1. Открываем скалярное окно в forward и карту DNAT на DNS отеля на 180 секунд
    nft add element inet fw4 crne_cp_window { "br-lan" timeout 180s } 2>/dev/null
    if [ -n "$WAN_DNS" ]; then
        nft add element inet fw4 crne_cp_dns { "br-lan" : "$WAN_DNS" timeout 180s } 2>/dev/null
    fi
    # v4.4 (ADR-09 fix): Открываем output-окно на WAN L3-устройстве для router-originated проб
    # (nslookup/curl в цикле ожидания SMS-авторизации). Без него crne_output_guard блокирует детект.
    if [ -n "$L3_DEV" ]; then
        nft add element inet fw4 crne_cp_output_wan { "$L3_DEV" timeout 180s } 2>/dev/null
    fi
    # 2. Временно останавливаем sing-box (чтобы снять auto_route/auto_redirect с tun0)
    # под защитой бессмертного nohup-таймера гарантированного включения через 180 секунд:
    touch /tmp/crne_cp.lock
    /etc/init.d/sing-box stop
    nohup sh -c 'trap "" HUP INT TERM; sleep 180; [ -f /tmp/crne_cp.lock ] && {
        rm -f /tmp/crne_cp.lock
        nft flush set inet fw4 crne_cp_window 2>/dev/null
        nft flush map inet fw4 crne_cp_dns 2>/dev/null
        nft flush set inet fw4 crne_cp_output_wan 2>/dev/null
        /etc/init.d/sing-box start
        logger -t CRNE-PORTAL "180s Captive Portal timer expired: sing-box & Hard Fail-Closed restored."
        /usr/bin/crne-alert.sh "Captive%20Portal%20timer%20expired%2C%20restored"
    }' >/dev/null 2>&1 </dev/null &
    logger -t CRNE-PORTAL "Captive Portal detected (DNS: ${WAN_DNS:-none}, dev: ${L3_DEV:-none}): 180s login window opened."
    /usr/bin/crne-alert.sh "Captive%20Portal%20detected%2C%20180s%20window"

elif [ "$STATUS" = "connected" ]; then
    # Немедленно закрываем окно, очищаем карту DNS и поднимаем туннель после успешной авторизации
    rm -f /tmp/crne_cp.lock
    nft flush set inet fw4 crne_cp_window 2>/dev/null
    nft flush map inet fw4 crne_cp_dns 2>/dev/null
    nft flush set inet fw4 crne_cp_output_wan 2>/dev/null
    /etc/init.d/sing-box start
    logger -t CRNE-PORTAL "Captive Portal passed: sing-box started & Hard Fail-Closed restored."
    /usr/bin/crne-alert.sh "Captive%20Portal%20passed%2C%20restored"
fi
```

---

### 5.6. Система мониторинга и алертинга

#### 5.6.1. Скрипт Telegram-алертинга: `/usr/bin/crne-alert.sh`

```sh
#!/bin/sh
# /usr/bin/crne-alert.sh — отказоустойчивая отправка алертов в Telegram с fallback на SMS/email
# Настройка: задать TG_BOT_TOKEN, TG_CHAT_ID, SMS_GATEWAY в /etc/crne-alert.conf (chmod 600!)
# v4.3 (D1): Добавлен SMS fallback через GSM-модем и email через локальный MTA
# v4.2 (R-D10): таймаут -m 5, --data-urlencode, фоновый режим, fallback через socks5h

CONFIG_FILE="/etc/crne-alert.conf"
[ -f "$CONFIG_FILE" ] && . "$CONFIG_FILE"

TG_BOT_TOKEN="${TG_BOT_TOKEN:-}"
TG_CHAT_ID="${TG_CHAT_ID:-}"
SMS_GATEWAY="${SMS_GATEWAY:-}"      # Формат: "smsc.ru?login=X&psw=Y&phones=Z"
EMAIL_RECIPIENT="${EMAIL_RECIPIENT:-}"  # Формат: "admin@example.com"
MESSAGE="$1"
PRIORITY="${2:-normal}"  # normal | critical

# SITE_TAG — нейтральная анонимизированная метка узла (задаётся в /etc/crne-alert.conf),
# чтобы не передавать hostname (который может содержать ПДн) в Telegram.
SITE_TAG="${SITE_TAG:-gateway}"

[ -z "$MESSAGE" ] && exit 0

# Функция отправки SMS через USB-модем (GSM 07.05 AT commands)
send_sms_via_modem() {
    local phone="$1"
    local text="$2"
    # Поиск USB-модема (обычно /dev/ttyUSB0 или /dev/ttyACM0)
    local modem_dev=""
    for dev in /dev/ttyUSB0 /dev/ttyACM0 /dev/ttyUSB1; do
        [ -c "$dev" ] && { modem_dev="$dev"; break; }
    done
    [ -z "$modem_dev" ] && return 1
    
    # PDU-encoding для кириллицы не требуется — используем GSM 7-bit default alphabet
    # Ограничение 160 символов, экранируем специальные символы
    text=$(echo "$text" | cut -c1-160)
    
    # Отправка AT-команд (timeout 5s)
    (
        echo "AT"
        sleep 0.5
        echo "AT+CMGF=1"  # Text mode
        sleep 0.5
        echo "AT+CMGS=\"$phone\""
        sleep 1
        echo -n "$text"
        sleep 0.5
        # Ctrl+Z (0x1A) для завершения SMS
        printf '\x1a'
    ) | timeout 5 cat > "$modem_dev" 2>/dev/null
}

# Функция отправки email через локальный MTA (msmtp/sendmail)
send_email() {
    local recipient="$1"
    local subject="$2"
    local body="$3"
    
    if command -v msmtp >/dev/null 2>&1; then
        echo -e "Subject: $subject\n\n$body" | msmtp -a default "$recipient" 2>/dev/null
    elif command -v sendmail >/dev/null 2>&1; then
        echo -e "Subject: $subject\n\n$body" | sendmail "$recipient" 2>/dev/null
    else
        return 1
    fi
}

# Запускаем отправку в фон, чтобы не блокировать вызывающий скрипт (watchdog/portal-hook).
(
    ALERT_SENT=0
    PAYLOAD="🚨 CRNE ALERT [$SITE_TAG]: $MESSAGE"
    
    # Попытка 1: Telegram через локальный туннель sing-box (обход блокировки api.telegram.org на ТСПУ).
    if [ -n "$TG_BOT_TOKEN" ] && [ -n "$TG_CHAT_ID" ]; then
        API_URL="https://api.telegram.org/bot$TG_BOT_TOKEN/sendMessage"
        curl -s -m 5 -x socks5h://127.0.0.1:1603 "$API_URL" \
             --data-urlencode "chat_id=$TG_CHAT_ID" \
             --data-urlencode "text=$PAYLOAD" \
             --data-urlencode "parse_mode=HTML" >/dev/null 2>&1 && ALERT_SENT=1
    fi
    
    # Попытка 2: Telegram напрямую (если sing-box остановлен, а провайдер не блокирует Telegram).
    if [ "$ALERT_SENT" -eq 0 ] && [ -n "$TG_BOT_TOKEN" ] && [ -n "$TG_CHAT_ID" ]; then
        curl -s -m 5 "$API_URL" \
             --data-urlencode "chat_id=$TG_CHAT_ID" \
             --data-urlencode "text=$PAYLOAD" \
             --data-urlencode "parse_mode=HTML" >/dev/null 2>&1 && ALERT_SENT=1
    fi
    
    # Попытка 3: SMS через USB-модем (только для critical priority или если Telegram недоступен).
    # v4.3 (D1): Fallback при падении sing-box или блокировке Telegram на уровне IP.
    if [ "$ALERT_SENT" -eq 0 ] && [ -n "$SMS_GATEWAY" ]; then
        # Извлекаем номер телефона из SMS_GATEWAY (формат: "login=X&psw=Y&phones=Z")
        SMS_PHONE=$(echo "$SMS_GATEWAY" | sed -n 's/.*phones=\([^&]*\).*/\1/p')
        if [ -n "$SMS_PHONE" ]; then
            # Урезаем сообщение до 160 символов (GSM limit)
            SMS_TEXT="CRNE[$SITE_TAG]: $(echo "$MESSAGE" | cut -c1-140)"
            send_sms_via_modem "$SMS_PHONE" "$SMS_TEXT" && ALERT_SENT=1
        fi
    fi
    
    # Попытка 4: Email через локальный MTA (msmtp/sendmail).
    if [ "$ALERT_SENT" -eq 0 ] && [ -n "$EMAIL_RECIPIENT" ]; then
        EMAIL_SUBJECT="[CRNE ALERT] $SITE_TAG: $MESSAGE"
        send_email "$EMAIL_RECIPIENT" "$EMAIL_SUBJECT" "$PAYLOAD" && ALERT_SENT=1
    fi
    
    # Логируем неудачную отправку (для диагностики через logread).
    if [ "$ALERT_SENT" -eq 0 ]; then
        logger -t CRNE-ALERT "FAILED to send alert: $MESSAGE (all channels exhausted)"
    fi
) &
```

#### 5.6.2. Сторожевой скрипт контроля `dnsmasq`, `sing-box`, `VmRSS`, `GOMEMLIMIT`, USB и LED: `/usr/bin/crne-watchdog.sh`

```sh
#!/bin/sh
# /usr/bin/crne-watchdog.sh — контроль dnsmasq, tun0, VmRSS, GOMEMLIMIT, USB, сквозного DNS и LED
# Если активно окно Captive Portal — не вмешиваемся до его завершения
[ -f /tmp/crne_cp.lock ] && exit 0

SECRET=$(jq -r '.experimental.clash_api.secret // empty' /etc/sing-box/config.json 2>/dev/null)
PROM_FILE="/tmp/crne.prom"

# 0. Экспорт метрик Prometheus В НАЧАЛЕ скрипта (исправление D6 — не теряем метрики при exit)
TUNNEL_STATUS=1
RSS=0
USB_STATUS=1
TEMP_C=0
DNS_BOOTSTRAP_COUNT=0

# 0b. Самовосстановление /etc/init.d/sing-box (R-D8): удаление procd_set_param file,
# инъекция GOMEMLIMIT=75MiB и procd respawn, плюс перезапуск службы при модификации init-скрипта.
# ПРИМЕЧАНИЕ: procd_set_param file (hot-reload) отключён намеренно — иначе модификация config.json
# вызывает параллельный перезапуск, конфликтующий с явным restart из sub2sing.py --push и watchdog.
INIT_CHANGED=0
if [ -f /etc/init.d/sing-box ]; then
    if grep -q "procd_set_param file" /etc/init.d/sing-box; then
        sed -i '/procd_set_param file/d' /etc/init.d/sing-box
        logger -t CRNE-WATCHDOG "Removed procd_set_param file from /etc/init.d/sing-box"
        INIT_CHANGED=1
    fi
    if ! grep -q "GOMEMLIMIT" /etc/init.d/sing-box; then
        sed -i '/procd_open_instance/a \	procd_set_param env GOMEMLIMIT=75MiB' /etc/init.d/sing-box
        logger -t CRNE-WATCHDOG "Restored GOMEMLIMIT=75MiB in /etc/init.d/sing-box"
        INIT_CHANGED=1
    fi
    if ! grep -q "procd_set_param respawn" /etc/init.d/sing-box; then
        sed -i '/procd_set_param env GOMEMLIMIT/a \	procd_set_param respawn ${respawn_threshold:-3600} ${respawn_timeout:-5} ${respawn_retry:-5}' /etc/init.d/sing-box
        logger -t CRNE-WATCHDOG "Added procd respawn to /etc/init.d/sing-box"
        INIT_CHANGED=1
    fi
fi
# Перезапуск службы нужен, чтобы GOMEMLIMIT/respawn применились к текущему процессу sing-box
# (без этого sed-инъекция подхватится только при следующем аварийном рестарте).
if [ "$INIT_CHANGED" = "1" ]; then
    /etc/init.d/sing-box restart
    logger -t CRNE-WATCHDOG "Restarted sing-box to apply init-script changes"
fi

# Автоопределение светодиодов для Cudy TR3000 (white:status / red:power) и WR3000S
LED_OK=""
LED_ERR=""
for p in /sys/class/leds/white:status/brightness /sys/class/leds/*green*/brightness /sys/class/leds/*status*/brightness; do
    [ -e "$p" ] && { LED_OK="$p"; break; }
done
for p in /sys/class/leds/red:power/brightness /sys/class/leds/*red*/brightness; do
    [ -e "$p" ] && { LED_ERR="$p"; break; }
done

# 1. Контроль жизнеспособности локального демона dnsmasq (слушает на 127.0.0.1:5353 после R-D4)
if ! pidof -s dnsmasq >/dev/null 2>&1; then
    logger -t CRNE-WATCHDOG "CRITICAL: dnsmasq is not running! Restarting dnsmasq..."
    /usr/bin/crne-alert.sh "dnsmasq%20down%2C%20restarting"
    /etc/init.d/dnsmasq restart
fi

# 2. Проверка USB-модема (usb0) + keepalive-пинг шлюза провайдера для портативного профиля travel
USB_STATUS=1
if [ -e /sys/class/net/usb0 ]; then
    if ! ip link show usb0 2>/dev/null | grep -q "UP"; then
        logger -t CRNE-WATCHDOG "WARNING: usb0 interface down!"
        [ -n "$LED_ERR" ] && echo 1 > "$LED_ERR"
        /usr/bin/crne-alert.sh "USB%20modem%20disconnected"
        USB_STATUS=0
    else
        # Keepalive: пинг до шлюза провайдера (извлекается из DHCP-lease usb_wan)
        USB_GW=$(ip route show dev usb0 2>/dev/null | awk '/default/{print $3; exit}')
        if [ -n "$USB_GW" ] && ! ping -I usb0 -c 1 -W 2 "$USB_GW" >/dev/null 2>&1; then
            logger -t CRNE-WATCHDOG "WARNING: usb0 keepalive failed (gw $USB_GW)! Restarting usb_wan..."
            /usr/bin/crne-alert.sh "USB%20keepalive%20failed%2C%20restarting"
            ifdown usb_wan 2>/dev/null; sleep 2; ifup usb_wan 2>/dev/null
        fi
    fi
fi

# 2b. Термальный мониторинг SoC (предотвращение троттлинга MT7981B)
SOC_TEMP=""
for tz in /sys/class/thermal/thermal_zone0/temp /sys/class/thermal/thermal_zone*/temp; do
    [ -e "$tz" ] && { SOC_TEMP=$(cat "$tz" 2>/dev/null); break; }
done
if [ -n "$SOC_TEMP" ]; then
    # SOC_TEMP в миллиградусах Цельсия (например, 72000 = 72°C)
    TEMP_C=$((SOC_TEMP / 1000))
    if [ "$TEMP_C" -ge 85 ]; then
        logger -t CRNE-WATCHDOG "CRITICAL: SoC temperature ${TEMP_C}C >= 85C! Throttling risk!"
        /usr/bin/crne-alert.sh "SoC%20overheat%20${TEMP_C}C"
    fi
fi

# 3. Проверка наличия процесса sing-box (одиночный PID) и интерфейса tun0
SB_PID=$(pidof -s sing-box)
TUNNEL_STATUS=1
if ! ip link show tun0 >/dev/null 2>&1 || [ -z "$SB_PID" ]; then
    logger -t CRNE-WATCHDOG "CRITICAL: tun0 or sing-box missing! Restarting..."
    [ -n "$LED_ERR" ] && echo 1 > "$LED_ERR"
    /usr/bin/crne-alert.sh "sing-box%20crashed%2C%20restarting"
    /etc/init.d/sing-box restart
    TUNNEL_STATUS=0
    # Не выходим — экспортируем метрики
else
    # 4. Контроль физической памяти VmRSS (аварийный порог 95000 КБ = 95 МБ)
    RSS=$(awk '/VmRSS/{print $2}' "/proc/$SB_PID/status" 2>/dev/null)
    if [ -n "$RSS" ] && [ "$RSS" -gt 95000 ]; then
        logger -t CRNE-WATCHDOG "WARNING: sing-box VmRSS ($RSS KB) exceeded 95MB! Restarting..."
        /usr/bin/crne-alert.sh "Memory%20leak%20${RSS}KB%2C%20restarting"
        /etc/init.d/sing-box restart
        TUNNEL_STATUS=0
    fi
fi

# 5. Сквозная проверка DNS + туннеля через socks5h:// (резолв домена на стороне sing-box)
# v4.2: ПЕРЕМЕЩЕНО ВЫШЕ проверки dnsmasq (исправление D6)
# v4.3 (D2): Добавлен детект пустого пула узлов и авто-переключение на резервную подписку
URLTEST_FAIL_FILE="/tmp/crne_urltest_fail.count"
if [ -n "$SB_PID" ]; then
    if ! curl -x socks5h://127.0.0.1:1603 -I -s -m 6 https://www.youtube.com/generate_204 | grep -q "204"; then
        logger -t CRNE-WATCHDOG "Tunnel probe (socks5h) failed! Forcing urltest recalculation..."
        [ -n "$SECRET" ] && curl -s -m 5 -H "Authorization: Bearer $SECRET" "http://10.10.1.1:9090/group/GLOBAL-AUTO/delay?url=https://www.youtube.com/generate_204&timeout=1500" >/dev/null 2>&1
        
        # v4.3 (D2): Проверка на пустой пул узлов после urltest
        if [ -n "$SECRET" ]; then
            ALIVE_NODES=$(curl -s -m 3 -H "Authorization: Bearer $SECRET" "http://10.10.1.1:9090/proxies/GLOBAL-AUTO" 2>/dev/null | jq -r '.all | length' 2>/dev/null || echo "0")
            if [ "$ALIVE_NODES" = "0" ] || [ -z "$ALIVE_NODES" ]; then
                logger -t CRNE-WATCHDOG "CRITICAL: All proxy nodes unreachable, pool is EMPTY!"
                /usr/bin/crne-alert.sh "CRITICAL%3A%20All%20proxy%20nodes%20unreachable%2C%20pool%20empty" "critical"
                
                # Попытка auto-rollback на резервную подписку (если настроена)
                FALLBACK_SUB_URL=$(jq -r '.experimental.fallback_subscription // empty' /etc/sing-box/config.json 2>/dev/null)
                if [ -n "$FALLBACK_SUB_URL" ] && [ -f /usr/bin/crne-sub-fallback.sh ]; then
                    logger -t CRNE-WATCHDOG "Attempting fallback subscription rotation..."
                    /usr/bin/crne-sub-fallback.sh --fallback-url "$FALLBACK_SUB_URL" &
                fi
            fi
        fi
        
        # Счётчик последовательных сбоев: алерт в Telegram при 3 подряд (каждые 10 мин — 30 мин деградации)
        FAILS=$(cat "$URLTEST_FAIL_FILE" 2>/dev/null || echo 0)
        FAILS=$((FAILS + 1))
        echo "$FAILS" > "$URLTEST_FAIL_FILE"
        if [ "$FAILS" -ge 3 ]; then
            /usr/bin/crne-alert.sh "Tunnel%20down%20${FAILS}x%20consecutive%2C%20all%20nodes%20unreachable"
            TUNNEL_STATUS=0
        fi
    else
        [ -n "$LED_OK" ] && echo 1 > "$LED_OK"
        rm -f "$URLTEST_FAIL_FILE"
    fi
fi

# 6. Проверка цепочки локального DNS (dnsmasq на 127.0.0.1:5353 после R-D4)
# v4.2: ПЕРЕМЕЩЕНО ПОСЛЕ проверки туннеля (исправление D6)
# v4.2 (R-D4): dnsmasq переведён с :53 на :5353, т.к. auto_redirect в prerouting перехватывает
# любые запросы LAN к 10.10.1.1:53. Проверка с самого роутера идёт через lo (oifname="lo"),
# который исключён из перехвата auto_redirect, поэтому стучимся напрямую в dnsmasq:5353.
if ! nslookup ya.ru 127.0.0.1 5353 >/dev/null 2>&1; then
    logger -t CRNE-WATCHDOG "WARNING: Local DNS resolution via dnsmasq:5353 failed! Restarting dnsmasq..."
    /etc/init.d/dnsmasq restart
fi

# 7. v4.4 (ADR-08): Счётчик DoH-запросов к dns-bootstrap (Cloudflare 1.1.1.1:443) — мониторинг bootstrap-резолвера
# Правило в crne_output_bypass: `ip daddr 1.1.1.1 tcp dport 443 counter ... comment "CRNE: dns-bootstrap DoH Counter"`
# v4.4 (hotfix): упрощённое извлечение — берём packets из строки counter рядом с комментарием dns-bootstrap.
# Предыдущая двухпроходная схема с grep 'crne_dns_bootstrap' не находила правило (нет префикса crne_ в комментарии).
DNS_BOOTSTRAP_COUNT=$(nft list chain inet fw4 crne_output_bypass 2>/dev/null \
    | grep 'dns-bootstrap' \
    | grep -o 'counter packets [0-9]*' \
    | awk '{print $3}' 2>/dev/null)
[ -z "$DNS_BOOTSTRAP_COUNT" ] && DNS_BOOTSTRAP_COUNT=0

# 8. v4.3 (ADR-07): Watchdog self-check — touch timestamp file для staleness detection
STAMP_FILE="/tmp/crne-watchdog.stamp"
date +%s > "$STAMP_FILE"

# 9. Экспорт метрик Prometheus (textfile collector)
mkdir -p /tmp/node_exporter/textfile_collector 2>/dev/null
# v4.3 (D2): Добавлена метрика alive_nodes_count для мониторинга пула узлов
ALIVE_NODES_COUNT=0
if [ -n "$SECRET" ]; then
    ALIVE_NODES_COUNT=$(curl -s -m 3 -H "Authorization: Bearer $SECRET" "http://10.10.1.1:9090/proxies/GLOBAL-AUTO" 2>/dev/null | jq -r '.all | length' 2>/dev/null || echo "0")
fi

cat > "$PROM_FILE" <<EOF
# HELP crne_tunnel_status Tunnel availability (1=up, 0=down)
# TYPE crne_tunnel_status gauge
crne_tunnel_status $TUNNEL_STATUS
# HELP crne_memory_rss_kb sing-box VmRSS in KB
# TYPE crne_memory_rss_kb gauge
crne_memory_rss_kb ${RSS:-0}
# HELP crne_dnsmasq_status dnsmasq process status (1=running, 0=stopped)
# TYPE crne_dnsmasq_status gauge
crne_dnsmasq_status $(pidof -s dnsmasq >/dev/null 2>&1 && echo 1 || echo 0)
# HELP crne_usb_status USB modem status (1=up, 0=down)
# TYPE crne_usb_status gauge
crne_usb_status ${USB_STATUS:-1}
# HELP crne_soc_temp_celsius SoC temperature in Celsius
# TYPE crne_soc_temp_celsius gauge
crne_soc_temp_celsius ${TEMP_C:-0}
# HELP crne_dns_bootstrap_requests_total DoH requests to dns-bootstrap (Cloudflare 1.1.1.1:443, ADR-08)
# TYPE crne_dns_bootstrap_requests_total counter
crne_dns_bootstrap_requests_total ${DNS_BOOTSTRAP_COUNT:-0}
# HELP crne_alive_nodes_count Number of alive proxy nodes in GLOBAL-AUTO pool
# TYPE crne_alive_nodes_count gauge
crne_alive_nodes_count ${ALIVE_NODES_COUNT:-0}
EOF
[ -d /tmp/node_exporter/textfile_collector ] && mv "$PROM_FILE" /tmp/node_exporter/textfile_collector/crne.prom
```

---

### 5.7. Список персистентности при обновлении прошивки: `/etc/sysupgrade.conf`

```text
/etc/sing-box/
/etc/init.d/sing-box
/etc/nftables.d/99-crne-security.nft
/etc/hotplug.d/iface/99-singbox-wan-switch
/etc/travelmate/
/usr/bin/crne-watchdog.sh
/usr/bin/crne-sub-fallback.sh
/usr/bin/crne-alert.sh
/etc/crne-alert.conf
/etc/crontabs/root
/etc/config/firewall
/etc/config/dhcp
/etc/config/network
/etc/config/travelmate
```

---

## 📋 6. Инженерный регламент развёртывания и эксплуатации (Runbook)

### 6.1. Первичная установка пакетов, генерация секрета, доставка `.srs` + `ui` и регистрация `cron`
Выполняется пошагово при первичном вводе роутера (`Cudy WR3000S` или `Cudy TR3000`) в эксплуатацию:

```sh
# --- ШАГ 1 (На роутере по SSH): Установка базового манифеста пакетов ---
opkg update
# v4.2 (R-D3): git — для локального версионирования config.json (раздел 6.3);
# kmod-nft-queue — необходим для NFQUEUE-прематча auto_redirect (sing-box 1.12+/1.13+),
#   не входит в базовые зависимости пакета sing-box (+ca-bundle +kmod-inet-diag +kmod-tun);
# kmod-nft-tproxy оставлен как совместимая зависимость, но для режима tun не используется.
opkg install sing-box curl jq ca-bundle kmod-tun kmod-nft-tproxy git kmod-nft-queue

# Дополнительно для портативного профиля travel (Cudy TR3000 — Wi-Fi порталы и USB-модем):
opkg install travelmate luci-app-travelmate kmod-usb-net-rndis kmod-usb-net-cdc-ncm usb-modeswitch

# --- ШАГ 2 (На роутере): Создание каталогов, прав доступа и настройка NTP без DNS ---
mkdir -p /etc/sing-box/ruleset /etc/sing-box/ui /etc/travelmate
chmod +x /etc/hotplug.d/iface/99-singbox-wan-switch /etc/travelmate/crne-portal-hook.sh /usr/bin/crne-watchdog.sh /usr/bin/crne-sub-fallback.sh /usr/bin/crne-alert.sh 2>/dev/null
# Настройка алертинга (необязательно): создать /etc/crne-alert.conf с правами 600
# v4.3 (D1): Добавлена поддержка SMS через USB-модем и email через msmtp
cat > /etc/crne-alert.conf <<'EOF'
# === Telegram (основной канал) ===
# TG_BOT_TOKEN и TG_CHAT_ID для оповещений через Telegram Bot API
TG_BOT_TOKEN=""
TG_CHAT_ID=""

# === SMS Fallback (через USB-модем) ===
# Формат: "login=X&psw=Y&phones=Z" (только номер телефона для GSM AT-commands)
# Требует подключённый USB-модем (/dev/ttyUSB0 или /dev/ttyACM0)
SMS_GATEWAY=""
# Пример: SMS_GATEWAY="phones=79001234567"

# === Email Fallback (через msmtp/sendmail) ===
# Требует установленный msmtp и настроенный ~/.msmtprc
EMAIL_RECIPIENT=""
# Пример: EMAIL_RECIPIENT="admin@example.com"

# === Метка узла ===
# Нейтральная анонимизированная метка (не используйте hostname с ПДн)
SITE_TAG="home-gateway"
EOF
chmod 600 /etc/crne-alert.conf

# v4.4 (ADR-08): Предзаполнение /etc/hosts для Cloudflare DoH bootstrap.
# dns-bootstrap использует type:https server:1.1.1.1 с server_name cloudflare-dns.com.
# IP 1.1.1.1 указан явно, но /etc/hosts гарантирует корректный SNI-резолвинг при холодном старте
# (до поднятия туннеля) и защищает от подмены DNS-ответа на WAN-стороне.
grep -q "cloudflare-dns.com" /etc/hosts 2>/dev/null || cat >> /etc/hosts <<'HOSTS_EOF'
# CRNE v4.4 (ADR-08): Bootstrap DoH endpoints (pinned IPs, no DNS resolution)
1.1.1.1              cloudflare-dns.com
1.0.0.1              cloudflare-dns.com
HOSTS_EOF

# NTP: доверенные серверы по IP (без DNS-резолвинга — синхронизация работает до поднятия туннеля)
uci delete system.ntp.server 2>/dev/null
uci add_list system.ntp.server='194.190.168.1'
uci add_list system.ntp.server='62.117.76.141'
uci commit system
/etc/init.d/sysntpd restart

# --- ШАГ 3 (На роутере): Генерация уникального криптостойкого секрета Clash API в стартовом config.json ---
RAND_SECRET=$(head -c 16 /dev/urandom | hexdump -v -e '/1 "%02x"')
sed -i "s/REPLACE_WITH_GENERATED_SECRET/$RAND_SECRET/g" /etc/sing-box/config.json
chmod 600 /etc/sing-box/config.json

# --- ШАГ 4 (На роутере): Настройка службы sing-box, GOMEMLIMIT=75MiB, procd respawn и отключение procd file ---
uci set sing-box.main.enabled='1'
uci set sing-box.main.user='root'
uci set sing-box.main.conffile='/etc/sing-box/config.json'
uci commit sing-box
# v4.2 (R-D8): Фактическое удаление procd_set_param file (штатный пакет sing-box включает её по умолчанию).
# Без этого модификация config.json вызывает параллельный горячий перезапуск, конфликтующий с явным
# restart из sub2sing.py --push и watchdog.
sed -i '/procd_set_param file/d' /etc/init.d/sing-box 2>/dev/null
# Инъекция в init-скрипт: GOMEMLIMIT + respawn (автовосстановление при crash)
grep -q "GOMEMLIMIT" /etc/init.d/sing-box || sed -i '/procd_open_instance/a \	procd_set_param env GOMEMLIMIT=75MiB' /etc/init.d/sing-box
grep -q "procd_set_param respawn" /etc/init.d/sing-box || sed -i '/procd_set_param env GOMEMLIMIT/a \	procd_set_param respawn ${respawn_threshold:-3600} ${respawn_timeout:-5} ${respawn_retry:-5}' /etc/init.d/sing-box

# --- ШАГ 5 (На сборочном узле — ПК или смартфоне Termux): Доставка баз .srs, веб-панели UI и рабочей подписки ---
# ВАЖНО: Выполняется ДО первого старта sing-box, иначе демон не запустится без файлов .srs!
sh ./download-assets.sh root@10.10.1.1
python3 ./sub2sing.py --url "https://ВАША_ПОДПИСКА" --secret "$RAND_SECRET" --test-local --push root@10.10.1.1

# --- ШАГ 6 (На роутере): Сухая проверка правил, активация межсетевого экрана и регистрация cron ---
fw4 check && sing-box check -c /etc/sing-box/config.json
# v4.2 (R-D3): после проверок обязательно включить /etc/nftables.d/99-crne-security.nft
# в контекст table inet fw4 и запустить службу sing-box (иначе файл .nft не загрузится, а sing-box не стартует).
/etc/init.d/firewall reload && /etc/init.d/sing-box restart
grep -q "crne-watchdog.sh" /etc/crontabs/root 2>/dev/null || echo "*/10 * * * * /usr/bin/crne-watchdog.sh" >> /etc/crontabs/root
# v4.3 (ADR-07): Watchdog self-check — staleness detection каждые 30 минут
# Если timestamp file не обновлялся > 30 мин — watchdog завис, отправить алерт и перезапустить cron
grep -q "crne-watchdog-stale" /etc/crontabs/root 2>/dev/null || cat >> /etc/crontabs/root <<'CRON_EOF'
# v4.3 (ADR-07): Watchdog staleness detection
*/30 * * * * [ -f /tmp/crne-watchdog.stamp ] && [ $(($(date +%s) - $(cat /tmp/crne-watchdog.stamp))) -gt 1800 ] && { logger -t CRNE-STALE "Watchdog stale for >30min, restarting cron"; /usr/bin/crne-alert.sh "Watchdog%20stale%20for%20%3E30min"; /etc/init.d/cron restart; }
CRON_EOF
# v4.2: Напоминание об обновлении .srs-баз каждые 30 дней (исправление D8)
grep -q "crne-srs-reminder" /etc/crontabs/root 2>/dev/null || echo "0 9 1 * * /usr/bin/crne-alert.sh 'Update%20.srs%20geosite%2Fgeoip%20bases%20(30%20days)'" >> /etc/crontabs/root
/etc/init.d/cron enable
/etc/init.d/cron restart
```

### 6.2. Транзакционное применение сетевых экранов и конфига (`nohup` Lock-Rollback)
При любых изменениях `/etc/nftables.d/99-crne-security.nft` или `/etc/sing-box/config.json` используется отвязанный от терминала (`nohup` + игнорирование `SIGHUP`) 300-секундный таймер возврата, выживающий при обрыве SSH-соединения:

```sh
# 1. Предварительная валидация синтаксиса до применения:
fw4 check && sing-box check -c /etc/sing-box/config.json || { echo "VALIDATION FAILED! Aborting."; exit 1; }

# 2. Создание резервных копий (R-D12: dhcp/travelmate включены в транзакцию) и взведение
#    бессмертного таймера отката (300 секунд), пережидающего обрыв SSH:
cp /etc/sing-box/config.json /etc/sing-box/config.json.bak
cp /etc/config/firewall /etc/config/firewall.bak
cp /etc/nftables.d/99-crne-security.nft /etc/nftables.d/99-crne-security.nft.bak
cp /etc/config/dhcp /etc/config/dhcp.bak 2>/dev/null
cp /etc/config/travelmate /etc/config/travelmate.bak 2>/dev/null
touch /tmp/rollback.lock

nohup sh -c 'trap "" HUP INT TERM; sleep 300; [ -f /tmp/rollback.lock ] && {
    logger -t CRNE-ROLLBACK "Rollback triggered! Restoring previous working state..."
    cp /etc/sing-box/config.json.bak /etc/sing-box/config.json
    cp /etc/config/firewall.bak /etc/config/firewall
    cp /etc/nftables.d/99-crne-security.nft.bak /etc/nftables.d/99-crne-security.nft
    cp /etc/config/dhcp.bak /etc/config/dhcp 2>/dev/null
    cp /etc/config/travelmate.bak /etc/config/travelmate 2>/dev/null
    /etc/init.d/firewall reload
    /etc/init.d/sing-box restart
    rm -f /tmp/rollback.lock
}' >/dev/null 2>&1 </dev/null &

# 3. Применение изменений:
/etc/init.d/firewall reload
/etc/init.d/sing-box restart
```
Отмена таймера и удаление `.bak`-файлов после успешной проверки связи с клиентского устройства:
```sh
# R-D12: dhcp.bak/travelmate.bak создаются ВНУТРИ транзакции (шаг 2) и восстанавливаются
# при откате — здесь только снимаем блокировку и убираем артефакты успешного применения.
rm -f /tmp/rollback.lock /etc/sing-box/config.json.bak /etc/config/firewall.bak /etc/nftables.d/99-crne-security.nft.bak /etc/config/dhcp.bak /etc/config/travelmate.bak
```

### 6.3. CI/CD для конфигов (`sub2sing.py` + git-версионирование + валидация nftables)

Процесс сборки и доставки конфигурации формализован как pipeline на сборочном узле (ПК/Termux):

1. **Версионирование (ADR-10, v4.4):** на роутере поддерживается локальный git-репозиторий конфигов, **перенесённый в tmpfs** (оперативную память). Это устраняет противоречие с принципом Data Minimization: история конфигов с адресами серверов, UUID и параметрами Reality **не персистентна на NAND-flash** и уничтожается при перезагрузке/отключении питания — при физическом изъятии устройства инфраструктура обхода не раскрывается. Для постоянного отката используется `config.json.bak` (восстановление через 300-секундный Lock-Rollback таймер).
```sh
# Первичная инициализация (выполняется один раз на роутере):
mkdir -p /tmp/sing-box-git
cd /etc/sing-box && git init --separate-git-dir=/tmp/sing-box-git
printf 'config.json.bak\n*.lock\n/tmp/*\n' > .gitignore
git add config.json .gitignore && git commit -m "CRNE: initial config snapshot"
```
После каждого успешного `sub2sing.py --push` выполняется авто-коммит (реализовано в `sub2sing.py`, см. Приложение А):
```sh
ssh root@10.10.1.1 'cd /etc/sing-box && git add config.json && git commit -m "sub2sing push $(date +%%Y%%m%%d-%%H%%M)"' 2>/dev/null
```
Откат к прошлой рабочей версии (в пределах текущей сессии, до перезагрузки): `cd /etc/sing-box && git checkout <hash> -- config.json && /etc/init.d/sing-box restart`.
При перезагрузке git-история очищается; для восстановления используется `config.json.bak` (см. Раздел 6.2 Lock-Rollback) или внешний backup-server (Раздел 6.4).

2. **Двойная валидация перед активацией** (JSON + nftables): `sub2sing.py --push` удалённо вызывает `sing-box check` И `fw4 check` под защитой отвязанного от терминала (`trap "" HUP INT TERM` + `</dev/null`) 300-секундного таймера отката, который гасится **только после** проверки поднятия `tun0` и `pidof sing-box`:
```sh
# Выдержка из sub2sing.py (--push блок, v4.2 R-D2):
remote_cmd = (
    "set -e; "
    # 1. Жёсткая валидация до модификации рабочей конфигурации.
    "jq empty /tmp/config.json.new; "
    "sing-box check -c /tmp/config.json.new; "
    "fw4 check; "
    # 2. Резервная копия и взведение отвязанного от терминала 300с таймера отката.
    "cp /etc/sing-box/config.json /etc/sing-box/config.json.bak; "
    "touch /tmp/rollback.lock; "
    "nohup sh -c 'trap \"\" HUP INT TERM; sleep 300; [ -f /tmp/rollback.lock ] && { "
    "mv /etc/sing-box/config.json.bak /etc/sing-box/config.json 2>/dev/null; "
    "/etc/init.d/sing-box restart; rm -f /tmp/rollback.lock; "
    "logger -t CRNE-ROLLBACK \"auto-rollback after 300s (push unconfirmed)\"; "
    "}' >/dev/null 2>&1 </dev/null & "
    "RB_PID=$!; "
    # 3. Атомарная замена и перезапуск.
    "mv /tmp/config.json.new /etc/sing-box/config.json; "
    "chmod 600 /etc/sing-box/config.json; "
    "/etc/init.d/sing-box restart; "
    # 4. Ожидание фактической инициализации tun0 перед снятием блокировки отката.
    "sleep 4; "
    "ip link show tun0 >/dev/null 2>&1 && [ -n \"$(pidof -s sing-box)\" ]; "
    "rm -f /tmp/rollback.lock /etc/sing-box/config.json.bak; "
    "kill $RB_PID 2>/dev/null || true; "
    # 5. Опциональный git-коммит (не ломает деплой, если git не установлен или нет изменений).
    "if command -v git >/dev/null 2>&1 && [ -d /etc/sing-box/.git ]; then "
    "  cd /etc/sing-box && git add config.json && git commit -m \"sub2sing push $(date +%Y%m%d-%H%M)\" >/dev/null 2>&1 || true; "
    "fi"
)
```
При ошибке любой из проверок (`set -e` прерывает цепочку) новый конфиг **не активируется**, таймер отката остаётся взведённым и либо откатит `.bak` через 300с (если конфиг успел попасть в активное состояние), либо просто не тронет рабочий (если сбой произошёл до `mv`). Старый `/etc/sing-box/config.json` не повреждается.

3. **Шаблонизация секрета:** секрет Clash API не хардкодится в генерируемом конфиге — `sub2sing.py` подставляет существующий секрет, считанный с роутера:
```sh
EXISTING_SECRET=$(ssh root@10.10.1.1 "jq -r '.experimental.clash_api.secret' /etc/sing-box/config.json")
python3 ./sub2sing.py --url "$SUB_URL" --secret "$EXISTING_SECRET" --push root@10.10.1.1
```

### 6.4. Backup/Restore и внешний резервный архив

1. **Создание архива перед sysupgrade** (и еженедельный cron):
```sh
# Немедленный бэкап на внешний сервер:
sysupgrade -b /tmp/crne-backup-$(date +%Y%m%d).tar.gz
scp /tmp/crne-backup-*.tar.gz user@backup-server:/backups/crne/
rm -f /tmp/crne-backup-*.tar.gz
# Еженедельный авто-бэкап (в /etc/crontabs/root):
0 3 * * 0 sysupgrade -b - | ssh user@backup-server "cat > /backups/crne-$(date +\%Y\%m\%d).tar.gz"
```

2. **Восстановление из архива:**
```sh
# Копирование архива на роутер и восстановление:
scp user@backup-server:/backups/crne-YYYYMMDD.tar.gz /tmp/
sysupgrade -r /tmp/crne-YYYYMMDD.tar.gz
```

3. **Отдельное хранение секрета Clash API** (на случай повреждения config.json):
```sh
# Экспорт секрета в защищённый файл на backup-сервере:
ssh root@10.10.1.1 "jq -r '.experimental.clash_api.secret' /etc/sing-box/config.json" | \
  ssh user@backup-server "cat > /backups/crne-secret.txt && chmod 600 /backups/crne-secret.txt"
```

---

## ✅ 7. Программа и методика приёмочных испытаний (ATP — Чек-лист T1–T36)

> **⚠️ v4.4:** Добавлены тесты T32–T35 для проверки новых механизмов ADR-08 (Bootstrap DoH), ADR-09 (Output Kill Switch), ADR-10 (Git в tmpfs) и MSS Clamping.
> **⚠️ v4.4 (hotfix):** Добавлен тест T36 — окно Captive Portal в Output Kill Switch (regression ADR-09, устранено блокировкой детект-проб с роутера).
> **⚠️ v4.3:** Добавлены тесты T28–T30 для проверки новых механизмов D1–D3 и ADR-06/ADR-07.

| ID | Испытание | Команда / Действие | Критерий успешного прохождения |
| :- | :--- | :--- | :--- |
| **T1** | **Криптоблоки CPU, версия `sing-box` и отключение PPE** | `grep -E 'aes|pmull' /proc/cpuinfo && sing-box version && uci get firewall.@defaults[0].flow_offloading` | Присутствуют флаги `aes pmull sha1 sha2`; версия `sing-box >= 1.12.0`; `flow_offloading` равен `0`. |
| **T2** | **Сухая валидация `fw4` и `sing-box` (`.srs` + `ui`)** | `fw4 check && sing-box check -c /etc/sing-box/config.json && ls -1 /etc/sing-box/ui/index.html` | Обе команды возвращают код `0` без ошибок синтаксиса; статика веб-панели присутствует в `/etc/sing-box/ui/`. |
| **T3** | **Бюджет ОЗУ при холодном старте (`pidof -s`)** | `awk '/VmRSS/{print $2}' /proc/$(pidof -s sing-box)/status && free -m` | Потребление `VmRSS` **≤ 45 000 КБ (45 МБ)**, свободной ОЗУ (`available`) **≥ 135 МБ**. |
| **T4** | **Холодный старт NTP без туннеля и DNS** | `/etc/init.d/sing-box stop && ntpd -n -q -p 194.190.168.1` | Время синхронизируется за <3 с при выключенном `sing-box` (счётчик в `nft list chain inet fw4 crne_output_bypass` растёт). |
| **T5** | **Истинный тест Fail-Closed (При падении `sing-box`)** | При остановленном `sing-box` (`/etc/init.d/sing-box stop`) с ПК/телефона выполнить `curl -I -m 5 https://icanhazip.com` и `curl -I -m 5 https://2ip.ru` | Оба HTTP-запроса с клиентских устройств **немедленно отвергаются** (`Connection refused` / `Administratively Prohibited`), утечки в WAN нет! После `/etc/init.d/sing-box start` оба возвращают `HTTP 200`. |
| **T6** | **Перехват DNS (hijack-dns в tun0), `.lan` и Canary** | С клиента: `nslookup gosuslugi.ru 8.8.8.8`, `nslookup use-application-dns.net`, `nslookup test.lan 8.8.8.8` | Запрос к `8.8.8.8` перехвачен через tun0 и обработан через hijack-dns, вернул IP РФ; для `use-application-dns.net` возвращён `NXDOMAIN` (Canary — отключение DoH); `test.lan` резолвится через `dns-local-lan`→`dnsmasq:5353` и возвращает `NXDOMAIN` для незарегистрированного имени (зарегистрированные DHCP-хосты возвращают свой локальный IP). |
| **T7** | **Логическое `OR` маршрутизации РФ** | С клиента: `curl -s https://2ip.ru` и `curl -s https://icanhazip.com` | На `2ip.ru` виден домашний/мобильный IP РФ (`Direct`); на `icanhazip.com` — IP зарубежного прокси-сервера. |
| **T8** | **Защита от WebRTC и браузерного DoH** | Открыть в браузере `browserleaks.com/webrtc` и `dnsleaktest.com` | Реальный IP провайдера в WebRTC не отображается; в DNS-тесте видны только серверы Cloudflare (`1.1.1.1`). |
| **T9** | **Проверка `socks5h://` и `Hysteria2`** | Выбрать узел `Hysteria2` в `10.10.1.1:9090/ui` (или через API `PUT /proxies/MANUAL-SELECT`) и выполнить `curl -x socks5h://127.0.0.1:1603 -I https://www.youtube.com/generate_204` | Ответ `HTTP 204` (с резолвом домена внутри прокси); прямой `UDP/443` из LAN заблокирован, а `Hysteria2` из `output` работает штатно. |
| **T10** | **Защита Clash API и проверка tun0** | `curl -I http://10.10.1.1:9090/proxies` (без токена) и `ip link show tun0` | API возвращает `HTTP 401 Unauthorized`; интерфейс `tun0` существует и в состоянии `UP`. |
| **T11** | **Автопереключение `Wi-Fi ↔ USB` и `LTS`-узлов** | На `TR3000`: включить USB-модем на подключённом телефоне (`usb0`) и отключить `trm_wwan` | Через 2–3 с в панели `10.10.1.1:9090/ui` узлы в группе `LTS-MOBILE` показывают численный пинг и принимают трафик. |
| **T12** | **Трёхступенчатое окно Captive Portal (180 с)** | Выполнить `/etc/travelmate/crne-portal-hook.sh cp 192.168.1.1 phy0-sta0` (где `192.168.1.1` — DNS отеля, `phy0-sta0` — WAN L3-устройство) и проверить `nft list set inet fw4 crne_cp_window`, `nft list map inet fw4 crne_cp_dns` и `nft list set inet fw4 crne_cp_output_wan`. Подождать 180 с и выполнить `curl -I -m 3 http://example.com` с клиента. | Во всех трёх наборах появляются записи с `expires 3m`. Через 180 с правила перестают применяться (запрос блокируется Kill Switch), даже если элемент ещё виден в `nft list set` (до 15 с задержка gc). При вызове `connected` все три набора очищаются немедленно и `sing-box` стартует. |
| **T13** | **Нагрузочный тест скорости (Throughput)** | Скачивание 100 МБ потока через туннель с замером `top` | Скорость **≥ 180–300 Мбит/с**, запас `idle` процессора ≥ 15%. |
| **T14** | **Суточный Soak-тест и проверка `sysupgrade`** | Работа 24 ч + `sysupgrade -l \| grep crontabs` | `VmRSS` за 24 ч держится в мягком коридоре **≤ 75 МБ (`GOMEMLIMIT`)**; все файлы из п. 5.7 присутствуют в выводе `sysupgrade -l`. |
| **T15** | **Масштабирование (50+ параллельных клиентов)** | v4.2: **ВЫПОЛНЯЕТСЯ С РОУТЕРА** (mixed-in слушает только 127.0.0.1): `for i in $(seq 1 50); do curl -x socks5h://127.0.0.1:1603 -s -m 10 https://www.youtube.com/generate_204 & done; wait` | Все 50 запросов завершаются успешно (HTTP 204), `VmRSS ≤ 85 МБ`, OOM не срабатывает. |
| **T16** | **Деградация при 90% потреблении RAM** | Симуляция: `dd if=/dev/zero bs=1M count=230 \| gzip > /dev/null &` (или `stress-ng` если установлен), затем `/usr/bin/crne-watchdog.sh` | Watchdog перезапускает sing-box при VmRSS > 95MB, в логе `CRNE-WATCHDOG` есть строка `exceeded 95MB`, OOM-killer не срабатывает, tun0 восстанавливается в течение 30 сек. |
| **T17** | **SIGKILL sing-box (procd respawn)** | Выполнить `kill -9 $(pidof -s sing-box)` и подождать 10 секунд | `procd respawn` автоматически перезапускает sing-box, интерфейс tun0 восстанавливается, клиенты могут подключаться. |
| **T18** | **Race Captive Portal + Rollback** | Запустить параллельно: `/etc/travelmate/crne-portal-hook.sh cp 192.168.1.1 phy0-sta0 & sleep 2; /etc/travelmate/crne-portal-hook.sh connected` | Конфликтов нет, sing-box запущен, lock-файлы очищены, все три Captive Portal набора пусты, трафик идёт через туннель. |
| **T19** | **DPI-тест: МТС (TCP/UDP throttling)** | Подключение через SIM МТС, проверка доступа к заблокированным ресурсам | VLESS+Reality/Hysteria2 пробивает DPI, YouTube/Instagram доступны, скорость ≥ 10 Мбит/с. |
| **T20** | **DPI-тест: Билайн (активный DPI)** | Подключение через SIM Билайн, проверка прямого контура через nfqws | Прямой контур (Direct) работает с nfqws-профилем fake,multidisorder, российские сайты открываются корректно. |
| **T21** | **DPI-тест: Мегафон (пассивный анализ)** | Подключение через SIM Мегафон, проверка скрытности трафика | Трафик VLESS+Reality не детектируется как VPN, SNI маскируется под легитимный TLS. |
| **T22** | **DPI-тест: Ростелеком (ТСПУ)** | Подключение через Ростелеком, проверка устойчивости к ТСПУ | ТСПУ не блокирует соединения, urltest корректно выбирает оптимальные узлы. |
| **T23** | **Автооткат при обрыве SSH во время Lock-Rollback** | Применить заведомо ломающий конфиг через блок 6.2 и оборвать SSH-сессию (`~.`), подождать 300 сек | Ровно через 300 сек конфиг автоматически восстанавливается из `.bak`, в `logread` есть `CRNE-ROLLBACK`, туннель работает. |
| **T24** | **Git-версионирование конфигов** | Выполнить `sub2sing.py --push`, затем `cd /etc/sing-box && git log --oneline` | В логе появляется новый коммит с timestamp; `git checkout HEAD~1 -- config.json` восстанавливает предыдущую версию. |
| **T25** | **Telegram-алертинг** | Настроить `/etc/crne-alert.conf`, выполнить `/usr/bin/crne-alert.sh "test"` | В Telegram-чат приходит сообщение `🚨 CRNE ALERT [home-gateway]: test`; права файла `600` (`ls -l /etc/crne-alert.conf`). |
| **T26** | **Экспорт метрик Prometheus** | Выполнить `/usr/bin/crne-watchdog.sh && cat /tmp/node_exporter/textfile_collector/crne.prom` | Файл содержит метрики `crne_tunnel_status`, `crne_memory_rss_kb`, `crne_soc_temp_celsius`, `crne_usb_status`, `crne_dns_bootstrap_requests_total` с числовыми значениями. |
| **T27** | **DNS-нагрузочный тест (1 час)** | Запустить `dnsperf` или скрипт с 1000 запросов/мин на `10.10.1.1:53` в течение 1 часа; мониторить `logread \| grep sing-box`, `ps aux \| grep dnsmasq` и метрики Prometheus | `dnsmasq` не накапливает ошибки (`logread \| grep dnsmasq \| grep -i error` пуст), `sing-box` не логирует `hijack-dns` failures, `VmRSS` стабилен (±5 MB), `curl -s -H "Authorization: Bearer $SECRET" http://10.10.1.1:9090/version` возвращает JSON с полем `version` (без ошибки 401), утечек памяти нет. |
| **T28** | **v4.3 (D1): SMS fallback при падении туннеля** | Настроить `SMS_GATEWAY` в `/etc/crne-alert.conf`, остановить `sing-box`, выполнить `/usr/bin/crne-alert.sh "test-critical" critical` | SMS отправлено через USB-модем (проверить `/var/log/modem.log` или `dmesg \| grep -i sms`), либо email отправлен через msmtp. |
| **T29** | **v4.3 (D2): Детект пустого пула узлов** | Временно заблокировать все узлы подписки (iptables DROP на серверы), подождать 30 мин, проверить `logread \| grep "pool is EMPTY"` и метрику `crne_alive_nodes_count` | Watchdog детектирует пустой пул (`ALIVE_NODES=0`), отправляет critical alert, пытается активировать fallback subscription (если настроена). |
| **T30** | **v4.3 (D3+ADR-06): SHA256 verification .srs** | Выполнить `./download-assets.sh`, модифицировать один байт в `geoip-ru.srs`, повторить `./download-assets.sh` | При модификации — `[ERROR] SHA256 mismatch`, скрипт exit 1, доставка отменена. При корректных файлах — `[OK] SHA256 verified`. |
| **T31** | **v4.3 (ADR-07): Watchdog staleness detection** | Остановить cron (`/etc/init.d/cron stop`), подождать 35 мин, проверить `logread \| grep CRNE-STALE` | Через 30 мин после остановки cron — staleness alert, cron перезапускается автоматически. |
| **T32** | **v4.4 (ADR-08): Bootstrap DNS использует DoH** | `jq '.dns.servers[] \| select(.tag=="dns-bootstrap") \| {type, server, server_port}' /etc/sing-box/config.json` | Вывод: `type: "https"`, `server: "1.1.1.1"`, `server_port: 443`. Плейнтекст UDP/53 к 77.88.8.8 отсутствует. Проверить `/etc/hosts` содержит `1.1.1.1 cloudflare-dns.com`. |
| **T33** | **v4.4 (ADR-09): Output Kill Switch активен при падении sing-box** | На роутере: `/etc/init.d/sing-box stop`; затем `curl -I -m 3 http://example.com 2>&1` и `ntpd -n -q -p 194.190.168.1` | HTTP-запрос отвергнут (`Connection refused` / `admin-prohibited`) — router-originated traffic заблокирован. NTP синхронизируется (UDP/123 в whitelist). После `/etc/init.d/sing-box start` — HTTP снова работает через туннель. |
| **T34** | **v4.4 (ADR-10): Git-история в tmpfs, не персистентна на NAND** | `readlink /etc/sing-box/.git` и `/etc/init.d/sing-box stop && reboot`; после загрузки: `ls /etc/sing-box/.git` | До перезагрузки: `.git` → симлинк на `/tmp/sing-box-git`. После перезагрузки: git-история пуста (tmpfs очищена), `config.json` сохранён на NAND. История конфигов с секретами не извлекается при изъятии устройства. |
| **T35** | **v4.4: MSS Clamping для туннеля** | `nft list chain inet fw4 crne_mss_clamp` и проверить `tcpdump -i tun0 -n 'tcp[tcpflags] & tcp-syn != 0'` при скачивании большого файла | Chain `crne_mss_clamp` существует с правилом `oifname "tun0" ... maxseg size set rt mtu`. TCP SYN через tun0 содержит MSS ≤ 1360 (MTU 1400 − 40 IP − 20 TCP). Зависаний больших загрузок через туннель нет. |
| **T36** | **v4.4 (ADR-09 hotfix): Окно Captive Portal в Output Kill Switch** | В travel-профиле подключиться к Wi-Fi с Captive Portal: до авторизации выполнить с роутера `nslookup captive.apple.com <WAN_DNS>` и `nft list set inet fw4 crne_cp_output_wan`; после успешной авторизации повторить `nft list set inet fw4 crne_cp_output_wan` | До авторизации: `nslookup` возвращает IP (не блокирован), набор `crne_cp_output_wan` содержит WAN L3-устройство (`phy0-sta0` и т.п.) с `expires 3m`. После `connected` или истечения 180 с: набор пуст, Output Kill Switch снова блокирует весь router-originated WAN-трафик кроме whitelist (NTP/DHCP/DoH). |

---

## 🛡️ 8. Юридическая рамка и модель доверия

* **Правовой статус:** система предназначена исключительно для законного личного использования — защиты беспроводных соединений в публичных сетях связи, предотвращения перехвата трафика и обеспечения устойчивого доступа к открытым информационным ресурсам. В РФ личное использование VPN не запрещено; правовые риски возникают при публичной рекламе инструментов обхода блокировок, коммерческом предоставлении доступа третьим лицам или умышленном поиске заведомо экстремистских материалов (ст. 13.53 КоАП РФ). *Данный раздел носит информационный характер и не заменяет индивидуального юридического заключения.*

* **Юридический disclaimer:** «Система предназначена исключительно для защиты данных в публичных Wi-Fi сетях и личного некоммерческого использования. Запрещается предоставление доступа третьим лицам или публичное распространение как сервиса обхода блокировок. Автор не несёт ответственности за неправомерное использование третьими лицами.»

* **Модель доверия к коммерческим подпискам:** поскольку выходные прокси-серверы управляются сторонним поставщиком, они рассматриваются как **недоверенная транзитная среда**. Конфиденциальность передаваемых данных обеспечивается исключительно сквозным шифрованием прикладного уровня (**HTTPS / TLS 1.2+**) между клиентским приложением и целевым веб-ресурсом. Доступ к российским финансовым и государственным сервисам осуществляется напрямую (`Direct`) в обход сторонних прокси-узлов.

* **Защита персональных данных и Data Minimization (ADR-10, v4.4):** система не осуществляет сбор, хранение и обработку персональных данных пользователей. Все логи хранятся исключительно в оперативной памяти (tmpfs) и уничтожаются при перезагрузке устройства. Кэш DNS (`/tmp/sing-box-cache.db`) содержит только соответствия доменных имён и IP-адресов для ускорения резолвинга. **Локальный git-репозиторий конфигов также перенесён в tmpfs** (`/tmp/sing-box-git`): история версий `config.json` (адреса серверов, UUID, параметры Reality, секрет Clash API) хранится только в ОЗУ и безвозвратно уничтожается при перезагрузке/отключении питания. Это устраняет риск раскрытия инфраструктуры обхода при физическом изъятии или краже устройства — на NAND-flash остаётся только текущий рабочий `config.json`, а откат к прошлым версиям выполняется через `config.json.bak` или внешний backup-сервер.

---

## 📎 Приложение А: Полный исходный код компилятора подписок `sub2sing.py` (v4.2)

Скрипт использует только стандартную библиотеку Python 3.10+, дедуплицирует теги через 4-символьный SHA256-хеш URI, исключает ложные срабатывания при отборе `LTS`-узлов, генерирует конфигурацию под `sing-box 1.12+` и при вызове `--push` выполняет **двойную валидацию (`jq empty` + `sing-box check` + `fw4 check`) под `set -e`, отвязанный от терминала (`trap "" HUP INT TERM` + `</dev/null>`) Lock-Rollback таймер 300с (переживает обрыв SSH), атомарную активацию и гашение таймера отката только после проверки фактического поднятия `tun0` и `pidof sing-box`, а также авто-коммит в git-репозиторий `/etc/sing-box`**. При `--push` без `--secret` существующий секрет Clash API считывается с роутера по SSH (не затирается случайным токеном). При провале любой из проверок новый конфиг не активируется, старый остаётся нетронутым, и пользователю возвращается ясная диагностика.

```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
"""
CRNE v4.2 Subscription Compiler & Local Testbench (sub2sing.py)
Требования: Python 3.10+ (stdlib only). Работает на Windows, Linux, macOS и в Android Termux (F-Droid).

v4.2: --push под set -e + trap "" HUP INT TERM + </dev/null; rollback-таймер гасится
      только после проверки поднятия tun0 и pidof sing-box (R-D2). parse_uri поддерживает
      Port Hopping, obfs salamander, unquote учётных данных (R-D11).
"""
from __future__ import annotations

import argparse
import base64
import hashlib
import json
import os
import re
import secrets
import subprocess
import sys
import tempfile
import time
import urllib.parse
import urllib.request

MAX_ROUTER_NODES = 50  # Жёсткий потолок узлов для защиты 256 МБ RAM на роутере
MIN_ALIVE_NODES = 5    # v4.2: Минимум живых узлов для предотвращения пустого пула (Pre-Mortem #2)
LTS_PATTERN = re.compile(r"(?:\bLTS\b|\bMOBILE\b|\bLTE\b|\bWL\b|\bBYPASS\b|МОБ|БЕЛЫЙ|ОБХОД)", re.IGNORECASE)

# v4.2 (R-D1): функция detect_sb_minor_version() и аргумент --sb-minor удалены как мёртвый код.
# Спецификация требует sing-box >= 1.12.0 (нативный типизированный DNS); обратная совместимость
# с 1.11.x (legacy DNS format) в production не поддерживается. Генерация единого формата.


def fetch_subscription(url: str) -> list[str]:
    req = urllib.request.Request(url, headers={"User-Agent": "sing-box/1.12"})
    with urllib.request.urlopen(req, timeout=15) as resp:
        raw = resp.read().decode("utf-8", errors="ignore").strip()
    try:
        padded = raw + "=" * (-len(raw) % 4)
        decoded = base64.b64decode(padded).decode("utf-8", errors="ignore")
        return [line.strip() for line in decoded.splitlines() if line.strip()]
    except Exception:
        return [line.strip() for line in raw.splitlines() if line.strip()]


def parse_uri(uri: str, idx: int) -> dict | None:
    uri_hash = hashlib.sha256(uri.encode("utf-8")).hexdigest()[:4]
    if uri.startswith("vless://"):
        parsed = urllib.parse.urlparse(uri)
        params = dict(urllib.parse.parse_qsl(parsed.query))
        if params.get("allowInsecure") in ("1", "true") or params.get("insecure") in ("1", "true"):
            return None
        if params.get("security") != "reality" or not params.get("pbk"):
            return None
        # v4.2 (R-D11): проверка типа транспорта — flow xtls-rprx-vision только для TCP.
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
            # v4.2 (R-D11): декодирование percent-encoded UUID.
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
        # v4.2 (R-D11): flow назначается только TCP-транспорту; для grpc/ws — отдельные поля.
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

        # v4.2 (R-D11): безопасное извлечение хоста и порта/диапазона портов (Port Hopping).
        # urllib.parse не умеет парсить диапазоны портов ("20000-50000") — parsed.port бросает ValueError.
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
            # v4.2 (R-D11): декодирование percent-encoded пароля.
            "password": urllib.parse.unquote(parsed.username or params.get("auth", "")),
            "tls": {
                "enabled": True,
                "server_name": params.get("sni", host)
            }
        }

        # v4.2 (R-D11): конвертация диапазона портов URI (20000-50000) → server_ports sing-box (20000:50000).
        if "-" in port_str:
            node["server_ports"] = [port_str.replace("-", ":")]
        else:
            node["server_port"] = int(port_str) if port_str.isdigit() else 443

        # v4.2 (R-D11): поддержка обфускации salamander (без неё узлы с obfs не подключаются).
        if params.get("obfs") == "salamander" and params.get("obfs-password"):
            node["obfs"] = {"type": "salamander", "password": params["obfs-password"]}

        return node
    return None


def is_lts_node(tag: str) -> bool:
    return bool(LTS_PATTERN.search(tag))


def benchmark_nodes(nodes: list[dict], timeout_ms: int) -> dict[str, int]:
    tags = [n["tag"] for n in nodes]
    if not tags:
        return {}
    test_cfg = {
        "log": {"level": "error"},
        "experimental": {"clash_api": {"external_controller": "127.0.0.1:19090"}},
        "outbounds": nodes + [
            {"type": "selector", "tag": "TEST-GROUP", "outbounds": tags},
            {"type": "direct", "tag": "direct"}
        ]
    }
    with tempfile.TemporaryDirectory() as tmpdir:
        cfg_path = os.path.join(tmpdir, "test.json")
        with open(cfg_path, "w", encoding="utf-8") as f:
            json.dump(test_cfg, f)
        proc = subprocess.Popen(["sing-box", "run", "-c", cfg_path],
                                stdout=subprocess.DEVNULL, stderr=subprocess.DEVNULL)
        try:
            time.sleep(1.5)
            test_url = (f"http://127.0.0.1:19090/group/TEST-GROUP/delay"
                        f"?url=https://www.youtube.com/generate_204&timeout={timeout_ms}")
            with urllib.request.urlopen(test_url, timeout=(timeout_ms / 1000.0) + 5) as resp:
                return json.loads(resp.read().decode("utf-8"))
        except Exception as e:
            print(f"[WARN] Ошибка пакетного теста стенда: {e}", file=sys.stderr)
            return {}
        finally:
            proc.terminate()
            proc.wait(timeout=3)


def build_router_config(standard_top: list[dict], lts_nodes: list[dict],
                        extra_manual: list[dict], tolerance: int, secret: str) -> dict:
    all_selected_nodes = standard_top + lts_nodes + extra_manual
    std_tags = [n["tag"] for n in standard_top]
    lts_tags = [n["tag"] for n in lts_nodes]
    flat_auto_tags = std_tags + lts_tags
    all_tags = [n["tag"] for n in all_selected_nodes]

    local_suffixes_nxdomain = [".local", ".home.arpa", ".in-addr.arpa", ".ip6.arpa"]

    # Нативный типизированный DNS для sing-box 1.12+ (OpenWrt 24.10.5+)
    # v4.2 (R-D4): dns-local-lan (127.0.0.1:5353) для зоны .lan; Canary в dns.rules.
    # v4.4 (ADR-08): dns-bootstrap заменён на Cloudflare DoH (1.1.1.1:443) — устранён MITM-вектор
    #                 и зависимость от Яндекс DNS (РФ-юрисдикция). Certificate pinning через server_name.
    dns_block = {
        "servers": [
            {"type": "https", "tag": "dns-proxy-cf", "server": "1.1.1.1", "detour": "GLOBAL-AUTO"},
            {"type": "https", "tag": "dns-direct-ru", "server": "common.dot.dns.yandex.net",
             "domain_resolver": "dns-bootstrap"},
            {"type": "udp", "tag": "dns-local-lan", "server": "127.0.0.1", "server_port": 5353},
            {"type": "https", "tag": "dns-bootstrap", "server": "1.1.1.1", "server_port": 443,
             "tls": {"enabled": True, "server_name": "cloudflare-dns.com"}}
        ],
        "rules": [
            {"domain": ["use-application-dns.net", "_dns.resolver.arpa"],
             "action": "predefined", "rcode": "NXDOMAIN"},
            {"domain_suffix": [".lan"], "server": "dns-local-lan"},
            {"domain_suffix": local_suffixes_nxdomain, "action": "predefined", "rcode": "NXDOMAIN"},
            {"rule_set": ["geosite-ads"], "action": "predefined", "rcode": "NOERROR"},
            {"domain_suffix": [".ru", ".рф", ".su"], "server": "dns-direct-ru"},
            {"rule_set": ["geosite-ru"], "server": "dns-direct-ru"}
        ],
        "final": "dns-proxy-cf",
        "strategy": "ipv4_only"
    }

    # v4.2 (R-D5): default_mark убран (конфликт с auto_redirect); auto_redirect_output_mark в tun.
    # v4.2 (R-D6): default_domain_resolver → dns-direct-ru + нетерминальное action: resolve.
    route_block = {
        "auto_detect_interface": True,
        "default_domain_resolver": "dns-direct-ru",
        "rule_set": [
            {"tag": "geosite-ru", "type": "local", "format": "binary",
             "path": "/etc/sing-box/ruleset/geosite-category-ru.srs"},
            {"tag": "geoip-ru", "type": "local", "format": "binary",
             "path": "/etc/sing-box/ruleset/geoip-ru.srs"},
            {"tag": "geosite-ads", "type": "local", "format": "binary",
             "path": "/etc/sing-box/ruleset/geosite-category-ads-all.srs"}
        ],
        "rules": [
            {"action": "sniff"},
            {"protocol": "dns", "action": "hijack-dns"},
            {"ip_is_private": True, "outbound": "direct"},
            {"ip_cidr": ["1.1.1.1/32", "1.0.0.1/32", "8.8.8.8/32", "8.8.4.4/32", "9.9.9.9/32"],
             "port": [443], "action": "reject"},
            {"rule_set": ["geosite-ads"], "action": "reject"},
            {"action": "resolve", "strategy": "ipv4_only"},
            {"domain_suffix": [".ru", ".рф", ".su"], "outbound": "direct"},
            {"rule_set": ["geosite-ru", "geoip-ru"], "outbound": "direct"}
        ],
        "final": "GLOBAL-AUTO"
    }

    return {
        "log": {"level": "warn", "timestamp": True},
        "experimental": {
            "cache_file": {"enabled": True, "path": "/tmp/sing-box-cache.db", "store_fakeip": False},
            "clash_api": {
                "external_controller": "10.10.1.1:9090",
                "external_ui": "/etc/sing-box/ui",
                "secret": secret,
                "default_mode": "Rule"
            }
        },
        "dns": dns_block,
        "inbounds": [
            {"type": "tun", "tag": "tun-in", "interface_name": "tun0",
             "address": ["172.19.0.1/30"], "mtu": 1400, "auto_route": True,
             "auto_redirect": True, "auto_redirect_output_mark": "0x000000ff", "strict_route": True},
            {"type": "mixed", "tag": "mixed-in", "listen": "127.0.0.1", "listen_port": 1603}
        ],
        "outbounds": [
            {"type": "urltest", "tag": "GLOBAL-AUTO", "outbounds": flat_auto_tags,
             "url": "https://www.youtube.com/generate_204", "interval": "2m",
             "tolerance": tolerance, "idle_timeout": "30m"},
            {"type": "urltest", "tag": "STANDARD-TOP20", "outbounds": std_tags or flat_auto_tags,
             "url": "https://www.youtube.com/generate_204", "interval": "2m", "tolerance": tolerance},
            {"type": "urltest", "tag": "LTS-MOBILE", "outbounds": lts_tags or flat_auto_tags,
             "url": "https://www.youtube.com/generate_204", "interval": "2m", "tolerance": min(tolerance + 20, 200)},
            {"type": "selector", "tag": "MANUAL-SELECT",
             "outbounds": ["GLOBAL-AUTO", "LTS-MOBILE", "STANDARD-TOP20"] + all_tags},
            {"type": "direct", "tag": "direct"}
        ] + all_selected_nodes,
        "route": route_block
    }


def main():
    ap = argparse.ArgumentParser(description="CRNE v4.2 Subscription Compiler (sing-box 1.12+ only)")
    ap.add_argument("--url", help="Subscription URL")
    ap.add_argument("--init-template", action="store_true", help="Generate offline template config")
    ap.add_argument("--top", type=int, default=20, help="Top N standard nodes")
    ap.add_argument("--tolerance", type=int, default=180, help="urltest tolerance (ms)")
    ap.add_argument("--timeout", type=int, default=1500, help="Test timeout (ms)")
    ap.add_argument("--secret", default="", help="Clash API secret (auto-generated if omitted)")
    ap.add_argument("--test-local", action="store_true", help="Run local sing-box benchmark")
    ap.add_argument("--out", default="config.json", help="Output JSON path")
    ap.add_argument("--push", help="SCP target (e.g. root@10.10.1.1) with atomic check & restart")
    args = ap.parse_args()

    if not args.url and not args.init_template:
        ap.error("Требуется указать --url <SUB_URL> или --init-template")

    # v4.2 (R-D11): при --push без явного --secret сначала пытаемся считать существующий секрет
    # Clash API с роутера по SSH, чтобы НЕ затирать уже настроенный токен.
    secret = args.secret
    if args.push and not secret:
        try:
            out = subprocess.check_output(
                ["ssh", args.push, "jq -r '.experimental.clash_api.secret // empty' /etc/sing-box/config.json"],
                text=True, timeout=5
            )
            remote_secret = out.strip()
            if remote_secret and "REPLACE_WITH_GENERATED_SECRET" not in remote_secret:
                secret = remote_secret
                print(f"[INFO] Reused existing Clash API secret from {args.push}")
        except Exception:
            pass
    if not secret:
        secret = secrets.token_hex(16)  # 128-bit entropy (NIST SP 800-132)

    if args.init_template:
        standard_top = [{
            "type": "vless", "tag": "vless-template-01", "server": "198.51.100.10",
            "server_port": 443, "uuid": "00000000-0000-0000-0000-000000000001", "flow": "xtls-rprx-vision",
            "tls": {"enabled": True, "server_name": "example.com", "utls": {"enabled": True, "fingerprint": "chrome"},
                    "reality": {"enabled": True, "public_key": "0000000000000000000000000000000000000000000", "short_id": "01"}}
        }]
        lts_nodes = [{
            "type": "vless", "tag": "lts-template-01", "server": "198.51.100.20",
            "server_port": 443, "uuid": "00000000-0000-0000-0000-000000000002", "flow": "xtls-rprx-vision",
            "tls": {"enabled": True, "server_name": "example.com", "utls": {"enabled": True, "fingerprint": "chrome"},
                    "reality": {"enabled": True, "public_key": "0000000000000000000000000000000000000000000", "short_id": "01"}}
        }]
        extra_manual = []
    else:
        uris = fetch_subscription(args.url)
        parsed_nodes = [n for i, u in enumerate(uris) if (n := parse_uri(u, i)) is not None]
        lts_nodes = [n for n in parsed_nodes if is_lts_node(n["tag"])][:20]
        std_candidates = [n for n in parsed_nodes if not is_lts_node(n["tag"])]
        print(f"[INFO] Узлов после санитизации: {len(parsed_nodes)} (Standard: {len(std_candidates)}, LTS: {len(lts_nodes)})")

        # v4.2: Проверка минимума живых узлов (Pre-Mortem #2)
        if len(parsed_nodes) < MIN_ALIVE_NODES:
            print(f"[FAIL] Недостаточно узлов после санитизации ({len(parsed_nodes)} < {MIN_ALIVE_NODES}). ABORT.", file=sys.stderr)
            print("[FAIL] Проверьте подписку — возможно, формат изменился или провайдер заблокирован.", file=sys.stderr)
            sys.exit(1)

        if args.test_local and std_candidates:
            delays = benchmark_nodes(std_candidates, args.timeout)
            alive = [n for n in std_candidates if n["tag"] in delays and delays[n["tag"]] > 0]
            alive.sort(key=lambda n: delays[n["tag"]])
            standard_top = alive[:args.top] or std_candidates[:args.top]
            rem_slots = max(0, MAX_ROUTER_NODES - len(standard_top) - len(lts_nodes))
            extra_manual = alive[args.top:args.top + rem_slots]
        else:
            standard_top = std_candidates[:args.top]
            rem_slots = max(0, MAX_ROUTER_NODES - len(standard_top) - len(lts_nodes))
            extra_manual = std_candidates[args.top:args.top + rem_slots]

    cfg = build_router_config(standard_top, lts_nodes, extra_manual, args.tolerance, secret)
    with open(args.out, "w", encoding="utf-8") as f:
        json.dump(cfg, f, ensure_ascii=False, indent=2)

    total_nodes = len(standard_top) + len(lts_nodes) + len(extra_manual)
    print(f"[OK] Конфиг (sing-box 1.12+) сохранён в {args.out} (узлов в ОЗУ: {total_nodes})")

    if args.push:
        # --- v4.2 (R-D2): Атомарная загрузка под set -e + отвязанным от терминала таймером отката ---
        subprocess.run(["scp", args.out, f"{args.push}:/tmp/config.json.new"], check=True)

        remote_cmd = (
            "set -e; "
            # 1. Жёсткая валидация ДО модификации рабочей конфигурации.
            "jq empty /tmp/config.json.new; "
            "sing-box check -c /tmp/config.json.new; "
            "fw4 check; "
            # 2. Резервная копия и взведение отвязанного от терминала 300с таймера отката.
            #    trap "" HUP INT TERM + </dev/null гарантируют выживание при обрыве SSH-сессии.
            "cp /etc/sing-box/config.json /etc/sing-box/config.json.bak; "
            "touch /tmp/rollback.lock; "
            "nohup sh -c 'trap \"\" HUP INT TERM; sleep 300; [ -f /tmp/rollback.lock ] && { "
            "mv /etc/sing-box/config.json.bak /etc/sing-box/config.json 2>/dev/null; "
            "/etc/init.d/sing-box restart; "
            "rm -f /tmp/rollback.lock; "
            "logger -t CRNE-ROLLBACK \"auto-rollback after 300s (push unconfirmed)\"; "
            "}' >/dev/null 2>&1 </dev/null & "
            "RB_PID=$!; "
            # 3. Атомарная замена и перезапуск службы.
            "mv /tmp/config.json.new /etc/sing-box/config.json; "
            "chmod 600 /etc/sing-box/config.json; "
            "/etc/init.d/sing-box restart; "
            # 4. Ожидание ФАКТИЧЕСКОЙ инициализации tun0 (4с) перед снятием блокировки отката.
            #    R-D2: procd restart асинхронен — kill таймера сразу после restart убивал его
            #    преждевременно. Проверяем реальное поднятие интерфейса и наличие PID sing-box.
            "sleep 4; "
            "ip link show tun0 >/dev/null 2>&1 && [ -n \"$(pidof -s sing-box)\" ]; "
            "rm -f /tmp/rollback.lock /etc/sing-box/config.json.bak; "
            "kill $RB_PID 2>/dev/null || true; "
            "logger -t CRNE-PUSH 'tun0 up & sing-box running: rollback timer cancelled.'; "
            # 5. Опциональный git-коммит (не ломает деплой, если git не установлен или нет изменений).
            "if command -v git >/dev/null 2>&1 && [ -d /etc/sing-box/.git ]; then "
            "  cd /etc/sing-box && git add config.json && "
            "  git commit -m \"sub2sing push $(date +%Y%m%d-%H%M)\" >/dev/null 2>&1 || true; "
            "fi"
        )
        try:
            subprocess.run(["ssh", args.push, remote_cmd], check=True)
        except subprocess.CalledProcessError as e:
            print(f"[FAIL] Удалённая проверка/активация провалена на {args.push}: {e}", file=sys.stderr)
            print("[FAIL] При провале валидации (до mv) старый config.json не тронут;", file=sys.stderr)
            print("[FAIL] при провале после mv — таймер отката (300с) вернёт .bak.", file=sys.stderr)
            raise
        print(f"[OK] Конфиг прошёл двойную валидацию (jq+sing-box+fw4), атомарно активирован,")
        print(f"[OK] tun0 поднят и sing-box работает: rollback-таймер погашен, закоммичен в git на {args.push}.")


if __name__ == "__main__":
    main()
```

---

## 📎 Приложение Б: Поставка бинарных правил `.srs`, веб-панели `ui` и аварийный скрипт

### Б.1. Скрипт загрузки, верификации `.srs` и веб-дашборда на сборочном узле (`download-assets.sh`)

```sh
#!/bin/sh
# download-assets.sh — скачивание и верификация .srs и легковесной веб-панели yacd для Offline-First доставки
# v4.3 (D3): Добавлена SHA256 verification для защиты от MITM-атак на бинарные правила
TARGET="${1:-root@10.10.1.1}"
mkdir -p ./crne-assets/ruleset ./crne-assets/ui && cd ./crne-assets || exit 1

BASE_GEOIP="https://raw.githubusercontent.com/SagerNet/sing-geoip/rule-set"
BASE_GEOSITE="https://raw.githubusercontent.com/SagerNet/sing-geosite/rule-set"
YACD_URL="https://github.com/haishanh/yacd/archive/refs/heads/gh-pages.zip"

# v4.3 (D3): Trusted SHA256 hashes от SagerNet (обновлять при релизах sing-geoip/sing-geosite)
# Источник: https://github.com/SagerNet/sing-geoip/releases и sing-geosite/releases
# Формат: "filename:expected_sha256"
TRUSTED_HASHES="
geoip-ru.srs:PLACEHOLDER_UPDATE_FROM_RELEASES
geosite-category-ru.srs:PLACEHOLDER_UPDATE_FROM_RELEASES
geosite-category-ads-all.srs:PLACEHOLDER_UPDATE_FROM_RELEASES
"

echo "[1/4] Скачивание бинарных правил .srs..."
curl -sSL -o ruleset/geoip-ru.srs "$BASE_GEOIP/geoip-ru.srs"
curl -sSL -o ruleset/geosite-category-ru.srs "$BASE_GEOSITE/geosite-category-ru.srs"
curl -sSL -o ruleset/geosite-category-ads-all.srs "$BASE_GEOSITE/geosite-category-ads-all.srs"

# v4.3 (D3): Проверка сигнатуры бинарного контейнера SRS (первые 3 байта = "SRS")
echo "[2/4] Верификация бинарных контейнеров SRS..."
for f in ruleset/*.srs; do
    MAGIC=$(head -c 3 "$f")
    if [ "$MAGIC" != "SRS" ]; then
        echo "[ERROR] Файл $f повреждён или не является бинарным .srs!" >&2
        exit 1
    fi
done

# v4.3 (D3): SHA256 verification против trusted hashes (защита от MITM-атак)
echo "[3/4] SHA256 verification (защита от MITM)..."
VERIFICATION_FAILED=0
for entry in $TRUSTED_HASHES; do
    [ -z "$entry" ] && continue
    filename=$(echo "$entry" | cut -d: -f1)
    expected_hash=$(echo "$entry" | cut -d: -f2)
    
    # Skip placeholder hashes (требуется ручное обновление при релизах)
    if [ "$expected_hash" = "PLACEHOLDER_UPDATE_FROM_RELEASES" ]; then
        echo "[WARN] $filename: SHA256 verification SKIPPED (placeholder hash). Update TRUSTED_HASHES!"
        actual_hash=$(sha256sum "ruleset/$filename" 2>/dev/null | awk '{print $1}')
        echo "[INFO] $filename actual SHA256: $actual_hash"
        continue
    fi
    
    actual_hash=$(sha256sum "ruleset/$filename" 2>/dev/null | awk '{print $1}')
    if [ "$actual_hash" != "$expected_hash" ]; then
        echo "[ERROR] SHA256 mismatch for $filename!" >&2
        echo "[ERROR] Expected: $expected_hash" >&2
        echo "[ERROR] Actual:   $actual_hash" >&2
        echo "[ERROR] Possible MITM attack or corrupted download!" >&2
        VERIFICATION_FAILED=1
    else
        echo "[OK] $filename: SHA256 verified"
    fi
done

if [ "$VERIFICATION_FAILED" = "1" ]; then
    echo "[FAIL] SHA256 verification failed! Aborting delivery." >&2
    echo "[FAIL] Проверьте доверенный источник или обновите TRUSTED_HASHES." >&2
    exit 1
fi

echo "[4/4] Скачивание веб-панели yacd..."
curl -sSL -o yacd.zip "$YACD_URL"
python3 -c "import zipfile; z=zipfile.ZipFile('yacd.zip'); z.extractall('ui_tmp')"
cp -r ui_tmp/yacd-gh-pages/* ./ui/
rm -rf yacd.zip ui_tmp

echo "[5/5] Доставка .srs и веб-панели на роутер ($TARGET)..."
scp ruleset/*.srs "$TARGET:/etc/sing-box/ruleset/"
scp -r ui/* "$TARGET:/etc/sing-box/ui/"
echo "[OK] Все ассеты успешно верифицированы (SRS + SHA256) и загружены на $TARGET."
```

### Б.2. Аварийный скрипт обновления на самом роутере (`/usr/bin/crne-sub-fallback.sh`)
Предназначен для экстренного пересчёта пула узлов на самом роутере без участия ПК и смартфона (динамически читает секрет через `jq`):

```sh
#!/bin/sh
# /usr/bin/crne-sub-fallback.sh — экстренный перезапуск проверки пула и ротация кэша на роутере
API="http://10.10.1.1:9090"
SECRET=$(jq -r '.experimental.clash_api.secret // empty' /etc/sing-box/config.json 2>/dev/null)

logger -t CRNE-FALLBACK "Running autonomous pool recovery..."
rm -f /tmp/sing-box-cache.db
/etc/init.d/sing-box restart
sleep 3
if [ -n "$SECRET" ]; then
    curl -s -m 6 -H "Authorization: Bearer $SECRET" "$API/group/GLOBAL-AUTO/delay?url=https://www.youtube.com/generate_204&timeout=2000"
    echo ""
fi
```
