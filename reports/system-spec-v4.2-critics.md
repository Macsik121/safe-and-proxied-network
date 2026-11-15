## Оценка спецификации v4.6

### ✅ Что действительно исправлено

**1. Hysteria2 в эталонном `config.json` — закрыто**
В §5.2 появился `hy2-template-01`:
```json
{"type": "hysteria2", "tag": "hy2-template-01", "server": "198.51.100.11",
 "password": "REPLACE_WITH_HY2_PASSWORD",
 "obfs": {"type": "salamander", ...}, "tls": {...}}
```
`GLOBAL-AUTO`, `STANDARD-TOP20` и `MANUAL-SELECT` теперь включают Hysteria2. Документарный разрыв между §5.2 и Приложением А закрыт (при условии, что `--init-template` тоже обновлён — по предоставленному фрагменту это не проверить).

**2. `ip rule del prio 100-199` — закрыто**
Заменено на итеративный `while ip rule del pref N 2>/dev/null; do :; done`. Синтаксически корректно, хотя удаляет **все** правила с `pref 100`, включая потенциально чужие. Для pet-проекта допустимо.

---

### ❌ ADR-11 «fail-closed fix» — заявлен, но архитектурно не работает

Это главная проблема v4.6. Раздел §1.1 утверждает: «клиент с `OUTBOUND=lts` направляется в таблицу `lts_only` (`default dev tun0`) — при падении tun0 его трафик **отбрасывается**, а не уходит в WAN». Реализация этого не обеспечивает.

#### Дефект 1 (критический): nft/ip rule не маршрутизируют через LTS-MOBILE

Механизм:
```
LTS client → ip rule prio 110 → lts_only table → default dev tun0
                                                     ↓
                                              tun0 (sing-box)
                                                     ↓
                                         route.rules: final = GLOBAL-AUTO
```
`lts_only` ведёт в **тот же tun0**, что и у всех VPN-клиентов. Внутри sing-box **нет** правила `source_ip_cidr → LTS-MOBILE`. Значит, трафик LTS-клиента выходит через `GLOBAL-AUTO` (urltest, где LTS-узлы — лишь часть плоского списка), а не через `LTS-MOBILE`. Различие между `vpn` и `lts` в конфиге **не реализовано**.

Чтобы это работало, нужен rule в `sing-box route.rules`:
```json
{"source_ip_cidr": ["10.10.1.60/32"], "outbound": "LTS-MOBILE"}
```
Такого правила в §5.2 нет.

#### Дефект 2 (критический): Clash API PUT — сломан

```sh
curl -s -X PUT "$SB_API/proxies/LTS-MOBILE" \
     -H "Authorization: Bearer $(cat /etc/crne-alert.conf 2>/dev/null | awk -F= '/^CLASH_SECRET=/{print $2}')" \
     -d '{"name":"LTS-MOBILE"}'
```

Три ошибки:
- **Неверный источник секрета.** `CLASH_SECRET` не определён в `/etc/crne-alert.conf` (там `TG_BOT_TOKEN`, `TG_CHAT_ID`, `SMS_GATEWAY`, `EMAIL_RECIPIENT`, `SITE_TAG`). Секрет Clash API лежит в `/etc/sing-box/config.json` → `.experimental.clash_api.secret`, читается через `jq`, как во всех остальных скриптах.
- **PUT на `urltest`-группу.** `LTS-MOBILE` — это `"type": "urltest"`, а не `"selector"`. Clash API PUT `/proxies/{name}` работает для селекторов; для `urltest` он либо вернёт ошибку, либо проигнорируется. Пиннить нужно `MANUAL-SELECT` на `LTS-MOBILE` — но тогда **все** клиенты пойдут через LTS, а не только помеченные.
- Даже если бы PUT сработал, это **глобальное** переключение селектора, а не per-client маршрутизация.

#### Дефект 3 (высокий): `lts_only` не fail-closed при падении tun0 в момент применения

```sh
ip route add default dev "$TUN_DEV" table "$LTS_TABLE_ID" 2>/dev/null || \
    logger -t CRNE-CLIENTS "WARN: cannot add default dev $TUN_DEV to lts_only table (tun0 down?)"
```

Если `tun0` в момент запуска скрипта отсутствует, route **не добавляется**, скрипт пишет WARN и продолжает. Тогда `lts_only` пуста, `ip rule lookup lts_only` не находит маршрут, пакет **проваливается в main table → WAN → утечка**. Заявленный fail-closed не работает.

**Исправление:** добавить `unreachable`-маршрут как fallback:
```sh
ip route add default dev "$TUN_DEV" table "$LTS_TABLE_ID" 2>/dev/null || \
    ip route add unreachable default table "$LTS_TABLE_ID"
```

---

### ⚠️ Оставшиеся пробелы (из моего предыдущего списка)

| Пункт | Статус в v4.6 |
|---|---|
| **FR-02** (ручные списки «всегда VPN/напрямую») | ❌ Не закрыт |
| **FR-13** (LuCI-интеграция) | ❌ Только Clash API UI |
| **FR-16 / NFR-14** (простая диагностика) | ❌ Нет `crne-diag` и упрощённой инструкции |
| **AT-04** (тест IPv6-утечки) | ❌ Нет T-теста |
| **ADR-12: NTP в РФ-юрисдикции** | ❌ Не обосновано, противоречит ADR-08 |
| **SHA256 плейсхолдеры** в `download-assets.sh` | ❌ Не заполнены (из Приложения Б) |
| **`--secret` reuse при сбое SSH** | ❌ Может ротировать токен (из Приложения А) |
| **`build_router_config` fallback групп** | ❌ `std_tags or flat_auto_tags` смешивает пулы |
| **NFR-04** (переключение ≤ 90 с) | ⚠️ `urltest interval: 2m` = 120 с |
| **`--init-template` с Hysteria2** | ⚠️ По предоставленному фрагменту не проверить |

---

### 📊 Итоговая оценка

| Категория | v4.5 | v4.6 | Комментарий |
|---|---|---|---|
| **Соответствие требованиям** | ~93–95% | **~93–95%** | Hysteria2 закрыт, но ADR-11 не работает → нетто-прирост ~0 |
| **Качество LLD** | Среднее | **Среднее-низкое** | ADR-11 v4.6 добавил 3 новых бага вместо исправления |
| **Честность документа** | Средняя | **Снижена** | §1.1 утверждает «устранена утечка R-DR5», хотя механизм не маршрутизирует через LTS и не fail-closed при отсутствии tun0 |

**Вердикт:** v4.6 — это **реакция на аудит**, а не его **закрытие**. Hysteria2 в §5.2 — реальный прогресс. Но ADR-11 «fix» — это **ложное исправление**: заявленная защита от утечки LTS-трафика в WAN не реализована. Для pet-проекта это допустимо как «фича в разработке», но документ **не должен** утверждать, что R-DR5 устранена.

**Что делать дальше (в порядке приоритета):**

1. **Либо честно упростить ADR-11:** убрать `OUTBOUND=lts` из спецификации, оставить только `direct` (bypass) и `vpn` (default). Тогда §1.1 перестаёт лгать, и документ становится консистентным.
2. **Либо реализовать LTS правильно:**
   - В `sing-box route.rules` добавить `{"source_ip_cidr": ["<LTS_CLIENT_IP>/32"], "outbound": "LTS-MOBILE"}` — это единственный рабочий способ per-client маршрутизации внутри sing-box.
   - Убрать `lts_only` table и `ip rule prio 110` — они не нужны и создают путаницу.
   - Убрать сломанный Clash API PUT.
3. **Добавить `unreachable` fallback** для `lts_only` (если оставляете таблицу) или для `main` при отсутствии tun0.
4. Заполнить SHA256 в `download-assets.sh` или перенести в «Known Limitations».
5. Добавить `sys.exit(1)` в `sub2sing.py` при `--push` без `--secret` и неудаче чтения секрета.
6. Добавить `crne-diag` и `crne-clients.conf` → `sing-box route.rules` вместо `ip rule`.

**После исправления пункта 2 (правильная LTS-маршрутизация через `route.rules`) спецификация достигнет ~96–97% соответствия.** Оставшиеся 3–4% — это осознанные компромиссы (NTP в РФ, отсутствие LuCI, плейсхолдеры SHA256), которые нужно явно вынести в раздел «Known Limitations / Out of Scope», а не замалчивать.