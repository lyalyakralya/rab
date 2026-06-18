# ПР №7. AppArmor, Capabilities и Docker

## 1. Linux Capabilities

### Разбор getcap /usr/bin/ping

**cap_net_raw=ep:**
- **cap_net_raw** - разрешает использование RAW и PACKET сокетов, необходимо для отправки ICMP пакетов (ping)
- **e (effective)** - capability активна и используется процессом
- **p (permitted)** - процесс может использовать эту capability

### CapPrm / CapEff / CapBnd - в чём разница

- **CapPrm (Permitted)** - набор capabilities, которые процесс может использовать (максимально возможные)
- **CapEff (Effective)** - capabilities, которые активны в данный момент
- **CapBnd (Bounding)** - максимальный набор capabilities, который процесс никогда не сможет получить

### setcap - демонстрация

**До:** python3 порт 80 → DENIED: порт 80 --- [Errno 13] Permission denied

**После setcap cap_net_bind_service=ep:** → OK: привязался к порту 80

**Почему лучше чем sudo:**
- Выдаётся только конкретное право (привязка к портам ниже 1024)
- Процесс не получает полный root доступ
- В случае эксплуатации уязвимости злоумышленник получит ограниченные возможности
- Принцип наименьших привилегий (least privilege)

### Флаги e, i, p в cap_net_raw+eip

- **e (effective)** - capability активна при запуске
- **i (inheritable)** - capability может быть передана дочерним процессам
- **p (permitted)** - capability разрешена для использования

## 2. AppArmor

### Количество профилей

enforce: N, complain: N

### Результаты pr07-reader

| Действие | Без профиля | complain | enforce |
|----------|-------------|----------|---------|
| Читать /tmp/pr07-allowed.txt | OK | OK | OK |
| Читать /etc/shadow | OK (Permission denied от ОС) | OK (Permission denied от ОС) | DENIED (AppArmor) |
| Писать в /tmp/pr07-output.txt | OK | OK | OK |
| Писать в /etc/pr07-hack.txt | Permission denied | Permission denied | DENIED (AppArmor) |

### Разбор строки DENIED
[timestamp] audit: type=1400 audit(...): apparmor="DENIED" operation="open" profile="/usr/local/bin/pr07-reader" name="/etc/shadow" pid=12345 comm="cat" requested_mask="r" denied_mask="r" fsuid=1000 ouid=0
- **operation="open"** - операция открытия файла
- **profile="/usr/local/bin/pr07-reader"** - какой профиль сработал
- **name="/etc/shadow"** - к какому файлу пытались обратиться
- **denied_mask="r"** - какое право было запрещено (чтение)
- **requested_mask="r"** - какое право запрашивалось

## 3. Docker - изоляция

| Ресурс | Хост | Контейнер |
|--------|------|-----------|
| Процессы | ~200-300 шт | ~10-15 шт |
| Сетевые интерфейсы | eth0, lo, docker0 | lo, eth0 (виртуальный) |
| /etc/shadow хоста | доступен | недоступен |
| Монтирование | разрешено | запрещено (Permission denied) |

### Capabilities: обычный vs --privileged

**Обычный CapEff:** [значение] → расшифровка

**--privileged CapEff:** [значение] → расшифровка

**Чего нет у обычного:** SYS_ADMIN, SYS_RAWIO, SYS_PTRACE, SYS_MODULE, SYS_TIME и многих других

**Почему --privileged опасен:**
- Даёт полный root доступ на хосте
- Может монтировать файловые системы
- Может изменять сетевые настройки хоста
- Может загружать/выгружать модули ядра
- Нарушает изоляцию контейнера

### Итоговый nginx

**Capabilities:** NET_BIND_SERVICE, CHOWN, DAC_OVERRIDE, SETGID, SETUID

**Почему именно эти:**
- NET_BIND_SERVICE - для привязки к порту 80 (привилегированный порт)
- CHOWN - для изменения владельца файлов
- DAC_OVERRIDE - для обхода прав доступа к файлам
- SETGID/SETUID - для смены группы/пользователя процесса

## 4. Эшелонированная защита

| Слой | Инструмент | Что ограничивает |
|------|------------|------------------|
| DAC | chmod/chown | Права доступа пользователей к файлам (чтение/запись/выполнение) |
| Capabilities | --cap-drop ALL + cap-add | Системные вызовы и привилегии процесса |
| MAC | AppArmor | Доступ к файлам и системным вызовам на уровне ядра |
| Изоляция | Docker namespaces | Процессы, сеть, файловая система, PID |

## Выводы

В ходе выполнения практической работы были изучены три уровня защиты:

1. **Linux Capabilities** - позволяют выдавать только необходимые привилегии, реализуя принцип наименьших привилегий

2. **AppArmor** - обеспечивает мандатный контроль доступа, ограничивая действия программ даже если они запущены от root

3. **Docker** - предоставляет изоляцию через namespaces и cgroups, но требует дополнительной настройки для безопасности

Комбинирование всех трёх уровней защиты обеспечивает эшелонированную защиту, где каждый уровень ограничивает возможности злоумышленника даже в случае компрометации одного из слоёв.



