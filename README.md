# Отключить звуковой сигнал ИБП (USB) в Synology NAS
Небольшой скрипт для отключения/включения звукового сигнала ИБП, подключенного через USB, в Synology DiskStation Manager.

Некоторое время назад в 4 утра на долгое время отключилось электричество, и мне пришлось встать и выключить ИБП, чтобы прекратить звуковой сигнал. В вышеупомянутом сообщении описывается, как отключить его навсегда. Тем не менее, мне не хотелось, чтобы он переставал подавать звуковой сигнал в дневное время. Следующие шаги и сценарии — это мое решение для отключения и включения звукового сигнала в определенное время дня (в моем случае между 00 и 09 часами).

**Важно:** обновления DSM, похоже, удаляют сценарии в /root/ и откатывают файл upsd.users до исходного состояния. Если вы обновите DSM, вам придется повторить процесс

## вступление
В большинстве дистрибутивов Linux имеется набор инструментов для управления ИБП. А именно:
* `upsc` - клиент ИБП, который можно использовать для запроса подробной информации об ИБП.
* `upsd` - the service / daemon
* `upscmd` - инструмент командной строки, используемый для отправки команд и изменения настроек.

### Некоторые примеры:

Список доступных ИБП:
```shell
user@nas:/$ upsc -l
ups
```

Перечислите все переменные и значения из конкретного ИБП:
```shell
panda@calvin:/$ upsc ups
battery.charge: 100
battery.type: PbAc
device.mfr: EATON
(...)
ups.beeper.status: enabled
(...)
```
Просмотр определенного поля/переменной:
```shell
panda@calvin:/$ upsc ups ups.beeper.status
enabled
```

Оба upscобщаются upscmd, используя upsdмодель клиент/сервер (порт TCP 3493). К сожалению, upsmcd недоступен в ОС Synology DiskStation Manager (DSM). Тем не менее, мы можем подключиться к Telnet и эмулировать отправляемые upsdкоманды . upscmdПротокол описан [здесь](https://networkupstools.org/docs/developer-guide.chunked/ar01s09.html). В нашем конкретном случае мы просто хотим выполнить 3 действия (4 сообщения):
1. Авторизоваться
    - `USERNAME <user>`
    - `PWD <pwd>`
2. Включить или отключить звуковой сигнал
    - `INSTCMD <upsname> <command>`
3. Выйти
    - `LOGOUT`


## Как:

### 1. SSH в NAS
Активируйте SSHd на панели управления Synology, если вы этого еще не сделали, и подключитесь к нему по SSH. Примечание: под Windows вам может понадобиться приличная консоль с ssh (git bash?) или использовать PuTTY.
```shell
ssh <username>@<nas_ip> -p <ssh_port>
```
### 2. Найдите файл конфигурации пользователя.
Файл upsd.users может находиться в двух местах, найдите нужное:
```shell
user@nas:/$ find /usr/syno/etc/ups/ /etc/ups/ -name "upsd.users"
(result /path/to/upsd.users)
```

### 3. Добавьте нового пользователя в upsd.
Отредактируйте файл upsd.users (в зависимости от местоположения файла согласно предыдущему шагу) и добавьте новую учетную запись пользователя с разрешениями на изменение статуса звукового сигнала.

Для файла внутри _/usr/syno/etc/ups_
```shell
user@nas:/$ sudo vim /usr/syno/etc/ups/upsd.users
Password: <insert your pwd>
```
Для файла внутри _/etc/ups_
```shell
user@nas:/$ sudo vim /etc/ups/upsd.users
Password: <insert your pwd>
```

Будьте осторожны с редактором VIM! Если вы с ним не знакомы:
* Перемещайте курсор вниз, <kbd>&#8595;</kbd> пока не найдете место, куда вы хотите добавить новые строки.
* Нажмите <kbd>I</kbd> чтобы войти в РЕЖИМ ВСТАВКИ
* Отредактируйте файл по мере необходимости
* Нажмите <kbd>Esc</kbd> клавишу, чтобы выйти из режима редактирования.
* Нажмите <kbd>:</kbd> чтобы подать команду
* Напишите "wq" (напишите выход) и нажмите <kbd>Enter</kbd>

Итак, отредактируйте файл upsd.users и добавьте нового пользователя с правами на включение/отключение звукового сигнала (замените <upsd_username>и <upsd_pwd>нужными значениями):
```shell
    [<upsd_username>]
        password = <upsd_pwd>
        actions = SET
        instcmds = beeper.enable beeper.disable ups.beeper.status
```

### 4. Создайте скрипт Python для выдачи команд.
```shell
synoservice --restart ups-usb
(wait a few seconds)
```

### 5. Создайте скрипт Python для выдачи команд.
```shell
sudo vim /root/upscmd.py
```

Содержимое скрипта upscmd.py:
```python
#!/bin/python2
import sys
import telnetlib

user = "<the upsd_username set in upsd.users>"
pwd = "<the upsd_pwd set in upsd.users>"

if len(sys.argv) == 2:
    cmd = sys.argv[1]
else:
    print("the ups command to issue is missing.")
    print("example: upscmd.py beeper.enable")
    exit(1)
    

tn = telnetlib.Telnet("127.0.0.1", 3493)

tn.write("USERNAME {0}\n".format(user))
response = tn.read_until("OK", timeout=2)
print "USERNAME cmd status: {0}".format(response.strip())

tn.write("PASSWORD {0}\n".format(pwd))
response = tn.read_until("OK", timeout=2)
print "PASSWORD cmd status: {0}".format(response.strip())

tn.write("INSTCMD ups {0}\n".format(cmd))
response = tn.read_until("OK", timeout=2)
print "INSTCMD cmd status: {0}".format(response.strip())

tn.write("LOGOUT\n")
print tn.read_all()
```
**Примечание**: Вы можете просто клонировать репозиторий в папку на вашем NAS, отредактировать файл, чтобы установить пользователя/пароль, а затем скопировать это и следующее ups_beeper_control.shв нужное место, например:
```shell
sudo cp /volume<N>/<path_to_your_file>/upscmd.py /root/
```

### 6.Создайте сценарий bash для вызова сценария.
Может быть пропущено или выполнено в одном скрипте. Он используется для вызова upscmd.pyсценария с помощью «beeper.enable» или «beeper.disable» в зависимости от времени суток, поскольку я хочу переустанавливать его не только в определенное время (через cron), но и при загрузке NAS. См. ups_beeper_control.sh.

### 7. Сделайте скрипты исполняемыми
```shell
sudo chmod u+x upscmd.py
sudo chmod u+x ups_beeper_control.sh
```

На этом этапе вы можете протестировать скрипты:
```shell
user@nas:/$ sudo /root/ups_beeper_control.sh disable
Password:
disable beeper...
USERNAME cmd status: OK
PASSWORD cmd status: OK
INSTCMD cmd status: OK

OK Goodbye

Waiting 5 seconds for UPS to update state...
Beeper disabled.

user@nas:/$ sudo /root/ups_beeper_control.sh curtime
Using current time to set UPS beeper status
enable beeper...
USERNAME cmd status: OK
PASSWORD cmd status: OK
INSTCMD cmd status: OK

OK Goodbye

Waiting 5 seconds for UPS to update state...
Beeper enabled.
```

### 8. Запланируйте это
Перейдите в веб-интерфейс DSM (Панель управления -> Планировщик задач) и добавьте 3 задачи:
1. Запланированная задача по включению звукового сигнала (ежедневно)
    - пользователь: root
    - график: ежедневно в 9 утра
    - команда: `bash /root/ups_beeper_control.sh enable`
    - отправлять электронное письмо, когда сценарий завершается ненормально
2. Запланированная задача по отключению звукового сигнала (аналогично приведенному выше)
3. Запускаемая задача включения/отключения при загрузке по текущему времени
    - команда: `bash /root/ups_beeper_control.sh curtime`
    
