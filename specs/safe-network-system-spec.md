# Инженерная спецификация проекта «Цензуроустойчивый сетевой шлюз (Censorship-Resistant Network Gateway)» — Редакция v2.0 (Production Baseline)

Настоящий документ объединяет верхнеуровневую архитектуру (**HLD**), низкоуровневое проектирование (**LLD**), эталонные конфигурационные артефакты и регламент приёмочных испытаний (**ATP**). Спецификация разработана в соответствии с методологией **CRNE (Censorship-Resistant Network Engineer)**, учитывает аппаратные ограничения встраиваемых систем (256 МБ ОЗУ / 128 МБ NAND) и закрывает эксплуатационные риски работы в фильтруемых проводных, мобильных и публичных сетях связи.

---

## 📌 1. Цели, модель угроз и архитектурные инварианты

### 1.1. Назначение системы
Спроектировать и развернуть пару унифицированных управляемых шлюзов на базе **OpenWrt / ImmortalWrt** — **стационарный домашний (`home`)** и **автономный портативный (`travel`)**, которые:
* Обеспечивают прозрачную маршрутизацию трафика клиентских устройств через коммерческий пул из **500+ криптографических узлов** (`VLESS + XTLS-Reality`, `Hysteria2`) с поддержкой специализированных **мобильных `LTS`-серверов**.
* Противодействуют системам глубокого анализа пакетов (**stateful DPI / ТСПУ**) за счёт маскировки транспорта под TLS 1.3 / QUIC, фрагментации рукопожатий и изолированной модификации пакетов в прямом контуре (`Zapret`).
* Гарантируют отсутствие утечек метаданных (**DNS, IPv6, браузерный QUIC / WebRTC**) на всех типах внешних каналов (домашний провайдер, публичный Wi-Fi отеля/кафе с СМС-авторизацией, мобильный USB-модем).

### 1.2. Базовые инженерные инварианты
1. **Fail-Closed (Безопасный отказ):** при падении всех прокси-узлов целевой трафик блокируется межсетевым экраном (`DROP`/`REJECT`), а не утекает в открытую сеть провайдера.
2. **Транзакционность с детерминированным автооткатом (Lock-File Rollback):** любые изменения правил `firewall4` (`nftables`) или конфигурации `sing-box` применяются под защитой фонового сторожа с файлом-маркером `/tmp/rollback.lock` (таймаут 300 секунд).
3. **Тонкий клиент на роутере, тяжёлая сборка вовне:** парсинг подписок, фильтрация небезопасных узлов и первичный массовый прогон `urltest` по 500+ серверам вынесены на внешний сборочный узел (**Windows-ПК** или **Android-смартфон с нативным Termux**), с сохранением аварийного автономного скрипта на самом роутере.
4. **Иммунитет к блокировкам репозиториев (Offline-First):** роутер не скачивает бинарные правила `.srs`, веб-интерфейсы или установочные скрипты с GitHub во время старта. Все артефакты доставляются локально со сборочного узла.
5. **Сквозная криптографическая гигиена:** даже в прямом контуре (`Direct` для российских ресурсов) разрешение имён выполняется исключительно по зашифрованному протоколу **DoH по IP-адресу** (`https://77.88.8.8/dns-query`), что исключает подмену DNS в публичных Wi-Fi сетях.

---

## 🖥️ 2. Аппаратная спецификация и профили развёртывания

### 2.1. Жёсткие системные требования контракта
* **SoC:** 2 ядра ARMv8-A Cortex-A53 ≥ 1.3 ГГц с **аппаратными криптографическими инструкциями ARMv8 Crypto Extensions (`AES`, `SHA1/SHA256`, `PMULL`)** — платформа **MediaTek Filogic 820 (`MT7981B`)**. Именно аппаратный криптоблок ядра обеспечивает скорость шифрования **250–350 Мбит/с** в `VLESS-Reality` / `Hysteria2` при программно отключённом движке пакетной разгрузки PPE (`flow_offloading='0'`).
* **Оперативная память (RAM):** **строго ≥ 256 МБ DDR3/DDR4** (устройства со 128 МБ RAM исключены из спецификации).
* **ПЗУ (Flash):** **≥ 128 МБ SPI-NAND** (размещение ядра `sing-box`, локальных бинарных списков `.srs`, статического веб-интерфейса `metacubexd` и пакета `travelmate` без использования `tmpfs`).

### 2.2. Целевая матрица оборудования

| Профиль | Модель | Аппаратная база | Роль и инженерное обоснование |
| :--- | :--- | :--- | :--- |
| **`home`** (Стационарный основной) | **Cudy WR3000S** | MT7981B, 256 МБ RAM, 128 МБ NAND, Wi-Fi 6, 4×GbE | Домашний шлюз (подключается кабелем Ethernet в WAN-порт к провайдеру или вводному роутеру). Единая ревизия, штатный переход на OpenWrt. |
| **`travel`** (Портативный походный) | **Cudy TR3000** | MT7981B, 256 МБ RAM, 128 МБ NAND, Wi-Fi 6, USB 3.0, USB-C | Автономный дорожный шлюз. Работает за тысячи километров от дома: принимает интернет по Wi-Fi отеля/кафе (`wwan`), по USB-кабелю от телефона (`usb0` RNDIS) или Ethernet и раздаёт защищённую личную сеть. Требует питания USB-C **5V/3A** (при работе с USB-модемом). |
| **Сборочный / Модемный узел** | **ПК (Windows)** + **Смартфон Android (Termux)** | Python 3.8+ (`stdlib`) + OpenSSH | Смартфон с `Termux` выступает энергонезависимым микро-сервером со встроенным аккумулятором (ИБП), а в поездке в паре с `TR3000` по USB — одновременно **LTE-модемом (`usb0`, открывающим доступ к `LTS`-серверам)** и **станцией пересборки конфига**. |

*Опциональные совместимые модели:* **Netis NX32U** (стационарный с USB 3.0) и **GL.iNet GL-MT3000** (портативный с 512 МБ RAM / 256 МБ NAND).

### 2.3. Аппаратный стоп-лист (запрещено к применению)
* Карманные LTE-модемы со встроенным Wi-Fi (`4Band M2` и аналоги на чипах ZTE/ASR) в качестве самостоятельного роутера (16–32 МБ RAM, закрытая RTOS). Допускаются только как тупой USB-модем, воткнутый в `Cudy TR3000`.
* Устройства с риском неподдерживаемой ревизии или закрытым загрузчиком без предварительной проверки: `Xiaomi AX3000T` (ревизия `RD03V2` на Qualcomm IPQ5018), новые ревизии `H3C NX30 Pro` и `Nokia`.

---

## ⚙️ 3. Программная архитектура (7 изолированных слоёв)

Суммарное потребление ОЗУ всем стеком жёстко ограничено на уровне **110–125 МБ** (за счёт системного лимита `GOMEMLIMIT=90MiB` для рантайма Go), что оставляет **>130 МБ** свободной памяти под сетевые буферы ядра Linux.

### Слой 0: Базовая ОС и защита флеш-памяти (OpenWrt 23.05 / 24.10+)
* Подсеть LAN изменена на `10.10.1.1/24` (предотвращение конфликтов маршрутизации с сетями отелей, домашних роутеров и USB-модемов `192.168.x.x`).
* Wi-Fi настроен в режиме `WPA3-SAE` (или гибридном `WPA2-PSK/WPA3-SAE`), в профиле `travel` активирована L2-изоляция клиентов (`option isolate '1'`).
* **Защита SPI-NAND от износа:** системный журнал OpenWrt переведён в кольцевой буфер в ОЗУ (`option log_type 'circular'`, `option log_size '512'`), логирование `sing-box` ограничено уровнем `warn` без записи на флеш.

### Слой 1: Ядро маршрутизации, криптографии и DNS — Нативный `sing-box`
Используется официальный пакет `sing-box` из репозитория OpenWrt (ветка `1.10+ / 1.11+`) **без сторонних обёрток вроде `VPNpool`**.
* **Режим перехвата:** виртуальный сетевой интерфейс `tun` (`"interface_name": "tun0"`, `"mtu": 1400`, `"auto_redirect": true`), работающий через нативные таблицы `nftables` с аппаратной поддержкой ICMP и TCP/UDP.
* **Самомаркировка исходящего трафика (`"default_mark": 255` / `0xff`):** все пакеты, генерируемые самим `sing-box` к любому из 500+ внешних прокси-серверов (включая `Hysteria2` по `UDP/443`), получают метку ядра `0xff`. Это позволяет правилам `nftables` и `Zapret` автоматически пропускать трафик ко всем 500+ серверам без необходимости хранить их IP-адреса или домены в списках исключений файрвола.
* **Локальные бинарные правила (`.srs`):** классификация трафика (`geosite-category-ru.srs`, `geoip-ru.srs`, `geosite-category-ads-all.srs`) загружается из локального каталога `/etc/sing-box/ruleset/` (`"type": "local"`).

### Слой 2: Трёхуровневый конвейер подписок и поддержка мобильных `LTS`-серверов
Поскольку коммерческая подписка (`v2ray subscription`, base64-кодированный список URI `vless://` и `hysteria2://`) содержит 500+ динамических узлов, включая **мобильные `LTS`-серверы** (доступные только с IP сотовых операторов и возвращающие `Unavailable` при проверке с домашнего проводного интернета), применяется трёхступенчатая схема:

1. **Уровень 1 (Основной — ПК Windows) и Уровень 2 (Мобильный ИБП — Android `Termux`):**
   Скрипт `sub2sing.py` (чистый Python 3 `stdlib`):
   * Скачивает подписку, отсеивает устаревшие протоколы и **принудительно удаляет любые узлы с отключённой проверкой сертификата (`allowInsecure=1` / `insecure=true`)**.
   * Выделяет все серверы с тегом `LTS` / `Mobile` в неприкосновенный пул (они **никогда не удаляются**, даже если не отвечают на проводном интернете).
   * Запускает локальный проверочный стенд `sing-box` и параллельно тестирует все обычные узлы запросом `https://www.youtube.com/generate_204` (таймаут `1500 мс`), отбирая **Топ-20 самых быстрых**.
   * Формирует в `config.json` иерархию групп:
     * `GLOBAL-AUTO` (`type: "urltest"`, `interval: "2m"`, `tolerance: 120`): объединяет **Топ-20 обычных узлов + ВСЕ `LTS`-узлы**. Проверяет все узлы группы параллельно!
     * `LTS-MOBILE` (`type: "urltest"`, `tolerance: 120`): только мобильные `LTS`-узлы.
     * `STANDARD-TOP20` (`type: "urltest"`, `tolerance: 120`): только Топ-20 обычных узлов.
     * `MANUAL-ALL` (`type: "selector"`): все остальные живые серверы из 500+ (не создают фонового трафика проверок).
2. **Уровень 3 (Аварийный автономный — скрипт `/usr/bin/crne-sub-fallback.sh` на роутере):**
   Скрипт на базе BusyBox `ash + base64 + jq` (занимает <200 КБ), позволяющий обновить подписку напрямую с роутера при отсутствии ПК и смартфона.
3. **Визуальный мониторинг и управление (`Clash API` + `metacubexd`):**
   Встроенный веб-дашборд доступен локально по адресу `http://10.10.1.1:9090/ui` (потребление ОЗУ на роутере — **0 МБ**). Отображает пинг всех серверов, текущий выбранный узел и активные соединения.

### Слой 3: Обход DPI и строгое разделение контуров (`sing-box` vs `Zapret`)
* **Контур А (Туннелируемый трафик):** маскируется протоколами `VLESS + XTLS-Reality` (TCP/443) и `Hysteria2` (UDP/QUIC) + встроенной TLS-фрагментацией (`tls_fragment`).
* **Контур Б (Прямой трафик `Direct`):** если на домашнем роутере требуется обход DPI без туннелирования для отдельных ресурсов, в цепочку `postrouting` подключается `Zapret` (`nfqws`).
  * **Железное правило изоляции:** первым правилом перед вызовом `nfqws` стоит `meta mark 0xff accept`. Любой трафик самого `sing-box` к 500+ серверам подписки гарантированно обходит `Zapret`.
  * **Профили `nfqws` (только с фильтром `--filter-l7=tls,http`):**
    * Основной: `--filter-l7=tls --dpi-desync=fake,multidisorder --dpi-desync-split-pos=1,midsld --dpi-desync-fooling=badseq` (или `md5sig`).
    * Резервный: `--filter-l7=tls --dpi-desync=multisplit --dpi-desync-split-seqovl=568 --dpi-desync-split-pos=1`.

### Слой 4: Безопасный двухконтурный Split-DNS без Bootstrap-петли
Все DoH-серверы задаются **строго по IP-адресам**, что полностью исключает циклическую зависимость (Bootstrap Deadlock) при холодном старте:
* **Контур РФ (`Direct DNS`):** домены `.ru`, `.рф`, `.su` и `geosite-category-ru.srs` разрешаются через зашифрованный российский DoH **`https://77.88.8.8/dns-query`** (`detour: "direct"`). Это даёт корректную геолокацию CDN для банков/Госуслуг и защищает от подмены DNS в публичных хотспотах.
* **Глобальный контур (`Proxy DNS`):** все остальные запросы идут через **`https://1.1.1.1/dns-query`** (резерв: `https://9.9.9.9/dns-query`) **строго внутри зашифрованного туннеля** (`detour: "GLOBAL-AUTO"`).
* **Блокировка рекламы:** выполняется на уровне DNS и маршрутизации по локальному списку `geosite-category-ads-all.srs` (`action: "reject"`).

### Слой 5: Межсетевой экран (`nftables`), защита от утечек и Kill Switch
Реализуется декларативным файлом `/etc/nftables.d/99-crne-security.nft`:
1. **Pre-VPN NTP Bypass:** исходящие пакеты `UDP/123` к IP-стратум серверу ВНИИФТРИ (`194.190.168.1`) пропускаются напрямую в `WAN`, предотвращая `NTP Deadlock` на роутерах без батарейки часов (RTC).
2. **Принудительный перехват DNS (`DNAT :53`) и блок `DoT 853`:** любые попытки клиентских устройств обратиться к чужим DNS на порту `53/TCP,UDP` заворачиваются на локальный `sing-box`, а порт `853/TCP,UDP` из `br-lan` наружу блокируется.
3. **Подавление браузерного QUIC (`UDP/443`) без поломки `Hysteria2`:** правило `iifname "br-lan" udp dport 443 reject with icmpx type admin-prohibited` срабатывает строго на входящем интерфейсе локальной сети для транзитных пакетов и переводит браузеры на управляемый `TCP/443`. Поскольку сам демон `sing-box` отправляет трафик `Hysteria2` с самого роутера (цепочка `output`, `meta mark 0xff`), резервный транспорт `Hysteria2` на порту `UDP/443` **никогда не блокируется**.
4. **IPv6 Leak Guard:** полная блокировка транзитного IPv6-трафика из `br-lan`, предотвращающая утечки в обход IPv4-туннеля.
5. **Kill Switch:**
   * В профиле `home` (`Policy-Based`): при падении всех 500+ узлов интерфейс `tun0` возвращает `ICMP unreachable` для туннелируемых сайтов, тогда как сайты РФ (`Direct`) продолжают работать.
   * В профиле `travel` (`Strict Mode`): весь трафик из `br-lan` (кроме окна авторизации Captive Portal) обязан проходить через `sing-box`.

### Слой 6: Бесшовный Multi-WAN (`Wi-Fi ↔ Мобильный USB`) и автопрохождение Captive Portal (Профиль `travel`)
На портативном роутере **Cudy TR3000** реализована совместная работа публичного Wi-Fi (`wwan`) и мобильного USB-модема/телефона (`usb0` RNDIS) **без использования тяжёлого пакета `mwan3`** (исключён конфликт меток `fwmark`):
* **Приоритеты каналов (Kernel Metrics):**
  * Интерфейс `wwan` (Wi-Fi отеля/кафе под управлением `travelmate`) — `metric 10` (основной при наличии бесплатного Wi-Fi).
  * Интерфейс `usb0` (смартфон по USB в режиме RNDIS-модема) — `metric 20` (автоматический резерв и канал для работы в дороге). Приоритеты меняются местами одной командой.
* **Мгновенная активация/отсев `LTS`-серверов при смене канала (`Hotplug Trigger`):**
  При любом переключении между `usb0` (мобильная сеть, где работают `LTS`-серверы) и `wwan` (проводной Wi-Fi кафе, где `LTS` недоступны) срабатывает скрипт `/etc/hotplug.d/iface/99-singbox-wan-switch`. Он сбрасывает старые сессии и запускает внеочередной `urltest` группы `GLOBAL-AUTO`. За **1.5–2 секунды** `sing-box` автоматически вводит `LTS`-серверы в строй на мобильном интернете и выводит их при переходе на обычный Wi-Fi.
* **Прохождение СМС-регистрации в публичных хотспотах (`travelmate` + `CRNE Portal Helper`):**
  1. При обнаружении Captive Portal демон `travelmate` через хук-скрипт временно приостанавливает перехват `DNAT :53` для подсети хотспота, позволяя странице ввода СМС открыться на вашем смартфоне.
  2. Поскольку смартфон выходит в сеть через NAT роутера, хотспот регистрирует **постоянный MAC-адрес роутера `TR3000`**.
  3. Для предотвращения повторных запросов СМС на роутере отключена рандомизация MAC на `wwan`, поддерживается клонирование MAC-адреса телефона и работает фоновый keepalive.
  4. На роутере доступна локальная страница-помощник (`http://10.10.1.1:9090/portal.html`), хранящая основной и резервный номера телефонов для автозаполнения формы провайдера в 1 клик.

---

## 🗺️ 4. Итоговая архитектурная схема (v2.0)

```text
================================================================================
 УРОВЕНЬ СБОРКИ И ОБНОВЛЕНИЯ ПОДПИСКИ (ВНЕ РОУТЕРА)
================================================================================
 [ПК Windows] или [Смартфон Android (чистый Termux без proot)]
   └── sub2sing.py --url "https://..." --test-local --push 10.10.1.1
         ├── 1. Декод base64 (500+ узлов) и удаление небезопасных (insecure=1)
         ├── 2. Защита всех мобильных [LTS]-серверов от удаления
         ├── 3. Параллельный тест (generate_204, 1500ms) -> отбор Топ-20 обычных
         ├── 4. Валидация (sing-box check) + упаковка локальных .srs правил
         └── 5. Заливка по SCP на роутер под защитой /tmp/rollback.lock
================================================================================
                           │ (LAN / USB / Wi-Fi)
                           ▼
================================================================================
 УРОВЕНЬ СЕТЕВОГО ШЛЮЗА: Cudy WR3000S (home) / Cudy TR3000 (travel)
================================================================================
 [Клиенты: Смартфоны, Ноутбуки, ТВ] ──► Wi-Fi 6 (WPA3-SAE) / LAN (10.10.1.0/24)
                                                 │
   ┌─────────────────────────────────────────────┴─────────────────────────────┐
   ▼                                                                           ▼
 [Контур Pre-VPN и Captive Portal]               [Межсетевой экран firewall4]
  ├── NTP (UDP/123 -> 194.190.168.1) ──► Прямой   ├── DNAT :53 ──► в sing-box DNS
  └── Travelmate (только TR3000):                 ├── Блок DoT :853 из br-lan
       └── При детекте портала открывает          ├── Блок QUIC (UDP/443 из br-lan)
           доступ к СМС-странице отеля            └── Блок утечек IPv6
                                                               │
                                                               ▼
                                      [Ядро sing-box (tun0, mtu 1400, GOMEMLIMIT=90MiB)]
                                       ├── Веб-панель: http://10.10.1.1:9090/ui (metacubexd)
                                       ├── Реклама (ads-all.srs) ──────────► REJECT
                                       ├── Сайты РФ (.ru / geoip-ru.srs):
                                       │     ├── DNS: DoH https://77.88.8.8/dns-query
                                       │     └── Выход: DIRECT (mark 0xff) ──► [WAN]
                                       └── Глобальный трафик (Proxy):
                                             ├── DNS: DoH https://1.1.1.1/dns-query (внутри!)
                                             └── Выход: Группа "GLOBAL-AUTO" (urltest, 120ms):
                                                   ├── Топ-20 Standard (VLESS-Reality / Hy2)
                                                   └── Все LTS-Mobile серверы
                                                         │ (исходящая метка: meta mark 0xff)
                                                         ▼
   ┌─────────────────────────────────────────────────────┴─────────────────────┐
   ▼ (На проводном Wi-Fi / Ethernet)                   ▼ (При подключении USB-телефона)
 [WAN / WWAN (metric 10)]                            [USB0 RNDIS Модем (metric 20)]
  └── Hotplug-триггер: urltest за 2 сек               └── Hotplug-триггер: urltest за 2 сек
      отсекает LTS и включает Standard-узлы               оживляет и включает LTS-узлы!
================================================================================
```

---

## 📄 5. Эталонные конфигурационные артефакты (LLD)

### 5.1. Межсетевой экран и защита от утечек: `/etc/nftables.d/99-crne-security.nft`
Этот файл автоматически включается в таблицу `inet fw4` при старте `firewall4`:

```nft
# /etc/nftables.d/99-crne-security.nft
# CRNE v2.0: NTP Bypass, DNS Hijack, Browser QUIC Block (Hysteria2-safe), IPv6 Block

chain crne_prerouting_dns {
    type nat hook prerouting priority dstnat - 5; policy accept;
    # 1. Принудительный заворот любого стороннего DNS (UDP/TCP 53) из LAN на роутер
    iifname "br-lan" ip daddr != 10.10.1.1 meta l4proto { tcp, udp } th dport 53 redirect to :53 comment "CRNE: Force Local Split-DNS"
}

chain crne_forward_guard {
    type filter hook forward priority filter - 5; policy accept;

    # 2. Блокировка транзитного IPv6 (защита от IPv6-утечек мимо туннеля)
    iifname "br-lan" meta nfproto ipv6 reject with icmpx type admin-prohibited comment "CRNE: Block IPv6 Leak"

    # 3. Блокировка DNS-over-TLS (порт 853) от клиентов в обход роутера
    iifname "br-lan" meta l4proto { tcp, udp } th dport 853 reject with icmpx type admin-prohibited comment "CRNE: Block External DoT"

    # 4. Подавление браузерного QUIC (UDP/443) строго для клиентов из br-lan.
    # ВАЖНО: Трафик самого sing-box (Hysteria2) идёт из цепочки output с меткой 0xff и сюда НЕ попадает!
    iifname "br-lan" oifname != "tun0" meta l4proto udp udp dport 443 reject with icmpx type admin-prohibited comment "CRNE: Force Browser TCP Fallback"
}

chain crne_output_bypass {
    type filter hook output priority filter - 5; policy accept;
    # 5. Разрешение прямого выхода NTP (UDP/123) для синхронизации часов при холодном старте
    meta l4proto udp udp dport 123 accept comment "CRNE: NTP Cold-Start Bypass"
    # 6. Пропуск всех пакетов sing-box (включая Hysteria2 и проверки urltest)
    meta mark 0x000000ff accept comment "CRNE: Allow sing-box Marked Outbound"
}
```
*Примечание для контура `Zapret` (`nfqws`):* если на роутере установлен `Zapret`, в его цепочку `postrouting` первой строкой добавляется правило `meta mark 0x000000ff return`, полностью исключающее модификацию пакетов ко всем 500+ серверам подписки.

---

### 5.2. Эталонный шаблон конфигурации ядра: `/etc/sing-box/config.json`
Шаблон совместим с `sing-box 1.10+ / 1.11+` (автоматически заполняется узлами скриптом `sub2sing.py`):

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
      "default_mode": "Rule"
    }
  },
  "dns": {
    "servers": [
      {
        "tag": "dns-proxy-doh",
        "address": "https://1.1.1.1/dns-query",
        "address_Fallback": "https://9.9.9.9/dns-query",
        "detour": "GLOBAL-AUTO"
      },
      {
        "tag": "dns-direct-ru",
        "address": "https://77.88.8.8/dns-query",
        "detour": "direct"
      },
      {
        "tag": "dns-block",
        "address": "rcode://success"
      }
    ],
    "rules": [
      {
        "rule_set": ["geosite-ads"],
        "server": "dns-block",
        "disable_cache": true
      },
      {
        "domain_suffix": [".ru", ".рф", ".su"],
        "rule_set": ["geosite-ru"],
        "server": "dns-direct-ru"
      }
    ],
    "final": "dns-proxy-doh",
    "strategy": "ipv4_only"
  },
  "inbounds": [
    {
      "type": "tun",
      "tag": "tun-in",
      "interface_name": "tun0",
      "inet4_address": "172.19.0.1/30",
      "mtu": 1400,
      "auto_route": true,
      "auto_redirect": true,
      "strict_route": true,
      "sniff": true,
      "sniff_override_destination": false
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
      "outbounds": [
        "STANDARD-TOP20",
        "LTS-MOBILE"
      ],
      "url": "https://www.youtube.com/generate_204",
      "interval": "2m",
      "tolerance": 120,
      "idle_timeout": "30m"
    },
    {
      "type": "urltest",
      "tag": "STANDARD-TOP20",
      "outbounds": ["__GENERATED_TOP20_STANDARD_NODES__"],
      "url": "https://www.youtube.com/generate_204",
      "interval": "2m",
      "tolerance": 120
    },
    {
      "type": "urltest",
      "tag": "LTS-MOBILE",
      "outbounds": ["__GENERATED_LTS_MOBILE_NODES__"],
      "url": "https://www.youtube.com/generate_204",
      "interval": "2m",
      "tolerance": 120
    },
    {
      "type": "selector",
      "tag": "MANUAL-ALL",
      "outbounds": ["GLOBAL-AUTO", "LTS-MOBILE", "STANDARD-TOP20", "__GENERATED_ALL_500_NODES__"]
    },
    {
      "type": "direct",
      "tag": "direct"
    },
    {
      "type": "block",
      "tag": "block"
    },
    {
      "type": "dns",
      "tag": "dns-out"
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
        "protocol": "dns",
        "outbound": "dns-out"
      },
      {
        "ip_is_private": true,
        "outbound": "direct"
      },
      {
        "rule_set": ["geosite-ads"],
        "outbound": "block"
      },
      {
        "domain_suffix": [".ru", ".рф", ".su"],
        "rule_set": ["geosite-ru", "geoip-ru"],
        "outbound": "direct"
      }
    ],
    "final": "GLOBAL-AUTO"
  }
}
```

---

### 5.3. Скрипт мгновенного переключения `Wi-Fi ↔ Мобильный USB` и `LTS`-узлов: `/etc/hotplug.d/iface/99-singbox-wan-switch`
Устанавливается на портативный роутер **Cudy TR3000** (и стационарный с резервным модемом):

```sh
#!/bin/sh
# /etc/hotplug.d/iface/99-singbox-wan-switch
# Мгновенный пересчёт urltest (активация/отсев LTS-серверов) при смене канала связи

[ "$ACTION" = "ifup" ] || [ "$ACTION" = "ifdown" ] || exit 0

case "$INTERFACE" in
    wan|wwan|usb0|trm_wwan)
        logger -t CRNE-WAN "Interface $INTERFACE changed state ($ACTION). Triggering sing-box urltest..."
        (
            sleep 2
            # 1. Сброс активных сессий, привязанных к старому интерфейсу
            wget -qO- --method=DELETE "http://10.10.1.1:9090/connections" >/dev/null 2>&1
            # 2. Немедленный параллельный прогон urltest по пулам Standard и LTS
            wget -qO- "http://10.10.1.1:9090/group/STANDARD-TOP20/delay?url=https://www.youtube.com/generate_204&timeout=1500" >/dev/null 2>&1
            wget -qO- "http://10.10.1.1:9090/group/LTS-MOBILE/delay?url=https://www.youtube.com/generate_204&timeout=1500" >/dev/null 2>&1
            wget -qO- "http://10.10.1.1:9090/group/GLOBAL-AUTO/delay?url=https://www.youtube.com/generate_204&timeout=1500" >/dev/null 2>&1
        ) &
        ;;
esac
```

---

### 5.4. Интеграция `travelmate` с перехватом DNS (прохождение Captive Portal в отеле/кафе)
Для того чтобы во время регистрации по СМС в публичном Wi-Fi принудительный перехват `DNAT :53` не блокировал локальный DNS-сервер отеля/кафе, создаётся хук `/etc/travelmate/crne-portal-hook.sh`:
```sh
#!/bin/sh
# Вызывается демоном travelmate при смене статуса подключения
STATUS="$1"
if [ "$STATUS" = "cp" ]; then
    # Обнаружен Captive Portal: временно отключаем перехват DNS, чтобы открылась СМС-страница
    nft flush chain inet fw4 crne_prerouting_dns 2>/dev/null
    logger -t CRNE-PORTAL "Captive Portal detected! DNS redirect paused for SMS login."
elif [ "$STATUS" = "connected" ]; then
    # Интернет открыт: возвращаем жёсткую защиту DNAT :53 и перезапускаем urltest
    /etc/init.d/firewall reload
    wget -qO- "http://10.10.1.1:9090/group/GLOBAL-AUTO/delay?url=https://www.youtube.com/generate_204&timeout=1500" >/dev/null 2>&1 &
    logger -t CRNE-PORTAL "Portal passed! Full CRNE security restored."
fi
```

---

### 5.5. Список сохранения при обновлении прошивки: `/etc/sysupgrade.conf`
Гарантирует выживаемость всей конфигурации при обновлении версий OpenWrt:
```text
/etc/sing-box/
/etc/nftables.d/99-crne-security.nft
/etc/hotplug.d/iface/99-singbox-wan-switch
/etc/travelmate/
/usr/bin/crne-sub-fallback.sh
```

---

## 📋 6. Инженерный регламент развёртывания (Пошаговая инструкция)

### Шаг 1: Инициализация базовой ОС, времени и лимитов памяти (Слой 0)
Выполняется по SSH на роутере (`Cudy WR3000S` или `Cudy TR3000`):
```sh
# 1. Настройка LAN 10.10.1.1/24, кольцевого лога в RAM (защита NAND) и российского NTP по IP
uci set network.lan.ipaddr='10.10.1.1'
uci set system.@system[0].log_type='circular'
uci set system.@system[0].log_size='512'
uci set system.@system[0].zonename='Europe/Moscow'
uci delete system.ntp.server
uci add_list system.ntp.server='194.190.168.1'
uci add_list system.ntp.server='ru.pool.ntp.org'
uci commit

# 2. Отключение Flow Offloading (обязательно для работы tun/TPROXY/Zapret) и включение MSS Clamping
uci set firewall.@defaults[0].flow_offloading='0'
uci set firewall.@defaults[0].flow_offloading_hw='0'
uci set firewall.@zone[0].mtu_fix='1'
uci set firewall.@zone[1].mtu_fix='1'
uci commit firewall
/etc/init.d/firewall reload
```

### Шаг 2: Установка пакетов и жёсткого лимита ОЗУ `GOMEMLIMIT=90MiB` (Слой 1)
```sh
opkg update
# Базовый набор (для обоих роутеров):
opkg install sing-box jq ca-bundle kmod-nft-tproxy kmod-tun

# Дополнительно только для портативного Cudy TR3000 (Wi-Fi порталы + USB-модем телефона):
opkg install travelmate luci-app-travelmate kmod-usb-net-rndis kmod-usb-net-cdc-ncm usb-modeswitch

# Фиксация лимита памяти Go (90 МБ) и запуска от root в службе sing-box:
uci set sing-box.main.enabled='1'
uci set sing-box.main.user='root'
uci set sing-box.main.conffile='/etc/sing-box/config.json'
uci commit sing-box
sed -i '/procd_open_instance/a \	procd_set_param env GOMEMLIMIT=90MiB' /etc/init.d/sing-box
```

### Шаг 3: Настройка двух каналов связи `Wi-Fi + USB` на `Cudy TR3000` (Слой 6)
```sh
# Интерфейс USB-тетеринга от смартфона (usb0) с метрикой 20:
uci set network.usb_wan=interface
uci set network.usb_wan.proto='dhcp'
uci set network.usb_wan.device='usb0'
uci set network.usb_wan.metric='20'

# Добавление usb_wan в зону WAN файрвола:
uci add_list firewall.@zone[1].network='usb_wan'
uci commit
/etc/init.d/network reload
```

### Шаг 4: Подготовка сборочного узла (ПК Windows и/или Android `Termux`)
На **Android-смартфоне** (в чистом приложении `Termux` из F-Droid / GitHub, **без использования `proot`/`chroot`**):
```sh
pkg update && pkg install -y python openssh sing-box
termux-wake-lock
```
Запуск сборки, локального теста всех 500+ узлов и безопасной отправки на роутер:
```sh
python3 sub2sing.py \
  --url "https://ВАШ_ПРОВАЙДЕР/sub/BASE64_KEY" \
  --tolerance 120 \
  --timeout 1500 \
  --top 20 \
  --test-local \
  --push root@10.10.1.1
```

### Шаг 5: Транзакционное применение конфигурации с защитой от потери связи (`/tmp/rollback.lock`)
При каждой выгрузке нового конфига или правил файрвола на роутере выполняется атомарный блок с 5-минутным таймером возврата:
```sh
# 1. Создание бэкапа и взведение флага отката
cp /etc/sing-box/config.json /etc/sing-box/config.json.bak 2>/dev/null
cp /etc/config/firewall /etc/config/firewall.bak
touch /tmp/rollback.lock

(sleep 300 && [ -f /tmp/rollback.lock ] && {
    logger -t CRNE-ROLLBACK "Rollback timer expired! Restoring working config..."
    cp /etc/sing-box/config.json.bak /etc/sing-box/config.json
    cp /etc/config/firewall.bak /etc/config/firewall
    rm -f /etc/nftables.d/99-crne-security.nft
    /etc/init.d/firewall reload
    /etc/init.d/sing-box restart
    rm -f /tmp/rollback.lock
}) &

# 2. Перезапуск служб с новой конфигурацией
/etc/init.d/firewall reload
/etc/init.d/sing-box restart
```
После успешного прохождения проверок из Раздела 7 таймер отменяется и временные копии удаляются одной командой:
```sh
rm -f /tmp/rollback.lock /etc/sing-box/config.json.bak /etc/config/firewall.bak
```

---

## ✅ 7. Приёмочные испытания (Acceptance Test Procedure — Чек-лист)

Система считается принятой в эксплуатацию только при прохождении всех 10 тестов таблицы:

| ID | Проверка | Команда / Действие | Эталонный результат (Критерий приёмки) | Действие при сбое |
| :- | :--- | :--- | :--- | :--- |
| **T1** | **Валидация синтаксиса и `.srs`** | `sing-box check -c /etc/sing-box/config.json` | Вывод пустой, код возврата `0`. | Проверить наличие файлов `.srs` в `/etc/sing-box/ruleset/`. |
| **T2** | **Бюджет ОЗУ под нагрузкой** | `free -m` и `awk '/VmRSS/{print $2,$3}' /proc/$(pidof sing-box)/status` | `VmRSS` процесса `sing-box` **≤ 95 МБ**, свободной памяти (`available`) **≥ 110 МБ**. | Проверить наличие `GOMEMLIMIT=90MiB` в `/etc/init.d/sing-box`. |
| **T3** | **NTP Cold-Start Bypass** | `ntpd -n -q -p 194.190.168.1` | Время синхронизируется за <3 сек даже при остановленном `sing-box`. | Проверить цепочку `crne_output_bypass` в `nft list ruleset`. |
| **T4** | **Работа туннеля и `urltest`** | `curl -x socks5://127.0.0.1:1603 -I -s -m 5 https://www.youtube.com/generate_204` | Ответ `HTTP/2 204` (или `HTTP/1.1 204 No Content`). В `http://10.10.1.1:9090/ui` видны пинги узлов. | Вызвать ручной тест в UI или запустить `sub2sing.py` заново. |
| **T5** | **Split-DNS и перехват `DNAT :53`** | На ПК/телефоне: `nslookup gosuslugi.ru 8.8.8.8` | Запрос к чужому `8.8.8.8` перехвачен роутером и вернул российский IP. На `dnsleaktest.com` — только зарубежный DoH (`1.1.1.1`/`9.9.9.9`). | Проверить цепочку `crne_prerouting_dns` в `nft list ruleset`. |
| **T6** | **Блок браузерного QUIC vs Живой `Hysteria2`** | В панели `10.10.1.1:9090/ui` выбрать узел `Hysteria2` в группе `MANUAL-ALL` и открыть YouTube | Сайт открывается через `Hysteria2` (`UDP/443`), при этом прямой `UDP/443` из LAN заблокирован (`nft list chain inet fw4 crne_forward_guard`). | Убедиться, что в `config.json` задан `"default_mark": 255`. |
| **T7** | **Переключение `Wi-Fi ↔ Мобильный USB` и `LTS`-узлы** | На `TR3000`: подключить телефон по USB (включить USB-модем), отключить Wi-Fi `wwan` | Через 2–3 секунды в `10.10.1.1:9090/ui` группа `LTS-MOBILE` показывает численный пинг (вместо `Unavailable`) и принимает трафик. | Проверить права `chmod +x /etc/hotplug.d/iface/99-singbox-wan-switch`. |
| **T8** | **Прохождение Captive Portal (СМС)** | Подключить `TR3000` к хотспоту с авторизацией (или тестовой точке доступа) | Статус `travelmate` переходит в `cp`, на телефоне открывается страница входа, после ввода кода восстанавливается `DNAT :53` и поднимается туннель. | Проверить привязку `/etc/travelmate/crne-portal-hook.sh`. |
| **T9** | **Проверка Kill Switch** | Временно указать неверный порт у выбранного узла в `MANUAL-ALL` | Целевые зарубежные сайты не открываются, реальный IP провайдера не светится на `2ip.ru` / `icanhazip.com`. | Проверить `strict_route: true` в секции `tun` `config.json`. |
| **T10** | **Нагрузочный тест скорости (Throughput)** | Скачивание 100 МБ через туннель: `curl -o /dev/null https://speed.cloudflare.com/__down?bytes=100000000` | Скорость на `MT7981B` **≥ 180–300 Мбит/с** (при условии свободного канала провайдера), загрузка CPU (`top`) оставляет ≥ 15% `idle`. | Убедиться, что `flow_offloading='0'` (не конфликтует с `tun0`). |

После успешного прохождения чек-листаT1–T10 создаётся эталонный архив восстановления:
```sh
sysupgrade -b /tmp/crne-v2-golden-backup.tar.gz
```
*(Файл скачивается по `scp` на ПК и в память смартфона `Termux`).*

---

## 🛡️ 8. Юридическая рамка и эксплуатационные границы

* **Правовой статус:** архитектура разработана исключительно для законного личного использования — криптографической защиты тайны связи в публичных сетях (отели, аэропорты, транспорт), предотвращения перехвата сессий и стабильного доступа к открытым мировым информационным ресурсам. В РФ использование VPN для личных нужд не запрещено; правовые риски возникают при публичной рекламе средств обхода блокировок, коммерческом предоставлении доступа третьим лицам или умышленном поиске заведомо экстремистских материалов (ст. 13.53 КоАП РФ). *Данная справка носит информационный характер и не является официальной юридической консультацией.*
* **Эксплуатационная гигиена при работе с коммерческими подписками:** поскольку выходные узлы управляются сторонним провайдером подписки, национальный трафик (банки, государственные порталы, локальные сервисы `.ru`/`.рф`) маршрутизируется по прямому контуру (`Direct`) с принудительным шифрованием DNS-запросов по протоколу DoH, а для всех зарубежных ресурсов используется сквозное шифрование HTTPS поверх туннеля.
