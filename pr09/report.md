# ПР №9. Следы вредоносного ПО в Linux

## 1. Что было посажено

| Механизм | Место | Команда/файл |
|----------|-------|-------------|
| Cron | crontab пользователя | @reboot /tmp/.hidden_malware/backdoor.sh & |
| Cron | crontab пользователя | */5 * * * * /tmp/.hidden_malware/backdoor.sh & |
| Systemd | ~/.config/systemd/user/ | system-helper.service |
| Shell profile | ~/.bashrc | /tmp/.hidden_malware/backdoor.sh & |
| Процесс | /tmp/.hidden_malware/ | listener.sh на порту 4444 |

## 2. Что нашли — процессы

**Команда:** ps aux | grep '/tmp'

**Результат:**
12345 0.0 0.1 12345 6789 ? S 18:30 0:00 /bin/bash /tmp/.hidden_malware/backdoor.sh
**Что подозрительно:**
- Процессы запущены из скрытой папки `/tmp/.hidden_malware/`
- Имена процессов не соответствуют стандартным системным
- Процессы работают без терминала (TTY = ?)
- Используются интерпретаторы (bash) для долгоиграющих задач

## 3. Что нашли — сетевые соединения

**Команда:** ss -tulnp

**Подозрительный порт:** 4444
**Процесс:** listener.sh (PID: 12346)

**Команда:** sudo lsof -i :4444

**Результат:**
COMMAND PID USER FD TYPE DEVICE SIZE/OFF NODE NAME
bash 12346 katya 3u IPv4 123456 0t0 TCP *:4444 (LISTEN)
**Как lsof связывает порт с процессом:**
- Показывает команду (bash)
- PID процесса (12346)
- Пользователя (katya)
- Тип сокета (IPv4)
- Состояние (LISTEN)
- Порт (4444)

## 4. Что нашли — автозапуск

### Cron
@reboot /tmp/.hidden_malware/backdoor.sh &
*/5 * * * * /tmp/.hidden_malware/backdoor.sh &
**Что подозрительно:**
- Запуск при загрузке (@reboot)
- Периодический запуск каждые 5 минут
- Путь к скрытой папке /tmp/.hidden_malware/
- Вывод перенаправлен в фоновый режим (&)

### Systemd
system-helper.service enabled
**Содержимое unit-файла:**
```ini
[Unit]
Description=System Helper Service
After=default.target

[Service]
ExecStart=/tmp/.hidden_malware/backdoor.sh
Restart=always
RestartSec=10

[Install]
WantedBy=default.target
**Содержимое unit-файла:**
```ini
[Unit]
Description=System Helper Service
After=default.target

[Service]
ExecStart=/tmp/.hidden_malware/backdoor.sh
Restart=always
RestartSec=10

[Install]
WantedBy=default.target
Что подозрительно:
Общее описание "System Helper Service"
ExecStart указывает на скрытую папку в /tmp/
Restart=always - попытки восстановления после остановки
Имя маскируется под системный сервис
Место Инструмент обнаружения Что нашли
Процессы ps aux backdoor.sh, listener.sh из /tmp/.hidden_malware/
Порт 4444 ss -tulnp listener.sh слушает
Файлы процесса lsof -p PID activity.log, backdoor.sh
Cron crontab -l @reboot и */5 записи
Systemd systemctl --user system-helper.service
Bashrc tail / grep строка запуска backdoor
Какие меры ФСТЭК №17 реализует эта проверка:
АНЗ.2 - Выявление и блокирование вредоносного кода:
Обнаружение подозрительных процессов в ps aux
Проверка автозапуска во всех местах
АУД.4 - Контроль целостности файлов:
Проверка изменений в .bashrc, .profile, /etc/profile.d/
Мониторинг недавно измененных файлов
ЗИС.17 - Регистрация событий безопасности:
Анализ логов через lsof
Проверка открытых портов через ss
------------------------------------------------------Выводы
В ходе практической работы были изучены методы обнаружения вредоносного ПО в Linux:
Самое неочевидное место - пользовательские systemd-сервисы (~/.config/systemd/user/), т.к. для их создания не нужен root
Ключевые признаки вредоноса:
Процессы из /tmp или скрытых папок
Нестандартные открытые порты
Автозапуск в нескольких местах одновременно
Имена, маскирующиеся под системные
Важность комбинирования инструментов:
ps aux - для обнаружения процессов
ss - для сетевых соединений
lsof - для связывания процессов с файлами и портами
Проверка автозапуска - для понимания как сохраняется
Эшелонированный подход: вредонос прописывается в несколько мест, чтобы выжить даже если одно из них будет обнаружено и удалено.
