{% raw %}
## Модули для работы с сетевым оборудованием

В предыдущих разделах для отправки команд на оборудование использовался модуль raw.
Он универсален, и с его помощью можно отправлять команды на любое устройство.

В этом разделе рассматриваются модули, которые работают с сетевым оборудованием.

Глобально модули для работы с сетевым оборудованием можно разделить на две части:
* модули для оборудования с поддержкой API
* модули для оборудования, которое работает только через CLI

Если оборудование поддерживает API, как, например, [NXOS](http://docs.ansible.com/ansible/devel/list_of_network_modules.html#nxos), то для него создано большое количество модулей, которые выполняют конкретные действия по настройке функционала (например, для NXOS создано более 60 модулей).

Для оборудования, которое работает только через CLI, Ansible поддерживает, как минимум, такие три типа модулей:
* os_command - выполняет команды show
* os_facts - собирает факты об устройствах
* os_config - выполняет команды конфигурации

Соответственно, для разных операционных систем будут разные модули. Например, для Cisco IOS модули будут называться:
* ios_command
* ios_config
* ios_facts

Аналогичные три модуля доступны для таких ОС:
* Dellos10
* Dellos6
* Dellos9
* EOS
* IOS
* IOS XR
* JUNOS
* SR OS
* VyOS

Полный список всех сетевых модулей, которые поддерживает Ansible, в [документации](http://docs.ansible.com/ansible/devel/list_of_network_modules.html).

Обратите внимание, что Ansible очень активно развивается в сторону поддержки работы с сетевым оборудованием, и в следующей версии Ansible, могут быть дополнительные модули.
Поэтому, если на момент чтения курса уже есть следующая версия Ansible (версия в курсе 2.4), используйте её и посмотрите в документации, какие новые возможности и модули появились.


В этом разделе все рассматривается на примере модулей для работы с Cisco IOS:
* ios_command
* ios_config
* ios_facts

> Аналогичные модули command, config и facts для других вендоров и ОС работают одинаково, поэтому, если разобраться, как работать с модулями для IOS, с остальными всё будет аналогично.

Кроме того, рассматривается модуль ntc-ansible, который не входит в core модули Ansible.

### Варианты подключения

Ansible поддерживает такие типы подключений:
* __paramiko__
* __SSH__ - OpenSSH. Используется по умолчанию
* __local__ - действия выполняются локально, на управляющем хосте


> При подключении по SSH по умолчанию используются SSH ключи, но можно переключиться на использование паролей.


По умолчанию Ansible загружает модуль Python на устройство, для того, чтобы выполнить действия.
Если же оборудование не поддерживает Python, как в случае с доступом к сетевому оборудованию через CLI, нужно указать, что модуль должен запускаться локально, на управляющем хосте Ansible.


### Особенности подключения к сетевому оборудованию

При  работе с сетевым оборудованием есть несколько параметров в playbook, которые нужно менять:
* gather_facts - надо отключить, так как для сетевого оборудования используются свои модули сбора фактов
* connection - управляет тем, как именно будет происходить подключение. Для сетевого оборудования необходимо установить в local


То есть, для каждого сценария (play), нужно указывать:
* gather_facts: false
* connection: local

Пример:
```
---

- name: Run show commands on routers
  hosts: cisco-routers
  gather_facts: false
  connection: local

```

В Ansible переменные можно указывать в разных местах, поэтому те же настройки можно указать по-другому.

Например, в разделе о [конфигурационном файле](../1_ansible_basics/configuration.md) рассматривалось, как отключить сбор фактов по умолчанию (файл ansible.cfg):
```
[defaults]

gathering = explicit
```

Такой вариант подходит в том случае, когда Ansible используется больше для подключения к сетевым устройствам (или локальные playbook используются для подключения к сетевому оборудованию).

В таком случае нужно будет наоборот явно включать сбор фактов, если он нужен.

Указать, что нужно использовать локальное подключение, также можно по-разному.

В инвентарном файле:
```
[cisco-routers]
192.168.100.1
192.168.100.2
192.168.100.3

[cisco-switches]
192.168.100.100

[cisco-routers:vars]
ansible_connection=local
```

Или в файлах переменных, например, в group_vars/all.yml:
```
---

ansible_connection: local
```

В следующих разделах будет использоваться такой вариант:
```
---

- name: Run show commands on routers
  hosts: cisco-routers
  gather_facts: false
  connection: local
```

В реальной жизни нужно выбрать тот вариант, который наиболее удобен для работы.


### Аргумент provider

Модули, которые используются для работы с сетевым оборудованием, требуют задания нескольких аргументов.

Для каждой задачи должны быть указаны такие аргументы:
* __host__ - имя или IP-адрес удаленного устройства
* __port__ - к какому порту подключаться
* __username__ - имя пользователя
* __password__ - пароль
* __transport__ - тип подключения: CLI или API. По умолчанию - cli
* __authorize__ - нужно ли переходить в привилегированный режим (enable, для Cisco)
* __auth_pass__ - пароль для привилегированного режима

> Если для подключения Ansible создан отдельный пользователь с privilege 15, можно не использовать параметры authorize и auth_pass.

Но Ansible также позволяет собрать их в один аргумент - __provider__.


Пример задания всех аргументов в задаче (task):
```
  tasks:

    - name: run show version
      ios_command:
        commands: show version
        host: "{{ inventory_hostname }}"
        username: cisco
        password: cisco
        transport: cli
```

Аргументы созданы как переменная ```cli``` в playbook, а затем передаются как переменная аргументу provider:
```
  vars:
    cli:
      host: "{{ inventory_hostname }}"
      username: cisco
      password: cisco
      transport: cli

  tasks:
    - name: run show version
      ios_command:
        commands: show version
        provider: "{{ cli }}"

```

И, самый удобный вариант - задавать аргументы в каталоге group_vars.

Например, если у всех устройств одинаковые значения аргументов, можно задать их в файле group_vars/all.yml:
```
---

cli:
  host: "{{ inventory_hostname }}"
  username: cisco
  password: cisco
  transport: cli
  authorize: yes
  auth_pass: cisco
```

Затем переменная используется в playbook так же, как и в случае указания переменных в playbook:
```
  tasks:
    - name: run show version
      ios_command:
        commands: show version
        provider: "{{ cli }}"
```

Кроме того, Ansible поддерживает задание параметров в переменных окружения:
* ANSIBLE_NET_USERNAME - для переменной username
* ANSIBLE_NET_PASSWORD - password
* ANSIBLE_NET_SSH_KEYFILE - ssh_keyfile
* ANSIBLE_NET_AUTHORIZE - authorize
* ANSIBLE_NET_AUTH_PASS - auth_pass


Значения в порядке возрастания приоритета:
* значения по умолчанию
* значения переменных окружения
* параметр provider
* аргументы задачи (task)

### Подготовка к работе с сетевыми модулями

В следующих разделах рассматривается работа с модулями ios_command, ios_facts и ios_config.
Для того, чтобы все примеры playbook работали, надо создать несколько файлов (проверить, что они есть).

Инвентарный файл myhosts:
```
[cisco-routers]
192.168.100.1
192.168.100.2
192.168.100.3

[cisco-switches]
192.168.100.100
```

Конфигурационный файл ansible.cfg:
```
[defaults]

inventory = ./myhosts

remote_user = cisco
ask_pass = True
```

В файле group_vars/all.yml надо создать переменную cli, чтобы не указывать каждый раз все параметры, которые нужно передать аргументу provider:
```
---

cli:
  host: "{{ inventory_hostname }}"
  username: "cisco"
  password: "cisco"
  transport: cli
  authorize: yes
  auth_pass: "cisco"
```


{% endraw %}