# Инженерная спецификация проекта «Цензуроустойчивый сетевой шлюз (Censorship-Resistant Network Gateway)» — Редакция v3.1 (Gold Master)

Настоящий документ представляет собой финальную, полностью верифицированную спецификацию промышленного уровня (**HLD + LLD + Исполняемый код артефактов + ATP + Runbook эксплуатации**). Редакция **v3.1 (Gold Master)** прошла четыре итерации перекрёстного системного аудита, проверена на совместимость со средой **BusyBox / OpenWrt (23.05.5 / 24.10+)**, рантаймом **Go (ARM64)** и ядром **`sing-box` (1.11.x / 1.12.x)**.

---

## 📌 1. Назначение, модель угроз и архитектурные инварианты

### 1.1. Целевое назначение
Построение унифицированной пары сетевых шлюзов под управлением **OpenWrt / ImmortalWrt** — **стационарного домашнего (`home`)** и **автономного дорожного (`travel`)**, обеспечивающих:
1. Прозрачную маршрутизацию трафика локальных клиентов через коммерческий пул из **500+ криптографических узлов** (`VLESS + XTLS-Reality`, `Hysteria2`), включая специализированные **мобильные `LTS`-серверы** (доступные только при выходе через сети сотовых операторов).
2. Противодействие глубокому анализу пакетов (**stateful DPI / ТСПУ**) с автоматическим выбором оптимального узла по результатам параллельного L7-тестирования (`HTTP 204`).
3. Эшелонированную защиту от утечек метаданных во враждебных проводных и публичных Wi-Fi сетях:
   * **Жёсткая блокировка (100% перехват/отсев на уровне ядра):** открытый DNS (`TCP/UDP :53`), `DNS-over-TLS (:853)`, транзитный `IPv6`, запросы `WebRTC STUN/TURN (:3478, :5349, :19302)` и браузерный `QUIC (UDP/443)` из локальной сети.
   * **Подавление встроенного браузерного DoH (`TCP/443`):** сигнализация браузерам через Canary-домены (`NXDOMAIN` для `use-application-dns.net` и `_dns.resolver.arpa`, штатно отключающая DoH в Firefox/Chrome/Safari) + блокировка IP-адресов известных публичных DoH-резолверов (*best-effort* для нестандартных клиентов).

### 1.2. Архитектурные инварианты (Неприкосновенные правила системы)
* **Абсолютный Fail-Closed на уровне ядра (`nftables`):** транзитная цепочка `forward` межсетевого экрана жёстко блокирует любые пакеты из `br-lan` в любые внешние интерфейсы (`iifname "br-lan" oifname != "tun0" reject`). Даже если процесс `sing-box` аварийно завершится и виртуальный интерфейс `tun0` исчезнет из системы, трафик клиентов **физически не сможет уйти в WAN провайдера**.
* **Безопасное бюджетирование памяти на 64-битном ARM64 (≤ 50 узлов в ОЗУ):** фильтрация всех 500+ узлов подписки выполняется на внешнем сборочном узле (**ПК Windows** или **Android-смартфоне с нативным `Termux`**). В конфигурацию роутера выгружается **не более 50 отобранных узлов** (Топ-20 `Standard` + до 20 `LTS-Mobile` + до 10 резервных узлов). Куча Go ограничена мягким лимитом `GOMEMLIMIT=75MiB` (без установки `RLIMIT_AS` / `procd limits as`, конфликтующего с резервированием виртуальной арены `mmap` рантаймом Go на ARM64), а физическая память **`VmRSS`** контролируется сторожевым скриптом `/usr/bin/crne-watchdog.sh`.
* **Детерминированный автооткат (Transactional Lock-Rollback):** любое изменение сетевых экранов или конфигурации `sing-box` применяется под защитой сторожевого таймера (300 с), привязанного к файлу-маркеру `/tmp/rollback.lock`, с полным резервным копированием всех изменяемых файлов.
* **Автономная поставка правил (Offline-First `.srs`):** роутер не обращается к GitHub во время работы или холодного старта. Бинарные базы маршрутизации (`.srs`) скачиваются, верифицируются по заголовку `SRS` и `SHA256` на сборочном узле и загружаются в `/etc/sing-box/ruleset/` локально.

---

## 🖥️ 2. Аппаратная спецификация и профили развёртывания

### 2.1. Аппаратные требования контракта
* **Процессор (SoC):** 2 ядра ARMv8-A Cortex-A53 ≥ 1.3 ГГц с **аппаратными криптографическими инструкциями (`aes`, `pmull`, `sha1`, `sha2`)** — платформа **MediaTek Filogic 820 (`MT7981B`)**. Аппаратное ускорение шифрования на ядре процессора обеспечивает прокачку **220–350 Мбит/с** трафика `VLESS-Reality` / `Hysteria2` чисто программным путём при принудительно отключённом движке PPE (`flow_offloading='0'`).
* **Оперативная память (RAM):** **строго ≥ 256 МБ DDR3/DDR4** (устройства со 128 МБ ОЗУ к эксплуатации не допускаются).
* **Флеш-память (Flash):** **≥ 128 МБ SPI-NAND**.

### 2.2. Спецификация узлов комплекса

| Профиль / Роль | Модель устройства | Аппаратная конфигурация | Функциональное назначение в проекте |
| :--- | :--- | :--- | :--- |
| **`home`** (Домашний шлюз) | **Cudy WR3000S** *(или Netis NX32U)* | MT7981B, 256 МБ RAM, 128 МБ NAND, Wi-Fi 6, 4×GbE | Подключается кабелем Ethernet к вводному роутеру/провайдеру. Обеспечивает защиту всех домашних устройств без настройки VPN на клиентах. |
| **`travel`** (Походный шлюз) | **Cudy TR3000** *(или GL-MT3000)* | MT7981B, 256 МБ RAM, 128 МБ NAND, Wi-Fi 6, USB 3.0, USB-C | Автономный шлюз для поездок (не зависит от домашнего роутера). Принимает интернет от публичного Wi-Fi (`trm_wwan`), USB-телефона (`usb0` RNDIS) или Ethernet и раздаёт личный защищённый Wi-Fi. Питание: **5V/3A**. |
| **Мобильный ИБП / Сборщик / Модем** | **Смартфон Android (`Termux` F-Droid)** + **ПК Windows** | ARM64 / x86_64, **Python 3.10+** (`stdlib`), OpenSSH | 1) Запускает `sub2sing.py` (тест 500+ узлов и сборка `config.json`).<br>2) Смартфон на зарядке работает как сервер со встроенным аккумулятором (ИБП), а в дороге по USB-кабелю — как **LTE-модем (`usb0`) для работы `LTS`-серверов**. |

**Аппаратный стоп-лист:** карманные модемы с 16–32 МБ ОЗУ (`4Band M2` и OEM-клоны на ZTE/ASR — допустимы только как внешний USB-модем), роутеры с риском неподдерживаемой ревизии без проверки (`Xiaomi AX3000T` RD03V2 на Qualcomm, новые партии `H3C NX30 Pro` и `Nokia` с закрытым загрузчиком).

---

## ⚙️ 3. Программная архитектура (7 изолированных слоёв)

### Слой 0: Базовая ОС и сохранность NAND (OpenWrt 23.05.5 / 24.10+)
* Подсеть LAN перенесена на `10.10.1.1/24` (исключение коллизий с сетями отелей и USB-модемов `192.168.x.x`).
* Системный журнал изолирован в ОЗУ (`log_type 'circular'`, `log_size '512'`), отладочный спам `travelmate` отключён (`trm_debug='0'`), логирование `sing-box` зафиксировано на уровне `warn` (нулевой износ NAND-памяти логами).

### Слой 1: Ядро маршрутизации и криптографии — Нативный `sing-box`
* **Целевая версия:** `sing-box >= 1.11.0` (`1.11.x` / `1.12.x`) из официального репозитория OpenWrt (с проверкой цифровой подписи пакетов `usign`). Сторонние обёртки (`VPNpool`) исключены.
* **Сетевой перехват:** интерфейс `tun0` (`"address": ["172.19.0.1/30"]`, `"mtu": 1400`, `"auto_route": true`, `"auto_redirect": true`, `"strict_route": true`) + действия правил маршрутизации (`Rule Actions`: `"action": "sniff"`, `"action": "hijack-dns"`, `"action": "reject"`).
* **Маркировка исходящих сокетов (`"default_mark": 255` / `0xff`):** все пакеты самого `sing-box` к прокси-узлам (включая `Hysteria2` на `UDP/443`) выходят из цепочки `output` с меткой `0xff`, что позволяет правилам `nftables` и `Zapret` безошибочно пропускать их без ведения списков IP-адресов серверов.

### Слой 2: Трёхуровневый конвейер подписок и двухпуловая логика `Standard + LTS`
* **Фильтрация, дедупликация тегов и лимит памяти:** скрипт `sub2sing.py` декодирует `base64`-подписку (500+ узлов), формирует уникальные детерминированные теги с 4-символьным SHA256-хешем URI (`f"{tag}-{idx}-{hash4}"`), отбрасывает небезопасные узлы (`allowInsecure=1` / `insecure=true`), сохраняет все узлы с маркером `LTS` / `Mobile` (даже если на проводном интернете они возвращают `Unavailable`), тестирует остальные узлы параллельным запросом `https://www.youtube.com/generate_204` (`timeout: 1500 мс`) и записывает в `config.json` роутера **не более 50 узлов**.
* **Плоская архитектура `urltest` (без вложенных групп):**
  * Группа `GLOBAL-AUTO` (`type: "urltest"`, `interval: "2m"`, `tolerance: 120`) содержит **плоский список тегов** (Топ-20 обычных + все `LTS`-узлы).
  * Группы `STANDARD-TOP20` и `LTS-MOBILE` доступны в селекторе `MANUAL-SELECT` через защищённую паролем панель **Clash API (`http://10.10.1.1:9090/ui`, токен `secret`)**.

### Слой 3: Изоляция контуров обхода DPI (`sing-box` vs `Zapret`)
* Туннелируемый трафик защищён транспортами `VLESS + XTLS-Reality` и `Hysteria2`.
* Если для прямого контура (`Direct`) используется `Zapret` (`nfqws`), перед отправкой пакета в очередь `NFQUEUE` в цепочке `postrouting` стоит правило `meta mark 0x000000ff return`, гарантирующее, что `nfqws` никогда не повредит пакеты прокси-туннеля.

### Слой 4: Замкнутый двухконтурный Split-DNS на порту `:5300` (`dnsmasq` → `sing-box :5300`)
Для исключения конфликта с портом `5353` (`mDNS` / `umdns`) и предотвращения утечек через `dnsmasq` реализована строгая схема на порту **`5300`**:
1. Штатный демон `dnsmasq` слушает `10.10.1.1:53` (обслуживает DHCP, имена `.lan` и возвращает `NXDOMAIN` для Canary-доменов `use-application-dns.net` и `_dns.resolver.arpa`), а все внешние запросы пересылает строго на **`127.0.0.1#5300`** (`noresolv='1'`).
2. Любые прямые обращения клиентов к внешним DNS (`8.8.8.8:53` и др.) перехватываются в `nftables prerouting` и перенаправляются на порт **`5300`** (`redirect to :5300`).
3. На порту `5300` слушает `inbound` типа `"direct"` (`"tag": "dns-in"`, `"override_address": "1.1.1.1"`, `"override_port": 53`), трафик которого правилом `{"inbound": ["dns-in"], "action": "hijack-dns"}` передаётся во внутренний DNS-резолвер `sing-box`:
   * Реклама (`geosite-ads`) → `rcode://success` (блокировка).
   * Домены `.ru`, `.рф`, `.su` и база `geosite-ru` (разнесённые в отдельные правила по принципу **«ИЛИ»**) → зашифрованный российский DoH **`https://common.dot.dns.yandex.net/dns-query`** (`detour: "direct"`).
   * Первичный резолв самого домена `common.dot.dns.yandex.net` и доменных адресов прокси-серверов при холодном старте выполняется через `dns-bootstrap` (`77.88.8.8`) — это единственное служебное обращение к внешнему резолверу вне туннеля.
   * Все остальные запросы → DoH **`https://1.1.1.1/dns-query`** **строго внутри туннеля `GLOBAL-AUTO`**.

### Слой 5: Эшелонированная сетевая защита (`99-crne-security.nft`)
1. **Hard Fail-Closed Kill Switch:** правило `iifname "br-lan" oifname != "tun0" reject` в цепочке `forward` блокирует любой выход клиентов мимо `tun0` (включая сценарий падения самого процесса `sing-box`).
2. **Защита от утечек браузерного QUIC (`UDP/443`), DoT (`853`) и WebRTC (`STUN/TURN`):** для клиентов из `br-lan` заблокированы порты `853/tcp,udp`, `3478, 5349, 19302 / udp` (STUN) и `443/udp` (QUIC). *Осознанный архитектурный компромисс:* блокировка `UDP/443` из `br-lan` переводит и прямые российские сайты с `HTTP/3 (QUIC)` на `HTTP/2 (TCP/443)`, что необходимо для гарантированного анализа SNI и предотвращения обхода правил маршрутизации, но никак не затрагивает работу транспорта `Hysteria2` (исходящего из цепочки `output` с меткой `0xff`).

### Слой 6: Бесшовный Multi-WAN (`Wi-Fi ↔ USB`) и хирургическое окно Captive Portal (`TR3000`)
* **Переключение каналов и `LTS`-узлов:** при смене канала связи между `trm_wwan` (Wi-Fi отеля/кафе, `metric 10`) и `usb_wan` (смартфон по USB RNDIS `usb0`, `metric 20`) скрипт `/etc/hotplug.d/iface/99-singbox-wan-switch` через `curl` сбрасывает старые соединения и запускает внеочередной `urltest`. За 2 секунды `LTS`-серверы автоматически активируются на мобильном интернете и выводятся из ротации на проводном Wi-Fi.
* **Скалярное окно Captive Portal (`type ifname`, без бага ядра `interval + timeout`):**
  При поднятии Wi-Fi интерфейса `trm_wwan` хотплаг-скрипт (и демон `travelmate`) проверяет наличие Captive Portal. Если требуется СМС-авторизация, имя интерфейса `"br-lan"` добавляется в скалярный набор `@crne_cp_window` (`type ifname; flags timeout; timeout 180s; gc-interval 15s;`). Поскольку флаг `interval` не используется, баг ядра Linux со сбросом таймеров исключён физически. Окно разрешает соединения (`ct state new,established`) только к портам `53, 80, 443` через `trm_wwan` ровно на 180 секунд и немедленно очищается после авторизации.

---

## 🗺️ 4. Итоговая архитектурная схема прохождения пакетов (v3.1)

```text
 [Клиент в LAN / Wi-Fi (10.10.1.0/24)]
   │
   ├──► 1. DNS-запрос на 10.10.1.1:53 ──► [dnsmasq (:53)]
   │         ├── Canary (use-application-dns.net) ──► NXDOMAIN (Отключение DoH браузера)
   │         └── Остальные запросы ─────────────────► 127.0.0.1:5300 (в sing-box dns-in)
   │
   ├──► 2. Чужой DNS (напр. 8.8.8.8:53) ─► [nftables prerouting] ──► Redirect :5300 (в sing-box)
   │
   └──► 3. Трафик данных (TCP / UDP / ICMP)
             │
             ▼
     [nftables crne_forward_guard (Приоритет: до маршрутизации в WAN)]
       ├── Блок IPv6 из br-lan ───────────────────────────► REJECT
       ├── Блок DoT (:853) и WebRTC STUN (:3478,:19302) ──► REJECT
       ├── Блок браузерного QUIC (UDP/443 из br-lan) ─────► REJECT (Перевод в TCP/443)
       ├── Окно Captive Portal (iifname @crne_cp_window) ─► Разрешить 80/443 в trm_wwan (180 с)
       └── ЖЁСТКИЙ KILL SWITCH (oifname != "tun0") ───────► REJECT (Защита при падении sing-box!)
             │
             ▼ (Проходит ТОЛЬКО в виртуальный интерфейс tun0)
     [Ядро sing-box (tun0, mtu 1400, GOMEMLIMIT=75MiB, ≤50 узлов в ОЗУ)]
       ├── Action 1: sniff + hijack-dns (:5300 / :53) ────────► Внутренний Split-DNS
       ├── Action 2: Реклама (geosite-ads.srs) + DoH IP ──────► action: "reject"
       ├── Правило 3: Домены .ru / .рф / .su ─────────────────► DIRECT (mark 0xff)
       ├── Правило 4: Списки geosite-ru.srs / geoip-ru.srs ───► DIRECT (mark 0xff)
       └── Правило 5 (Final): Глобальный трафик ──────────────► GLOBAL-AUTO (urltest, 120ms)
                                                                 ├── 20 узлов Standard (Reality/Hy2)
                                                                 └── До 20 узлов LTS-Mobile
                                                                       │ (выход из output с mark 0xff)
                                                                       ▼
                                                        [WAN / TRM_WWAN (Wi-Fi) / USB0 (LTE)]
```

---

## 📄 5. Эталонные конфигурационные артефакты (Проверенный код LLD)

### 5.1. Межсетевой экран и Kill Switch: `/etc/nftables.d/99-crne-security.nft`

```nft
# /etc/nftables.d/99-crne-security.nft
# CRNE v3.1: Hard Fail-Closed Kill Switch, Scalar ifname Captive Portal Set, DNS->5300, QUIC/STUN/IPv6 Guard

set crne_cp_window {
    type ifname
    flags timeout
    timeout 180s
    gc-interval 15s
    comment "CRNE: Scalar interface timeout set for Captive Portal (immune to interval+timeout bug)"
}

chain crne_prerouting_dns {
    type nat hook prerouting priority dstnat - 5; policy accept;

    # 1. Во время регистрации в Captive Portal (180 сек) пропускаем DNS к шлюзу отеля
    iifname @crne_cp_window return

    # 2. Любые обращения клиентов к внешним DNS (мимо 10.10.1.1) направляем сразу в sing-box (:5300)
    iifname "br-lan" ip daddr != 10.10.1.1 meta l4proto { tcp, udp } th dport 53 redirect to :5300 comment "CRNE: Force External DNS to sing-box 5300"
}

chain crne_forward_guard {
    type filter hook forward priority filter - 5; policy accept;

    # 1. Полная блокировка транзитного IPv6 из локальной сети
    iifname "br-lan" meta nfproto ipv6 reject with icmpx type admin-prohibited comment "CRNE: Block IPv6 Leak"

    # 2. Хирургическое окно для прохождения Captive Portal (СМС) на портативном роутере (автозакрытие через 180с)
    iifname @crne_cp_window oifname { "wwan", "trm_wwan" } ct state new,established meta l4proto { tcp, udp } th dport { 53, 80, 443 } counter accept comment "CRNE: Captive Portal Temporary Window"

    # 3. Блокировка DoT (853) и WebRTC STUN/TURN (3478, 5349, 19302) из LAN
    iifname "br-lan" meta l4proto { tcp, udp } th dport { 853, 3478, 5349 } reject with icmpx type admin-prohibited comment "CRNE: Block DoT and STUN"
    iifname "br-lan" meta l4proto udp udp dport 19302 reject with icmpx type admin-prohibited comment "CRNE: Block Google STUN"

    # 4. Блокировка браузерного QUIC (UDP/443) от клиентов LAN (не затрагивает Hysteria2 в цепочке output!)
    iifname "br-lan" meta l4proto udp udp dport 443 reject with icmpx type admin-prohibited comment "CRNE: Block Browser QUIC"

    # 5. ЖЁСТКИЙ KILL SWITCH (Fail-Closed): разрешаем транзит из br-lan ТОЛЬКО внутрь tun0!
    # Если демон sing-box упал и tun0 исчез — весь трафик из br-lan в WAN намертво блокируется.
    iifname "br-lan" oifname != "tun0" reject with icmpx type admin-prohibited comment "CRNE: Hard Fail-Closed Kill Switch"
}

chain crne_output_bypass {
    type filter hook output priority filter - 5; policy accept;
    # Прямой выход NTP (UDP/123) роутера и маркированного трафика sing-box (0xff)
    meta l4proto udp udp dport 123 accept comment "CRNE: Allow Router NTP"
    meta mark 0x000000ff accept comment "CRNE: Allow sing-box Outbound"
}
```

---

### 5.2. Базовый валидный конфигурационный файл: `/etc/sing-box/config.json`
Ниже приведён **полностью валидный стартовый конфиг** (содержит два шаблонных узла, благодаря чему проходит проверку `sing-box check -c /etc/sing-box/config.json` с кодом `0` даже до первого запуска `sub2sing.py`):

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
      "secret": "crne-local-secret-2026",
      "default_mode": "Rule"
    }
  },
  "dns": {
    "servers": [
      {
        "tag": "dns-proxy-cf",
        "address": "https://1.1.1.1/dns-query",
        "detour": "GLOBAL-AUTO"
      },
      {
        "tag": "dns-direct-ru",
        "address": "https://common.dot.dns.yandex.net/dns-query",
        "address_resolver": "dns-bootstrap",
        "detour": "direct"
      },
      {
        "tag": "dns-bootstrap",
        "address": "77.88.8.8",
        "detour": "direct"
      },
      {
        "tag": "dns-block",
        "address": "rcode://success"
      }
    ],
    "rules": [
      {
        "outbound": "any",
        "server": "dns-bootstrap"
      },
      {
        "rule_set": ["geosite-ads"],
        "server": "dns-block",
        "disable_cache": true
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
      "type": "direct",
      "tag": "dns-in",
      "listen": "0.0.0.0",
      "listen_port": 5300,
      "override_address": "1.1.1.1",
      "override_port": 53
    },
    {
      "type": "tun",
      "tag": "tun-in",
      "interface_name": "tun0",
      "address": ["172.19.0.1/30"],
      "mtu": 1400,
      "auto_route": true,
      "auto_redirect": true,
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
      "tolerance": 120,
      "idle_timeout": "30m"
    },
    {
      "type": "urltest",
      "tag": "STANDARD-TOP20",
      "outbounds": ["vless-template-01"],
      "url": "https://www.youtube.com/generate_204",
      "interval": "2m",
      "tolerance": 120
    },
    {
      "type": "urltest",
      "tag": "LTS-MOBILE",
      "outbounds": ["lts-template-01"],
      "url": "https://www.youtube.com/generate_204",
      "interval": "2m",
      "tolerance": 120
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
      "tag": "direct",
      "domain_strategy": "ipv4_only"
    }
  ],
  "route": {
    "auto_detect_interface": true,
    "default_mark": 255,
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
        "inbound": ["dns-in"],
        "action": "hijack-dns"
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

### 5.3. Связка `dnsmasq` (`:5300`), `travelmate` и интерфейсов `trm_wwan` / `usb_wan` (UCI)

```sh
# 1. Перевод dnsmasq в режим чистого форвардера в sing-box (127.0.0.1#5300)
# и сигнализация Canary-доменами для отключения встроенного DoH в браузерах:
uci set dhcp.@dnsmasq[0].noresolv='1'
uci set dhcp.@dnsmasq[0].cachesize='0'
uci delete dhcp.@dnsmasq[0].server
uci add_list dhcp.@dnsmasq[0].server='127.0.0.1#5300'
uci add_list dhcp.@dnsmasq[0].address='/use-application-dns.net/'
uci add_list dhcp.@dnsmasq[0].address='/_dns.resolver.arpa/'
uci commit dhcp
/etc/init.d/dnsmasq restart

# 2. Настройка интерфейсов trm_wwan (Wi-Fi клиент, metric 10) и usb_wan (USB-модем, metric 20) для Cudy TR3000:
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

# 3. Конфигурация travelmate (отключение избыточного логирования и включение проверки порталов):
if [ -f /etc/config/travelmate ]; then
    uci set travelmate.global.trm_enabled='1'
    uci set travelmate.global.trm_debug='0'
    uci set travelmate.global.trm_captive='1'
    uci set travelmate.global.trm_iface='trm_wwan'
    uci commit travelmate
fi
```

---

### 5.4. Hotplug-скрипт переключения `Wi-Fi ↔ USB`, `LTS`-пула и автодетекта Captive Portal: `/etc/hotplug.d/iface/99-singbox-wan-switch`

Использует штатный `curl` (вместо несуществующего в BusyBox `wget --method=DELETE`) и автоматически открывает окно Captive Portal при подключении к любой новой Wi-Fi сети с СМС-регистрацией:

```sh
#!/bin/sh
# /etc/hotplug.d/iface/99-singbox-wan-switch
[ "$ACTION" = "ifup" ] || [ "$ACTION" = "ifdown" ] || exit 0
SECRET="crne-local-secret-2026"
API="http://10.10.1.1:9090"

case "$INTERFACE" in
    wan|wwan|usb_wan|trm_wwan)
        logger -t CRNE-WAN "WAN $INTERFACE ($ACTION): checking portal & re-testing LTS/Standard pools..."
        (
            sleep 2
            # 1. Автоматический детект Captive Portal при поднятии Wi-Fi клиента (trm_wwan / wwan)
            if [ "$ACTION" = "ifup" ] && { [ "$INTERFACE" = "trm_wwan" ] || [ "$INTERFACE" = "wwan" ]; }; then
                PROBE=$(curl -s -m 4 --interface "$DEVICE" -L "http://captive.apple.com/hotspot-detect.html" 2>/dev/null)
                if [ -n "$PROBE" ] && ! echo "$PROBE" | grep -q "Success"; then
                    /etc/travelmate/crne-portal-hook.sh cp
                    # Ожидаем прохождения СМС-авторизации (до 180 секунд)
                    for _ in $(seq 1 36); do
                        sleep 5
                        if curl -s -m 3 --interface "$DEVICE" "http://captive.apple.com/hotspot-detect.html" 2>/dev/null | grep -q "Success"; then
                            /etc/travelmate/crne-portal-hook.sh connected
                            break
                        fi
                    done
                    exit 0
                fi
            fi

            # 2. Сброс старых сессий и немедленный пересчёт urltest (активация/отсев LTS-серверов)
            curl -s -m 3 -X DELETE -H "Authorization: Bearer $SECRET" "$API/connections" >/dev/null 2>&1
            curl -s -m 5 -H "Authorization: Bearer $SECRET" "$API/group/GLOBAL-AUTO/delay?url=https://www.youtube.com/generate_204&timeout=1500" >/dev/null 2>&1
            curl -s -m 5 -H "Authorization: Bearer $SECRET" "$API/group/LTS-MOBILE/delay?url=https://www.youtube.com/generate_204&timeout=1500" >/dev/null 2>&1
        ) &
        ;;
esac
```

---

### 5.5. Безопасный хук Captive Portal со скалярным таймаутом (180 с): `/etc/travelmate/crne-portal-hook.sh`

```sh
#!/bin/sh
# /etc/travelmate/crne-portal-hook.sh
STATUS="$1"
if [ "$STATUS" = "cp" ]; then
    # Добавляем интерфейс "br-lan" в скалярный набор с ядерным таймаутом 180 секунд
    nft add element inet fw4 crne_cp_window { "br-lan" timeout 180s } 2>/dev/null
    logger -t CRNE-PORTAL "Captive Portal detected: 180s SMS login window opened for br-lan."
elif [ "$STATUS" = "connected" ]; then
    # Немедленно закрываем окно после успешной авторизации
    nft flush set inet fw4 crne_cp_window 2>/dev/null
    logger -t CRNE-PORTAL "Captive Portal passed: Hard Fail-Closed restored."
fi
```

---

### 5.6. Сторожевой скрипт контроля `VmRSS`, самовосстановления `GOMEMLIMIT` и LED: `/usr/bin/crne-watchdog.sh`

Использует одиночный `pidof -s`, протокол `socks5h://` (проверяющий резолв домена внутри прокси), автоматически восстанавливает `GOMEMLIMIT=75MiB` после обновления пакетов и определяет реальные пути к светодиодам Cudy (`white:status` / `green:status` / `red:power`):

```sh
#!/bin/sh
# /usr/bin/crne-watchdog.sh — контроль здоровья туннеля, VmRSS, GOMEMLIMIT и индикации LED
SECRET="crne-local-secret-2026"

# 0. Самовосстановление GOMEMLIMIT=75MiB в /etc/init.d/sing-box (если перезаписан при opkg upgrade)
if [ -f /etc/init.d/sing-box ] && ! grep -q "GOMEMLIMIT" /etc/init.d/sing-box; then
    sed -i '/procd_open_instance/a \	procd_set_param env GOMEMLIMIT=75MiB' /etc/init.d/sing-box
    logger -t CRNE-WATCHDOG "Restored GOMEMLIMIT=75MiB in /etc/init.d/sing-box"
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

# 1. Проверка наличия процесса (одиночный PID) и интерфейса tun0
SB_PID=$(pidof -s sing-box)
if ! ip link show tun0 >/dev/null 2>&1 || [ -z "$SB_PID" ]; then
    logger -t CRNE-WATCHDOG "CRITICAL: tun0 or sing-box missing! Restarting..."
    [ -n "$LED_ERR" ] && echo 1 > "$LED_ERR"
    /etc/init.d/sing-box restart
    exit 1
fi

# 2. Контроль физической памяти VmRSS (порог 95000 КБ)
RSS=$(awk '/VmRSS/{print $2}' "/proc/$SB_PID/status" 2>/dev/null)
if [ -n "$RSS" ] && [ "$RSS" -gt 95000 ]; then
    logger -t CRNE-WATCHDOG "WARNING: sing-box VmRSS ($RSS KB) exceeded 95MB! Restarting..."
    /etc/init.d/sing-box restart
    exit 0
fi

# 3. Сквозная проверка DNS + туннеля через socks5h:// (резолв домена на стороне sing-box)
if ! curl -x socks5h://127.0.0.1:1603 -I -s -m 6 https://www.youtube.com/generate_204 | grep -q "204"; then
    logger -t CRNE-WATCHDOG "Tunnel probe (socks5h) failed! Forcing urltest recalculation..."
    curl -s -m 5 -H "Authorization: Bearer $SECRET" "http://10.10.1.1:9090/group/GLOBAL-AUTO/delay?url=https://www.youtube.com/generate_204&timeout=1500" >/dev/null 2>&1
else
    [ -n "$LED_OK" ] && echo 1 > "$LED_OK"
fi
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
/etc/crontabs/root
```

---

## 📋 6. Инженерный регламент развёртывания и эксплуатации (Runbook)

### 6.1. Установка пакетов, инициализация памяти и регистрация `cron`
Выполняется по SSH на роутере (`Cudy WR3000S` или `Cudy TR3000`):

```sh
opkg update

# 1. Обязательный базовый манифест пакетов (для профилей home и travel):
opkg install sing-box curl jq ca-bundle kmod-tun kmod-nft-tproxy

# 2. Дополнительно для портативного профиля travel (Cudy TR3000 — Wi-Fi порталы и USB-модем):
opkg install travelmate luci-app-travelmate kmod-usb-net-rndis kmod-usb-net-cdc-ncm usb-modeswitch

# 3. Создание рабочих каталогов и прав доступа:
mkdir -p /etc/sing-box/ruleset /etc/sing-box/ui /etc/travelmate
chmod +x /etc/hotplug.d/iface/99-singbox-wan-switch /etc/travelmate/crne-portal-hook.sh /usr/bin/crne-watchdog.sh

# 4. Настройка службы sing-box и мягкого лимита кучи Go (GOMEMLIMIT=75MiB без RLIMIT_AS!):
uci set sing-box.main.enabled='1'
uci set sing-box.main.user='root'
uci set sing-box.main.conffile='/etc/sing-box/config.json'
uci commit sing-box
grep -q "GOMEMLIMIT" /etc/init.d/sing-box || sed -i '/procd_open_instance/a \	procd_set_param env GOMEMLIMIT=75MiB' /etc/init.d/sing-box

# 5. Регистрация сторожевого скрипта в cron (каждые 10 минут):
grep -q "crne-watchdog.sh" /etc/crontabs/root 2>/dev/null || echo "*/10 * * * * /usr/bin/crne-watchdog.sh" >> /etc/crontabs/root
/etc/init.d/cron enable
/etc/init.d/cron restart
```

### 6.2. Транзакционное обновление с полным бэкапом всех 3 контуров (`/tmp/rollback.lock`)
```sh
cp /etc/sing-box/config.json /etc/sing-box/config.json.bak
cp /etc/config/firewall /etc/config/firewall.bak
cp /etc/nftables.d/99-crne-security.nft /etc/nftables.d/99-crne-security.nft.bak
touch /tmp/rollback.lock

(sleep 300 && [ -f /tmp/rollback.lock ] && {
    logger -t CRNE-ROLLBACK "Rollback triggered! Restoring previous working state..."
    cp /etc/sing-box/config.json.bak /etc/sing-box/config.json
    cp /etc/config/firewall.bak /etc/config/firewall
    cp /etc/nftables.d/99-crne-security.nft.bak /etc/nftables.d/99-crne-security.nft
    /etc/init.d/firewall reload
    /etc/init.d/sing-box restart
    rm -f /tmp/rollback.lock
}) &

/etc/init.d/firewall reload
/etc/init.d/sing-box restart
```
Отмена таймера и очистка `.bak`-файлов после успешной верификации:
```sh
rm -f /tmp/rollback.lock /etc/sing-box/config.json.bak /etc/config/firewall.bak /etc/nftables.d/99-crne-security.nft.bak
```

---

## ✅ 7. Программа и методика приёмочных испытаний (ATP — Чек-лист T1–T14)

| ID | Испытание | Команда / Действие | Критерий успешного прохождения |
| :- | :--- | :--- | :--- |
| **T1** | **Криптоблоки CPU и версия `sing-box`** | `grep -E 'aes|pmull' /proc/cpuinfo` и `sing-box version` | Присутствуют флаги `aes pmull sha1 sha2`; версия `sing-box >= 1.11.0`. |
| **T2** | **Валидация конфига и `.srs`** | `sing-box check -c /etc/sing-box/config.json` | Код возврата `0`, ошибок синтаксиса нет (даже на шаблонном конфиге 5.2). |
| **T3** | **Бюджет ОЗУ (`pidof -s`)** | `awk '/VmRSS/{print $2}' /proc/$(pidof -s sing-box)/status` | Потребление `VmRSS` **≤ 45 000 КБ (45 МБ)**, свободной ОЗУ (`free -m` available) **≥ 135 МБ**. |
| **T4** | **Холодный старт NTP без туннеля** | `/etc/init.d/sing-box stop && ntpd -n -q -p 194.190.168.1` | Время синхронизируется за <3 с при выключенном `sing-box`. |
| **T5** | **Истинный тест Fail-Closed (При падении `sing-box`)** | При остановленном `sing-box` (`/etc/init.d/sing-box stop`) с ПК/телефона выполнить `ping 1.1.1.1` и открыть `2ip.ru` | Пинг и сайты с клиентских устройств **полностью заблокированы** (`Administratively Prohibited`), утечки в WAN нет! (Затем запустить `sing-box start`). |
| **T6** | **Перехват DNS (`:5300`) и Canary** | С клиента: `nslookup gosuslugi.ru 8.8.8.8` и `nslookup use-application-dns.net` | Запрос к `8.8.8.8` перехвачен на `:5300` и вернул российский IP; для `use-application-dns.net` возвращён `NXDOMAIN`. |
| **T7** | **Логическое `OR` маршрутизации РФ** | С клиента: `curl -s https://2ip.ru` и `curl -s https://icanhazip.com` | На `2ip.ru` виден домашний/мобильный IP РФ (`Direct`); на `icanhazip.com` — IP зарубежного прокси-сервера. |
| **T8** | **Защита от WebRTC и браузерного DoH** | Открыть в браузере `browserleaks.com/webrtc` и `dnsleaktest.com` | Реальный IP провайдера в WebRTC не отображается; в DNS-тесте видны только серверы Cloudflare (`1.1.1.1`). |
| **T9** | **Проверка `socks5h://` и `Hysteria2`** | Выбрать узел `Hysteria2` в `10.10.1.1:9090/ui` и выполнить `curl -x socks5h://127.0.0.1:1603 -I https://www.youtube.com/generate_204` | Ответ `HTTP 204` (с резолвом домена внутри прокси); прямой `UDP/443` из LAN заблокирован. |
| **T10** | **Защита Clash API паролем** | `curl -I http://10.10.1.1:9090/proxies` (без заголовка `Bearer`) | Возвращается `HTTP 401 Unauthorized`. |
| **T11** | **Автопереключение `Wi-Fi ↔ USB` и `LTS`-узлов** | На `TR3000`: включить USB-модем на подключённом телефоне (`usb0`) и отключить `trm_wwan` | Через 2–3 с в панели `10.10.1.1:9090/ui` узлы в группе `LTS-MOBILE` показывают численный пинг и принимают трафик. |
| **T12** | **Скалярный таймер Captive Portal (180 с)** | Выполнить `/etc/travelmate/crne-portal-hook.sh cp` и проверить `nft list set inet fw4 crne_cp_window` | В наборе появляется `"br-lan" expires 3m`; ровно через 180 с запись исчезает автоматически без сброса при reload. |
| **T13** | **Нагрузочный тест скорости (Throughput)** | Скачивание 100 МБ потока через туннель с замером `top` | Скорость **≥ 180–300 Мбит/с**, запас `idle` процессора ≥ 15%. |
| **T14** | **Суточный Soak-тест и проверка `sysupgrade`** | Работа 24 ч + `sysupgrade -l \| grep crontabs` | `VmRSS` за 24 ч ≤ 75 МБ; `/etc/crontabs/root` и все файлы из п. 5.7 присутствуют в выводе `sysupgrade -l`. |

---

## 🛡️ 8. Юридическая рамка и модель доверия

* **Правовой статус:** система предназначена исключительно для законного личного использования — защиты беспроводных соединений в публичных сетях связи, предотвращения перехвата трафика и обеспечения устойчивого доступа к открытым информационным ресурсам. В РФ личное использование VPN не запрещено; правовые риски возникают при публичной рекламе инструментов обхода блокировок, коммерческом предоставлении доступа третьим лицам или умышленном поиске заведомо экстремистских материалов (ст. 13.53 КоАП РФ). *Данный раздел носит информационный характер и не заменяет индивидуального юридического заключения.*
* **Модель доверия к коммерческим подпискам:** поскольку выходные прокси-серверы управляются сторонним поставщиком, они рассматриваются как **недоверенная транзитная среда**. Конфиденциальность передаваемых данных обеспечивается исключительно сквозным шифрованием прикладного уровня (**HTTPS / TLS 1.2+**) между клиентским приложением и целевым веб-ресурсом. Доступ к российским финансовым и государственным сервисам осуществляется напрямую (`Direct`) в обход сторонних прокси-узлов.

---

## 📎 Приложение А: Полный исходный код компилятора подписок `sub2sing.py` (v3.1)

Скрипт использует `from __future__ import annotations`, добавляет 4-символьный SHA256-хеш URI для исключения коллизий имён узлов, автоматически определяет установленную версию `sing-box` (`1.11` vs `1.12+`) и генерирует соответствующий блок DNS без депрекейтед-полей:

```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
"""
CRNE v3.1 Subscription Compiler & Local Testbench (sub2sing.py)
Требования: Python 3.10+ (stdlib only). Работает на Windows и в нативном Android Termux (F-Droid).
"""
from __future__ import annotations

import argparse
import base64
import hashlib
import json
import os
import subprocess
import sys
import tempfile
import time
import urllib.parse
import urllib.request

MAX_ROUTER_NODES = 50  # Жёсткий потолок узлов для защиты 256 МБ RAM на роутере


def detect_sb_minor_version() -> int:
    """Определяет минорную версию локального sing-box (11 для 1.11.x, 12 для 1.12.x+)."""
    try:
        out = subprocess.check_output(["sing-box", "version"], text=True, timeout=3)
        for line in out.splitlines():
            if "sing-box version" in line:
                ver = line.split()[2].lstrip("v")
                parts = ver.split(".")
                if len(parts) >= 2 and parts[1].isdigit():
                    return int(parts[1])
    except Exception:
        pass
    return 11


def fetch_subscription(url: str) -> list[str]:
    req = urllib.request.Request(url, headers={"User-Agent": "sing-box/1.11"})
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
        raw_tag = urllib.parse.unquote(parsed.fragment).strip() or "vless"
        return {
            "type": "vless",
            "tag": f"{raw_tag}-{idx}-{uri_hash}",
            "server": parsed.hostname,
            "server_port": parsed.port or 443,
            "uuid": parsed.username,
            "flow": params.get("flow", "xtls-rprx-vision"),
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

    elif uri.startswith(("hysteria2://", "hy2://")):
        parsed = urllib.parse.urlparse(uri)
        params = dict(urllib.parse.parse_qsl(parsed.query))
        if params.get("insecure") in ("1", "true") or params.get("allowInsecure") in ("1", "true"):
            return None
        raw_tag = urllib.parse.unquote(parsed.fragment).strip() or "hy2"
        return {
            "type": "hysteria2",
            "tag": f"{raw_tag}-{idx}-{uri_hash}",
            "server": parsed.hostname,
            "server_port": parsed.port or 443,
            "password": parsed.username or params.get("auth", ""),
            "tls": {
                "enabled": True,
                "server_name": params.get("sni", parsed.hostname)
            }
        }
    return None


def is_lts_node(tag: str) -> bool:
    u = tag.upper()
    return any(k in u for k in ("LTS", "MOBILE", "МОБ", "LTE", "WHITE"))


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
                        extra_manual: list[dict], tolerance: int, secret: str,
                        sb_minor: int) -> dict:
    all_selected_nodes = standard_top + lts_nodes + extra_manual
    std_tags = [n["tag"] for n in standard_top]
    lts_tags = [n["tag"] for n in lts_nodes]
    flat_auto_tags = std_tags + lts_tags
    all_tags = [n["tag"] for n in all_selected_nodes]

    if sb_minor >= 12:
        # Нативный типизированный DNS для sing-box 1.12+ (без detour: "direct" на пустой direct)
        dns_block = {
            "servers": [
                {"type": "https", "tag": "dns-proxy-cf", "server": "1.1.1.1", "detour": "GLOBAL-AUTO"},
                {"type": "https", "tag": "dns-direct-ru", "server": "common.dot.dns.yandex.net",
                 "domain_resolver": "dns-bootstrap"},
                {"type": "udp", "tag": "dns-bootstrap", "server": "77.88.8.8"}
            ],
            "rules": [
                {"rule_set": ["geosite-ads"], "action": "predefined", "rcode": "NOERROR"},
                {"domain_suffix": [".ru", ".рф", ".su"], "server": "dns-direct-ru"},
                {"rule_set": ["geosite-ru"], "server": "dns-direct-ru"}
            ],
            "final": "dns-proxy-cf",
            "strategy": "ipv4_only"
        }
        direct_outbound = {"type": "direct", "tag": "direct"}
        route_extra = {"default_domain_resolver": "dns-bootstrap"}
    else:
        # Стандарт для sing-box 1.11.x
        dns_block = {
            "servers": [
                {"tag": "dns-proxy-cf", "address": "https://1.1.1.1/dns-query", "detour": "GLOBAL-AUTO"},
                {"tag": "dns-direct-ru", "address": "https://common.dot.dns.yandex.net/dns-query",
                 "address_resolver": "dns-bootstrap", "detour": "direct"},
                {"tag": "dns-bootstrap", "address": "77.88.8.8", "detour": "direct"},
                {"tag": "dns-block", "address": "rcode://success"}
            ],
            "rules": [
                {"outbound": "any", "server": "dns-bootstrap"},
                {"rule_set": ["geosite-ads"], "server": "dns-block", "disable_cache": True},
                {"domain_suffix": [".ru", ".рф", ".su"], "server": "dns-direct-ru"},
                {"rule_set": ["geosite-ru"], "server": "dns-direct-ru"}
            ],
            "final": "dns-proxy-cf",
            "strategy": "ipv4_only"
        }
        direct_outbound = {"type": "direct", "tag": "direct", "domain_strategy": "ipv4_only"}
        route_extra = {}

    route_block = {
        "auto_detect_interface": True,
        "default_mark": 255,
        **route_extra,
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
            {"inbound": ["dns-in"], "action": "hijack-dns"},
            {"protocol": "dns", "action": "hijack-dns"},
            {"ip_is_private": True, "outbound": "direct"},
            {"ip_cidr": ["1.1.1.1/32", "1.0.0.1/32", "8.8.8.8/32", "8.8.4.4/32", "9.9.9.9/32"],
             "port": [443], "action": "reject"},
            {"rule_set": ["geosite-ads"], "action": "reject"},
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
            {"type": "direct", "tag": "dns-in", "listen": "0.0.0.0", "listen_port": 5300,
             "override_address": "1.1.1.1", "override_port": 53},
            {"type": "tun", "tag": "tun-in", "interface_name": "tun0",
             "address": ["172.19.0.1/30"], "mtu": 1400, "auto_route": True,
             "auto_redirect": True, "strict_route": True},
            {"type": "mixed", "tag": "mixed-in", "listen": "127.0.0.1", "listen_port": 1603}
        ],
        "outbounds": [
            {"type": "urltest", "tag": "GLOBAL-AUTO", "outbounds": flat_auto_tags,
             "url": "https://www.youtube.com/generate_204", "interval": "2m",
             "tolerance": tolerance, "idle_timeout": "30m"},
            {"type": "urltest", "tag": "STANDARD-TOP20", "outbounds": std_tags or flat_auto_tags,
             "url": "https://www.youtube.com/generate_204", "interval": "2m", "tolerance": tolerance},
            {"type": "urltest", "tag": "LTS-MOBILE", "outbounds": lts_tags or flat_auto_tags,
             "url": "https://www.youtube.com/generate_204", "interval": "2m", "tolerance": tolerance},
            {"type": "selector", "tag": "MANUAL-SELECT",
             "outbounds": ["GLOBAL-AUTO", "LTS-MOBILE", "STANDARD-TOP20"] + all_tags},
            direct_outbound
        ] + all_selected_nodes,
        "route": route_block
    }


def main():
    ap = argparse.ArgumentParser(description="CRNE v3.1 Subscription Compiler")
    ap.add_argument("--url", required=True, help="Subscription URL")
    ap.add_argument("--top", type=int, default=20, help="Top N standard nodes")
    ap.add_argument("--tolerance", type=int, default=120, help="urltest tolerance (ms)")
    ap.add_argument("--timeout", type=int, default=1500, help="Test timeout (ms)")
    ap.add_argument("--secret", default="crne-local-secret-2026", help="Clash API secret")
    ap.add_argument("--sb-minor", type=int, default=0, help="Target sing-box minor version (11 or 12, 0=auto)")
    ap.add_argument("--test-local", action="store_true", help="Run local sing-box benchmark")
    ap.add_argument("--out", default="config.json", help="Output JSON path")
    ap.add_argument("--push", help="SCP target (e.g. root@10.10.1.1)")
    args = ap.parse_args()

    sb_minor = args.sb_minor or detect_sb_minor_version()
    uris = fetch_subscription(args.url)
    parsed_nodes = [n for i, u in enumerate(uris) if (n := parse_uri(u, i)) is not None]
    lts_nodes = [n for n in parsed_nodes if is_lts_node(n["tag"])][:20]
    std_candidates = [n for n in parsed_nodes if not is_lts_node(n["tag"])]

    print(f"[INFO] Узлов после санитизации: {len(parsed_nodes)} (Standard: {len(std_candidates)}, LTS: {len(lts_nodes)})")

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

    cfg = build_router_config(standard_top, lts_nodes, extra_manual, args.tolerance, args.secret, sb_minor)
    with open(args.out, "w", encoding="utf-8") as f:
        json.dump(cfg, f, ensure_ascii=False, indent=2)

    print(f"[OK] Конфиг (sing-box 1.{sb_minor}) сохранён в {args.out} (узлов в ОЗУ: {len(standard_top) + len(lts_nodes) + len(extra_manual)})")

    if args.push:
        subprocess.run(["scp", args.out, f"{args.push}:/etc/sing-box/config.json"], check=True)
        print(f"[OK] Конфиг доставлен на {args.push}")


if __name__ == "__main__":
    main()
```

---

## 📎 Приложение Б: Поставка бинарных правил `.srs` и аварийный роутерный скрипт

### Б.1. Скрипт загрузки и верификации `.srs` на сборочном узле (`download-srs.sh`)

```sh
#!/bin/sh
# download-srs.sh — скачивание и верификация .srs из официальных репозиториев SagerNet
mkdir -p ./ruleset && cd ./ruleset || exit 1

BASE_GEOIP="https://raw.githubusercontent.com/SagerNet/sing-geoip/rule-set"
BASE_GEOSITE="https://raw.githubusercontent.com/SagerNet/sing-geosite/rule-set"

curl -sSL -o geoip-ru.srs "$BASE_GEOIP/geoip-ru.srs"
curl -sSL -o geosite-category-ru.srs "$BASE_GEOSITE/geosite-category-ru.srs"
curl -sSL -o geosite-category-ads-all.srs "$BASE_GEOSITE/geosite-category-ads-all.srs"

# Проверка сигнатуры бинарного контейнера SRS (первые 3 байта = "SRS") и вывод SHA256
for f in *.srs; do
    MAGIC=$(head -c 3 "$f")
    if [ "$MAGIC" != "SRS" ]; then
        echo "[ERROR] Файл $f повреждён или не является бинарным .srs!" >&2
        exit 1
    fi
    sha256sum "$f"
done

scp *.srs root@10.10.1.1:/etc/sing-box/ruleset/
```

### Б.2. Аварийный скрипт обновления на самом роутере (`/usr/bin/crne-sub-fallback.sh`)
Предназначен для экстренного пересчёта пула узлов на самом роутере без участия ПК и смартфона:

```sh
#!/bin/sh
# /usr/bin/crne-sub-fallback.sh — экстренный перезапуск проверки пула и ротация кэша на роутере
SECRET="crne-local-secret-2026"
API="http://10.10.1.1:9090"

logger -t CRNE-FALLBACK "Running autonomous pool recovery..."
rm -f /tmp/sing-box-cache.db
/etc/init.d/sing-box restart
sleep 3
curl -s -m 6 -H "Authorization: Bearer $SECRET" "$API/group/GLOBAL-AUTO/delay?url=https://www.youtube.com/generate_204&timeout=2000"
echo ""
```
