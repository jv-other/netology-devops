### Как сдавать задания

Вы уже изучили блок «Системы управления версиями», и начиная с этого занятия все ваши работы будут приниматься ссылками на .md-файлы, размещённые в вашем публичном репозитории.

Скопируйте в свой .md-файл содержимое этого файла; исходники можно посмотреть [здесь](https://raw.githubusercontent.com/netology-code/sysadm-homeworks/devsys10/04-script-03-yaml/README.md). Заполните недостающие части документа решением задач (заменяйте `???`, ОСТАЛЬНОЕ В ШАБЛОНЕ НЕ ТРОГАЙТЕ чтобы не сломать форматирование текста, подсветку синтаксиса и прочее, иначе можно отправиться на доработку) и отправляйте на проверку. Вместо логов можно вставить скриншоты по желани.

# Домашнее задание к занятию "4.3. Языки разметки JSON и YAML"


## Обязательная задача 1
Мы выгрузили JSON, который получили через API запрос к нашему сервису:
```
    { "info" : "Sample JSON output from our service\t",
        "elements" :[
            { "name" : "first",
            "type" : "server",
            "ip" : 7175 
            }
            { "name" : "second",
            "type" : "proxy",
            "ip : 71.78.22.43
            }
        ]
    }
```
  Нужно найти и исправить все ошибки, которые допускает наш сервис

Решение:

```json
{
  "info": "Sample JSON output from our service\t",
  "elements": [
    {
      "name": "first",
      "type": "server",
      "ip": 7175
    },
    {
      "name": "second",
      "type": "proxy",
      "ip": "71.78.22.43"
    }
  ]
}
```

Ошибки (поле elements):
- разный тип поля "ip", для первого элемента десятичное представление, для второго используется десятичное с точками
- пропущена запятая между элементами
- у второго элемента пропущена закрывающая кавычка для названия поля ip
- отсутствуют кавычки в значении поля ip

## Обязательная задача 2
В прошлый рабочий день мы создавали скрипт, позволяющий опрашивать веб-сервисы и получать их IP. К уже реализованному функционалу нам нужно добавить возможность записи JSON и YAML файлов, описывающих наши сервисы. Формат записи JSON по одному сервису: `{ "имя сервиса" : "его IP"}`. Формат записи YAML по одному сервису: `- имя сервиса: его IP`. Если в момент исполнения скрипта меняется IP у сервиса - он должен так же поменяться в yml и json файле.

### Ваш скрипт:
```python
#!/usr/bin/env python3

import socket
import json
import yaml

HOSTS_LIST_FILE = "services"
DEF_HOSTS_LIST = ({"drive.google.com": "unknown"}, {"mail.google.com": "unknown"}, {"google.com": "unknown"})


def read_hosts_list(file_format):
    try:
        with open(f"{HOSTS_LIST_FILE}.{file_format}", "r") as file:
            return yaml.safe_load(file)
    except IOError:
        return DEF_HOSTS_LIST


def write_hosts_list(hosts, file_format):
    with open(f"{HOSTS_LIST_FILE}.{file_format}", "w+") as file:
        file.write(json.dumps(hosts, indent=2) if "json" == file_format else yaml.dump(hosts))


def get_ip(hostname):
    return socket.gethostbyname(hostname)


def update_services(file_format):
    hosts_list = read_hosts_list(file_format)
    result = []
    for service in hosts_list:
        service_name = list(service.keys())[0]
        service_ip = service[service_name]
        current_ip = get_ip(service_name)
        if current_ip != service_ip:
            print(f"[ERROR] {service_name} IP mismatch: {service_ip} {current_ip}")
        print(f"{service_name} - {current_ip}")
        result.append(dict([(service_name, current_ip)]))
    write_hosts_list(result, file_format)


# json
print("json file:")
update_services("json")

# yaml
print("\nyaml file:")
update_services("yaml")
```

### Вывод скрипта при запуске при тестировании:
```
>>> json file:
>>> drive.google.com - 64.233.162.194
>>> mail.google.com - 142.250.150.19
>>> google.com - 74.125.205.139

>>> yaml file:
>>> drive.google.com - 64.233.162.194
>>> mail.google.com - 142.250.150.19
>>> google.com - 74.125.205.139
```

### json-файл(ы), который(е) записал ваш скрипт:
```json
[
  {
    "drive.google.com": "64.233.162.194"
  },
  {
    "mail.google.com": "142.250.150.19"
  },
  {
    "google.com": "74.125.205.139"
  }
]
```

### yml-файл(ы), который(е) записал ваш скрипт:
```yaml
- drive.google.com: 64.233.162.194
- mail.google.com: 142.250.150.19
- google.com: 74.125.205.139
```
