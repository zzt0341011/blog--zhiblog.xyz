---
title: "Руководство по очистке майнинг-трояна на 3x-ui"
description: "Полное руководство по диагностике и очистке сервера с панелью 3x-ui, заражённого майнинг-трояном: поиск процессов с высокой загрузкой CPU, идентификация майнера Monero, удаление crontab и systemd-служб, усиление безопасности SSH."
bookCollapseSection: true
---

Проект 3x-ui: https://github.com/MHSanaei/3x-ui

3x-ui — это модифицированная версия x-ui с обновлённым ядром, поддержкой новых протоколов и многопользовательского режима. Одна из полезных функций — возможность модификации ядра xray. По дизайну и плавности анимаций 3x-ui заметно превосходит другие прокси-панели.

![3x-ui--10001.jpg](https://zhiblog.xyz/img/3x-ui--10001.jpg)

Некоторое время назад была раскрыта критическая уязвимость в 3x-ui, после чего многие серверы были взломаны. На моём сервере после взлома был установлен майнинг-сервис. При входе в 3x-ui через терминал [FinalShell](https://www.hostbuf.com/) я заметил, что загрузка CPU постоянно составляла 100%.

![3x-ui--10002.jpg](https://zhiblog.xyz/img/3x-ui--10002.jpg)

Команда `top` показала процесс с загрузкой CPU 96,4% и именем `system`. На обычном Debian системный процесс называется `systemd`, поэтому это вызвало подозрения.

![3x-ui--10003.jpg](https://zhiblog.xyz/img/3x-ui--10003.jpg)

Я обратился за помощью к ChatGPT.

![3x-ui--10004.jpg](https://zhiblog.xyz/img/3x-ui--10004.jpg)

| Провайдер VPS | Минимальная цена | Особенности | Способы оплаты |
| :--- | :--- | :--- | :--- |
| [BWG Bandwagon](https://bandwagonhost.com/vps-hosting.php?aff=73801) | $49.99/год | **Самый быстрый и стабильный** | WeChat, Alipay, PayPal, банковская карта |
| [DMIT](https://www.dmit.io/cart.php?aff=8412) | $36.9/год | CN2 GIA, Hong Kong CMI | **Alipay**, PayPal, банковская карта |
| [hostdare](https://bill.hostdare.com/aff.php?aff=3766&pid=60) | $16/год | Большой трафик, быстрый CMI | **Alipay**, PayPal, банковская карта |
| [BuyVM](https://my.frantech.ca/aff.php?aff=7175) | ¥18 | Большой диск, безлимитный трафик, разблокировка стриминга | **Alipay**, PayPal, банковская карта, криптовалюта |
| [RackNerd](https://my.racknerd.com/aff.php?aff=6665) | $8.89/год | **Дёшево**, большой трафик, высокие характеристики | **Alipay**, PayPal, банковская карта, криптовалюта |
| [CloudCone](https://app.cloudcone.com.cn/vps/265/create?token=mcs-24-ssd-vps-1&ref=11035) | $13.99/месяц | **Дёшево**, большой трафик, большой диск | **Alipay**, PayPal, банковская карта |
| [Vultr](https://www.vultr.com/?ref=9560397) | $5/месяц | Десятки локаций по миру, $100 при регистрации | **Alipay**, PayPal, криптовалюта |

Примерный порядок действий от ChatGPT:

---

Информация о процессе:

```
PID: 511620  
USER: root  
CPU: 96.7%  
TIME+: 7d+9h  
COMMAND: system
```

Важные моменты:

1. **Имя процесса `system` не является нормальным именем системного демона Linux**
   - Основные системные процессы обычно называются `systemd` (не `system`), поэтому это имя вызывает подозрения.
   - Многие майнинг-трояны и вредоносные скрипты маскируются под имена вроде `system`, `kworker`, `bash`.

2. **Работает уже 7 дней 9 часов с высокой загрузкой CPU**
   - Длительная высокая загрузка часто указывает на майнинг или бесконечный цикл.

3. **Запущен от root**
   - Если это вредонос, значит он получил права root, что делает угрозу серьёзнее.

### Сначала определите, что это

Выполните:

```bash
ps -fp 511620
```

чтобы увидеть полный путь и команду запуска.

Затем:

```bash
ls -l /proc/511620/exe
```

чтобы найти исполняемый файл.

Для проверки сетевых соединений:

```bash
netstat -pantu | grep 511620
```

или:

```bash
ss -pantu | grep 511620
```

### Если подтверждено, что процесс вредоносный

1. **Немедленно завершите его**

```bash
kill -9 511620
```

2. **Удалите файлы**  
   Если `/proc/511620/exe` указывает на существующий файл:

```bash
rm -f /путь/к/файлу
```

3. **Очистите автозапуск**  
   Проверьте cron и службы автозапуска:

```bash
crontab -l
ls /etc/cron* -R
cat /etc/rc.local
systemctl list-unit-files | grep enabled
```

4. **Проверьте подозрительных пользователей**

```bash
cat /etc/passwd | grep -v '/nologin'
```

### Предотвращение повторного заражения

- Обновите систему:

```bash
apt update && apt upgrade -y
```

- Смените пароль root:

```bash
passwd
```

- Отключите удалённый вход root по SSH (отредактируйте `/etc/ssh/sshd_config`, установите `PermitRootLogin no`, затем `systemctl restart ssh`).
- Установите fail2ban или другие средства защиты от брутфорса.

---

Я заподозрил, что процесс `system` — это **майнинг-троян**. Следуя подсказкам ChatGPT, я подтвердил, что это программа для майнинга Monero, замаскированная под системный процесс.

![3x-ui--10005.jpg](https://zhiblog.xyz/img/3x-ui--10005.jpg)

Дальнейшие шаги очистки:

### Доказательства

- **`./Download/system`**: программа находится в каталоге `Download` под именем `system`.
- **`-o pool.hashvault.pro:443`**: `hashvault.pro` — домен пула для майнинга Monero.
- **`-u 8AMoGmzK...`**: адрес кошелька злоумышленника; сервер майнил для кого-то другого.
- **`-k --tls`**: типичные параметры программы `xmrig` (передача по TLS).

### Что делать

#### 1. Немедленно завершите процесс

```bash
kill -9 511620
```

#### 2. Удалите майнер

```bash
rm -f /root/Download/system
```

Если каталог неизвестен:

```bash
ls -l /root/Download
```

#### 3. Найдите похожие файлы (на случай копий)

```bash
find / -type f -name "system" -o -name "xmrig" 2>/dev/null
```

#### 4. Проверьте автозапуск (чтобы не запустился после перезагрузки)

```bash
crontab -l
ls -l /etc/cron* /var/spool/cron* 2>/dev/null
grep -r "Download/system" /etc/ 2>/dev/null
```

#### 5. Проверьте подозрительные службы

```bash
systemctl list-units --type=service | grep system
```

#### 6. Смените пароль

```bash
passwd
```

Также рекомендуется отключить удалённый вход root. Отредактируйте:

```bash
nano /etc/ssh/sshd_config
```

Измените на:

```
PermitRootLogin no
```

Затем:

```bash
systemctl restart ssh
```

---

### Более глубокая проверка, если троян хорошо скрыт

Часто троян прописывается в crontab пользователя или в systemd-службу.

#### 1. Проверьте crontab пользователя root

```bash
crontab -l -u root
```

Если там есть команды с `Download/system` или `xmrig`, удалите их. Редактирование:

```bash
crontab -e -u root
```

#### 2. Поиск команд майнинга во всех файлах cron

```bash
grep -r "Download/system" /etc/cron* /var/spool/cron* 2>/dev/null
```

#### 3. Проверка служб автозапуска systemd

Многие майнеры маскируются под `system-update.service`, `network.service` и т.п.:

```bash
systemctl list-unit-files --type=service | grep enabled
systemctl status system
systemctl status system-update
```

Если служба указывает на `Download/system` или неизвестный каталог, отключите и удалите её:

```bash
systemctl disable --now имя_службы
rm -f /etc/systemd/system/имя_службы
systemctl daemon-reload
```

#### 4. Проверка `rc.local` и скриптов запуска

```bash
cat /etc/rc.local
ls -l /etc/init.d/
grep -r "Download/system" /etc/init.d/
```

#### 5. Предотвращение повторного заражения

- **Смена пароля**: `passwd`
- **Запрет входа root по SSH** (в `/etc/ssh/sshd_config` установить `PermitRootLogin no`)
- **Только вход по ключу** (`PasswordAuthentication no`)
- Установка Fail2ban для защиты SSH от брутфорса:

```bash
apt install fail2ban -y
```

---

### Реально найденный троян в crontab

Первая строка crontab оказалась задачей трояна:

```bash
* * * * * pgrep -x ibus || exec /root/.cache/ibus/ibus &
```

Это означает:

- Каждую минуту проверяется наличие процесса с именем `ibus`
- Если его нет, запускается `/root/.cache/ibus/ibus` (на самом деле майнинг-троян, замаскированный под метод ввода ibus)
- Даже после завершения процесса он снова запустится через минуту

Вторая строка (`~/.acme.sh/acme.sh --cron`) — обычная задача продления сертификата, она не вредоносна.

### Способ очистки

1. **Удалите задачу трояна из crontab**

```bash
crontab -e -u root
```

Удалите строку:

```bash
* * * * * pgrep -x ibus || exec /root/.cache/ibus/ibus &
```

Сохраните и выйдите.

2. **Завершите работающий процесс трояна**

```bash
pkill -f "/root/.cache/ibus/ibus"
```

3. **Удалите файлы трояна**

```bash
rm -rf /root/.cache/ibus
```

4. **Проверьте, нет ли связанной службы systemd**

```bash
systemctl list-unit-files --type=service | grep ibus
systemctl status ibus
```

Если найдена подозрительная служба ibus:

```bash
systemctl disable --now ibus
rm -f /etc/systemd/system/ibus.service
systemctl daemon-reload
```

В конце удалите оставшиеся демоны и задачи, смените пароль администратора и перезагрузите сервер.
