### Как сдавать задания

Вы уже изучили блок «Системы управления версиями», и начиная с этого занятия все ваши работы будут приниматься ссылками на .md-файлы, размещённые в вашем публичном репозитории.

Скопируйте в свой .md-файл содержимое этого файла; исходники можно посмотреть [здесь](https://raw.githubusercontent.com/netology-code/sysadm-homeworks/devsys10/04-script-02-py/README.md). Заполните недостающие части документа решением задач (заменяйте `???`, ОСТАЛЬНОЕ В ШАБЛОНЕ НЕ ТРОГАЙТЕ чтобы не сломать форматирование текста, подсветку синтаксиса и прочее, иначе можно отправиться на доработку) и отправляйте на проверку. Вместо логов можно вставить скриншоты по желани.

# Домашнее задание к занятию "4.2. Использование Python для решения типовых DevOps задач"

## Обязательная задача 1

Есть скрипт:
```python
#!/usr/bin/env python3
a = 1
b = '2'
c = a + b
```

### Вопросы:
| Вопрос  | Ответ |
| ------------- | ------------- |
| Какое значение будет присвоено переменной `c`?  | `>>> TypeError: unsupported operand type(s) for +: 'int' and 'str'`  |
| Как получить для переменной `c` значение 12?  | `c = str(a) + b`  |
| Как получить для переменной `c` значение 3?  | `c = a + int(b)`  |

## Обязательная задача 2
Мы устроились на работу в компанию, где раньше уже был DevOps Engineer. Он написал скрипт, позволяющий узнать, какие файлы модифицированы в репозитории, относительно локальных изменений. Этим скриптом недовольно начальство, потому что в его выводе есть не все изменённые файлы, а также непонятен полный путь к директории, где они находятся. Как можно доработать скрипт ниже, чтобы он исполнял требования вашего руководителя?

```python
#!/usr/bin/env python3

import os

bash_command = ["cd ~/netology/sysadm-homeworks", "git status"]
result_os = os.popen(' && '.join(bash_command)).read()
is_change = False
for result in result_os.split('\n'):
    if result.find('modified') != -1:
        prepare_result = result.replace('\tmodified:   ', '')
        print(prepare_result)
        break
```

### Ваш скрипт:
```python
#!/usr/bin/env python3

import os

SOURCE_PATH = "~/netology/sysadm-homeworks"

repository_path = os.path.abspath(os.path.expanduser(SOURCE_PATH))
bash_command = ["cd " + repository_path, "git status"]
result_os = os.popen(' && '.join(bash_command)).read()

for result in result_os.split('\n'):
    if result.find('modified') != -1:
        prepare_result = repository_path + result.replace('\tmodified:   ', os.sep)
        print(prepare_result)
```

### Вывод скрипта при запуске при тестировании:
```
:~$ ./changes.sh

>>> /home/vagrant/netology/sysadm-homeworks/03-sysadmin/04-os/README.md
>>> /home/vagrant/netology/sysadm-homeworks/03-sysadmin/05-fs/README.md
>>> /home/vagrant/netology/sysadm-homeworks/README.md
```

## Обязательная задача 3
1. Доработать скрипт выше так, чтобы он мог проверять не только локальный репозиторий в текущей директории, а также умел воспринимать путь к репозиторию, который мы передаём как входной параметр. Мы точно знаем, что начальство коварное и будет проверять работу этого скрипта в директориях, которые не являются локальными репозиториями.

### Ваш скрипт:
```python
#!/usr/bin/env python3

import os
import sys

if 2 > len(sys.argv):
    print("Invalid arguments. Path not specified.")
    exit(1)

SOURCE_PATH = sys.argv[1]

repository_path = os.path.abspath(os.path.expanduser(SOURCE_PATH))
bash_command = ["cd " + repository_path, "git status"]
result_os = os.popen(' && '.join(bash_command)).read()

for result in result_os.split('\n'):
    if result.find('modified') != -1:
        prepare_result = repository_path + result.replace('\tmodified:   ', os.sep)
        print(prepare_result)

exit(0)
```

### Вывод скрипта при запуске при тестировании:
```
:~$ ./changes.sh ~/netology/sysadm-homeworks

>>> /home/vagrant/netology/sysadm-homeworks/03-sysadmin/04-os/README.md
>>> /home/vagrant/netology/sysadm-homeworks/03-sysadmin/05-fs/README.md
>>> /home/vagrant/netology/sysadm-homeworks/README.md
```

## Обязательная задача 4
1. Наша команда разрабатывает несколько веб-сервисов, доступных по http. Мы точно знаем, что на их стенде нет никакой балансировки, кластеризации, за DNS прячется конкретный IP сервера, где установлен сервис. Проблема в том, что отдел, занимающийся нашей инфраструктурой очень часто меняет нам сервера, поэтому IP меняются примерно раз в неделю, при этом сервисы сохраняют за собой DNS имена. Это бы совсем никого не беспокоило, если бы несколько раз сервера не уезжали в такой сегмент сети нашей компании, который недоступен для разработчиков. Мы хотим написать скрипт, который опрашивает веб-сервисы, получает их IP, выводит информацию в стандартный вывод в виде: <URL сервиса> - <его IP>. Также, должна быть реализована возможность проверки текущего IP сервиса c его IP из предыдущей проверки. Если проверка будет провалена - оповестить об этом в стандартный вывод сообщением: [ERROR] <URL сервиса> IP mismatch: <старый IP> <Новый IP>. Будем считать, что наша разработка реализовала сервисы: `drive.google.com`, `mail.google.com`, `google.com`.

### Ваш скрипт:
```python
#!/usr/bin/env python3

import socket

HOSTS_LIST_FILE = "services.txt"
DEF_HOSTS_LIST = (["drive.google.com", "unknown"], ["mail.google.com", "unknown"], ["google.com", "unknown"])


def read_hosts_list():
    try:
        with open(HOSTS_LIST_FILE, "r") as file:
            return tuple((line.strip().split(" ")) for line in file.readlines())
    except IOError:
        return DEF_HOSTS_LIST


def write_hosts_list(hosts):
    with open(HOSTS_LIST_FILE, "w+") as file:
        file.write("\n".join(" ".join(item) for item in hosts))


def get_ip(hostname):
    return socket.gethostbyname(hostname)


hosts_list = read_hosts_list()
result = []
for host in hosts_list:
    current_ip = get_ip(host[0])
    if current_ip != host[1]:
        print(f"[ERROR] {host[0]} IP mismatch: {host[1]} {current_ip}")
    print(f"{host[0]} - {current_ip}")
    result.append([host[0], current_ip])
write_hosts_list(result)
```

### Вывод скрипта при запуске при тестировании:
```bash
>>> [ERROR] drive.google.com IP mismatch: unknown 64.233.165.194
>>> drive.google.com - 64.233.165.194
>>> [ERROR] mail.google.com IP mismatch: unknown 142.251.1.19
>>> mail.google.com - 142.251.1.19
>>> [ERROR] google.com IP mismatch: unknown 64.233.161.139
>>> google.com - 64.233.161.139
```
