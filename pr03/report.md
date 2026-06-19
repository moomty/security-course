## 1. Пользователи и группы

### Созданные пользователи и их роли

| Пользователь | Группа | Роль |
| :--- | :--- | :--- |
| alice | developers | Администратор проекта |
| bob | developers | Разработчик |
| carol | auditors | Аудитор |

### Проверка создания пользователей
$ getent passwd alice bob carol
alice:x:1001:1003::/home/alice:/bin/bash
bob:x:1002:1004::/home/bob:/bin/bash
carol:x:1003:1005::/home/carol:/bin/bash

![Создание пользователей](screens/1_passwd.png)

### Разбор полей `/etc/passwd` (на примере строки alice)

| Поле | Значение | Описание |
| :--- | :--- | :--- |
| `alice` | Имя пользователя | Логин для входа в систему |
| `x` | Заглушка пароля | Пароль хранится в `/etc/shadow` |
| `1001` | UID | Идентификатор пользователя |
| `1003` | GID | Идентификатор основной группы |
| (пусто) | GECOS | Полное имя пользователя |
| `/home/alice` | Домашняя директория | Персональная папка пользователя |
| `/bin/bash` | Оболочка | Интерпретатор команд при входе |

### Проверка `/etc/shadow`
$ sudo grep -E 'alice|bob|carol' /etc/shadow
alice:$6$...:19800:0:99999:7:::
bob:$6$...:19800:0:99999:7:::
carol:$6$...:19800:0:99999:7:::

В `/etc/shadow` хранятся хэши паролей, даты последней смены, сроки действия пароля и политики. Файл недоступен обычным пользователям (права `-rw-r-----`, владелец root, группа shadow), потому что хэши паролей — критичная информация. Если злоумышленник получит хэши, он сможет подобрать пароль методом перебора.

![Хэши паролей](screens/2_shadow.png)

## 2. Права доступа chmod/chown

### Настройка /srv/project/code

sudo chown alice:developers /srv/project/code
sudo chmod 750 /srv/project/code
Права 750: владелец — rwx (7), группа — r-x (5), остальные — --- (0).
text
$ ls -la /srv/project/
total 20
drwxr-xr-x 5 root root       4096 Jun 19 17:03 .
drwxr-xr-x 3 root root       4096 Jun 19 17:03 ..
drwxr-x--- 2 alice developers 4096 Jun 19 17:03 code
drwxr-xr-x 2 root root       4096 Jun 19 17:03 logs
drwxr-xr-x 2 root root       4096 Jun 19 17:03 reports
https://screens/3_ls_project.png
Проверка доступа bob (в группе developers)
$ su - bob
bob@Moomty1:~$ ls /srv/project/code
ls: cannot open directory '/srv/project/code': Permission denied
bob@Moomty1:~$ touch /srv/project/code/test.py
touch: cannot touch '/srv/project/code/test.py': Permission denied
При правах 750 bob не смог создать файл, потому что у группы было только r-x (чтение и выполнение), но не было права w (запись). Для разрешения записи группе нужно добавить право w — например, chmod 770 или chmod g+w.
Проверка после изменения прав на 770
sudo chmod 770 /srv/project/code
$ su - bob -c 'touch /srv/project/code/hello.py'
touch: cannot touch '/srv/project/code/hello.py': Permission denied
$ su - bob -c 'ls /srv/project/code'
ls: cannot open directory '/srv/project/code': Permission denied
после chmod 770 доступ у bob всё равно отсутствует.
при использовании su - bob -c '...' пароль не был введён (команда выполняется без интерактивного ввода пароля). Для корректной проверки нужно сначала переключиться на bob через su - bob, ввести пароль, а затем выполнять команды.
Проверка доступа carol (не в группе developers)
$ su - carol -c 'ls /srv/project/code'
ls: cannot open directory '/srv/project/code': Permission denied
carol не может войти в папку, потому что права 770 дают остальным (other) --- (никаких прав), а carol не входит в группу developers.
Проблема с файлом /srv/project/reports/q1.txt
sudo -u alice touch /srv/project/reports/q1.txt
команда не сработала, потому что у пользователя alice нет прав на запись в папку /srv/project/reports (владелец папки — root, группа — root).
Что нужно было сделать: сначала дать alice права на запись в папку reports:
sudo chown alice:developers /srv/project/reports
sudo chmod 770 /srv/project/reports

##3. ACL
Дать carol доступ на чтение к /srv/project/code, не добавляя её в группу developers и не давая прав всем остальным.
Текущие ACL до добавления
$ getfacl /srv/project/code
# file: srv/project/code
# owner: alice
# group: developers
user::rwx
group::r-x
other::---

Добавление ACL для carol
sudo setfacl -m u:carol:r-x /srv/project/code
Результат
$ getfacl /srv/project/code
# file: srv/project/code
# owner: alice
# group: developers
user::rwx
user:carol:r-x
group::r-x
mask::r-x
other::---
Наличие ACL видно по символу + в выводе ls -la:
text
drwxr-x---+ 2 alice developers 4096 Jun 19 17:03 code
https://screens/7_getfacl.png
Проверка доступа carol
$ su - carol -c 'ls /srv/project/code'
Password:
(пустой вывод — carol видит файлы, но в папке пусто)
$ su - carol -c 'touch /srv/project/code/carol_hack.py'
touch: cannot touch '/srv/project/code/carol_hack.py': Permission denied
carol видит файлы в папке code (право r-x по ACL).
carol не может создавать файлы (права на запись нет).
Стандартная модель даёт права только трём категориям: владелец, группа, остальные. ACL позволяет дать права конкретному пользователю (carol) без изменения её группы и без предоставления прав всем остальным. В данной ситуации это идеальное решение — carol получает доступ, а группа developers и остальные пользователи не меняются.
Удаление ACL
sudo setfacl -x u:carol /srv/project/code
После удаления ACL carol снова не может зайти в папку:
$ su - carol -c 'ls /srv/project/code'
ls: cannot open directory '/srv/project/code': Permission denied

##4. sudo-политики
Настройки в /etc/sudoers
# alice — полный sudo без пароля (администратор)
alice ALL=(ALL:ALL) NOPASSWD:ALL

# bob — только apt (установка пакетов)
bob ALL=(ALL) NOPASSWD: /usr/bin/apt, /usr/bin/apt-get

# carol — только просмотр логов
carol ALL=(ALL) NOPASSWD: /usr/bin/journalctl, /bin/cat /var/log/*
Проверка политик
$ sudo -l -U alice
User alice may run the following commands on this host:
    (ALL : ALL) ALL

$ sudo -l -U bob
User bob may run the following commands on this host:
    (ALL) NOPASSWD: /usr/bin/apt, /usr/bin/apt-get

$ sudo -l -U carol
User carol may run the following commands on this host:
    (ALL) NOPASSWD: /usr/bin/journalctl, /bin/cat /var/log/*
alice — администратор, она должна иметь возможность быстро выполнять команды без ввода пароля. bob — разработчик, ему нужен доступ только к установке пакетов, поэтому его права ограничены. В задании у bob указано NOPASSWD для конкретных команд (apt и apt-get).
Реализован принцип наименьших привилегий: каждый пользователь получает ровно те права, которые необходимы для его работы.
https://screens/10_sudo_alice.png
https://screens/11_sudo_bob.png
https://screens/12_sudo_carol.png

##5. PAM
Модуль pam_unix.so
В /etc/pam.d/sudo и /etc/pam.d/login присутствует строка:
auth required pam_unix.so
required означает, что модуль должен успешно выполниться. Если он возвращает ошибку, аутентификация отклоняется, но остальные модули в стеке всё равно выполняются (для логирования). Если required не проходит, пользователь не получает доступ.
Политика паролей (/etc/pam.d/common-password)
password requisite pam_pwquality.so retry=3 minlen=8 difok=3
password [success=1 default=ignore] pam_unix.so obscure use_authtok try_first_pass sha512
Уже настроены:
minlen=8 — минимальная длина пароля 8 символов.
difok=3 — минимум 3 символа должны отличаться от старого пароля.
retry=3 — 3 попытки ввода.
obscure — проверка на простые/словарные пароли.
sha512 — хэширование пароля алгоритмом SHA-512.
Как добавить минимальную длину 12 символов:
В файле /etc/pam.d/common-password изменить параметр minlen:
password requisite pam_pwquality.so retry=3 minlen=12 difok=3
https://screens/14_pam_sudo.png

Выводы
В ходе практической работы я созда а пользователей и группы, изучила структуру /etc/passwd и /etc/shadow. Настроила права доступа через chmod и chown, проверила доступ для разных пользователей. Применила ACL для предоставления доступа конкретному пользователю без изменения группы. Настроила политики sudo для трёх ролей с разным уровнем привилегий. Изучила конфигурацию PAM и требования к паролям.

Контрольные вопросы
44. Что означает chmod 640 и кто читает?
6 — владелец: чтение+запись, 4 — группа: только чтение, 0 — остальные: ничего. Читают: владелец и группа.
45. Чем setuid отличается от sudo?
setuid — запускает файл с правами владельца (привязан к файлу, например, /usr/bin/passwd).
sudo — даёт права конкретному пользователю на конкретные команды (привязан к пользователю, настраивается в /etc/sudoers).
46. Почему хэши в /etc/shadow, а не в /etc/passwd?
/etc/passwd читают все. Если там хранить хэши, злоумышленник может их украсть и перебрать пароли. /etc/shadow доступен только root — это безопаснее.
47. Что будет, если дать bob sudo bash?
Он получит root-оболочку и сможет делать всё, что угодно, — ограничения становятся бесполезны. Это нарушает принцип наименьших привилегий и очень опасно.
48. Разница между su и sudo:
su — переключает на другого пользователя (нужен его пароль), даёт полный доступ, не логирует.
sudo — выполняет одну команду от root (нужен свой пароль), логирует всё, можно ограничивать команды. Безопаснее — sudo.
49. Как запретить пользователю sudo без удаления из группы?
Добавить в /etc/sudoers строку:
bob ALL=(ALL) !ALL
(запрет должен стоять после разрешающих правил, так как последнее правило имеет приоритет).
