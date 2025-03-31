# Тестовое задание
Есть 2 сервера: A и Б (hostA и hostB соответственно). 
Ubuntu Server 20.04 LTS без предустановленных пакетов и программ. 
Необходимо предложить автоматизированное решение, которое позволит:
1. Отключить авторизацию SSH по паролю. На обоих серверах.
2. Добавить пользователя DevOps, настроить авторизацию по ключам и предоставить
sudo без пароля. На обоих серверах.
3. Настроить fail2ban на сервере А с блокировкой на 1 час, если в течение 1 минуты
было 3 неудачных попыток входа.
4. На сервер A установить PostgreSQL, создать БД app и custom:
a) добавить пользователя app с паролем app, у которого будет полный доступ
к БД app;
b) добавить пользователя custom с паролем custom, у которого будет полный
доступ к БД custom;
c) добавить пользователя service с паролем service, у которого будет доступ на
чтение к обеим базам.
5. На сервер Б установить nginx и настроить его так, чтобы при обращении на
сервере к localhost (или его доменному имени) открывался сайт https://ya.ru
6. Настроить доступ к PostgreSQL на сервере А только с сервера Б, закрыть доступ к
nginx на сервере Б с сервера А.
7. Настроить бэкапирование PostgreSQL с сервера А на сервер Б

## Подготовка окружения перед запуском проекта
Для запуска проекта необходимо сгенерировать ssh ключи для доступа к удалённым хостам:

```sh
ssh-keygen -t rsa
```
Просто несколько раз прожимаем Enter. Ключи сохранятся по стандартному пути.

Дадим ключам права RW чтобы в будущем ssh на них не ругался.

```sh
sudo chmod 600 /home/user/.ssh/id_rsa
sudo chmod 600 /home/user/.ssh/id_rsa.pub
```
Затем нужно поделиться публичным ключом с серверами, они же должны знать, что к ним знакомый хост стучится, а не не понятно кто :)

```sh
ssh-copy-id DevOps@hostA
ssh-copy-id DevOps@hostB
```
## Запуск проекта
Тестовое задание было решено выполнить при помощи Ansible

```sh
ansible-playbook -i ./inv/inventory ansible-playbook.yml
```
Будет запущен плейбук, который пошагово будет выполнять таски, которые там прописаны

Разберем, что делают таски в этом плейбуке

Данная таска конфигурирует общие настройки для обоих серверов, такие как создание пользователя DevOps, sudo без пароля, ssh и так далее.

При первом запуске плейбук сразу стриггерится notify, поскольку это первый запуск, соответственно все хендлерсы будут обработаны. Следующие разы notify будет обработан только в случае изменений в тасках.

```yml
---
---
- name: Конфигурация hostA и hostB
  hosts: all
  become: yes
  vars:
    user: DevOps
    ssh_key: "{{ lookup('file', '/home/user/.ssh/id_rsa.pub') }}"
  tasks:
    - name: Отключение авторизации SSH по паролю
      lineinfile:
        path: /etc/ssh/sshd_config
        regexp: '^#?PasswordAuthentication'
        line: 'PasswordAuthentication no'
        state: present
      notify:
        - Restart SSH

    - name: Создание юзера DevOps
      user:
        name: "{{ user }}" # берем из переменной
        state: present
        groups: sudo
        append: yes
        shell: /bin/bash

    - name: Определение SSH ключа юзеру DevOps
      authorized_key:
        user: "{{ user }}" 
        key: "{{ ssh_key }}"
        state: present

    - name: Настройка входа по sudo без пароля
      lineinfile:
        path: /etc/sudoers
        state: present
        line: '{{ user }} ALL=(ALL) NOPASSWD:ALL'
        validate: 'visudo -cf %s'
  handlers:
  - name: Restart SSH
    service:
      name: sshd
      state: restarted
```


В данном случае таска настраивает только сервер hostA. Будет настроен fail2ban, конфигурационный файл которого лежит в директории ./var.

```yml
- name: Конфигурация hostA
  hosts: hostA
  become: yes
  tasks:

    - name: Установка fail2ban
      apt:
        name: fail2ban
        state: present

    - name: Конфигурация f2b
      template:
        src: vars/fail2ban
        dest: /etc/fail2ban/jail.d/f2b 
```

Так же будет установлена СУБД postgres. После установки СУБД будет сразу изменен файл postgresql.conf, путем добавления строчки listen_addresses='*', которая дает возможность подключаться к ней с любого хоста. 
Это будет необходимо для последующих действий, таких как создание непосредственно самих баз, пользователей и выдачу им необходимых прав, согласно заданию.

  
```yml
    - name: Установка psycopr на hostA
      apt:
        name: python3-psycopg2
        state: present

    - name: Установка postgres на hostA
      apt:
        name: postgresql
        state: present
        update_cache: yes

    - name: Настройка доступа к базе
      lineinfile:
        path: /etc/postgresql/12/main/postgresql.conf
        line: listen_addresses='*'
        state: present
      notify: Restart postgres

    - name: Создание базы app
      become: yes
      become_user: user
      postgresql_db:
        state: present
        name: app

    - name: Создание базы custom
      become: yes
      become_user: user
      postgresql_db:
        state: present
        name: custom

    - name: Создание юзера app - полный доступ к базе app
      postgresql_user:
        name: app
        password: app
        db: app
        priv: ALL
        state: present

    - name: Создание юзера custom - полный доступ к базе custom
      postgresql_user:
        name: custom
        password: custom
        db: custom
        priv: ALL
        state: present

    - name: Создание юзера service
      postgresql_user:
        name: service
        password: service
        state: present

    - name: Доступ на чтение к app (SELECT) 
      postgresql_privs:
        database: app
        role: service
        privs: SELECT
        type: table
        obj: ALL_IN_SCHEMA
        schema: public

    - name: Доступ на чтение к custom базам (SELECT) 
      postgresql_privs:
        database: custom
        role: service
        privs: SELECT
        type: table
        obj: ALL_IN_SCHEMA
        schema: public
``` 

Далее будет настроен сетевой доступ к только что созданным базам. Для этого был отредактирован файл pg_hba.conf, был прописан парольный доступ к базам (md5). То есть СУБД будет доступна только с хоста hostB, однако и локально с hostA база так же будет доступна.

```yml
    - name: Настройка доступа к базе только c hostB
      lineinfile:
        path: /etc/postgresql/12/main/pg_hba.conf
        line: "host all all hostB md5"
        state: present
        create: yes
      notify: Restart postgres
```

В этом пункте к сожалению возникли трудности, поскольку никак не получалось настроить ssh между hostA и hostB для передачи созданных резервных копий.
Скрипт бекапа был в описан в ./var/pg-backup. Так же был описан планировщик cron. Rsync был использовал по причине его большей производительности и меньших времязатратах, поскольку происходит синхронизация только тех данных, которые были изменены, а не БД целиком. 

```yml 
    - name: Установка rsync
      apt:
        name: rsync
        state: present

    - name: Копирование скрипта бекапа
      template:
        src: vars/pg-backup
        dest: /usr/local/bin/pg_backup.sh

    - name: Задание расписание бекапов
      cron:
        name: "PostgreSQL backup"
        minute: "0"
        hour: "2"
        job: "/usr/local/bin/pg_backup.sh"
```

По аналогии с предыдущей таской здесь так же описаны хендлерсы.

```yml

  handlers:
  - name: Restart fail2ban
    service:
      name: fail2ban
      state: reloaded

  - name: Restart postgres
    service:
      name: postgresql
      state: restarted 
      enabled: yes  
```

Для настройки hostB была описана следующая таска. Здесь потребовалось сконфигурировать веб-сервер таким образом, что при вызове localhost будет открываться сайт renue.ru. 

```yml
- name: Конфигурация hostB
  hosts: hostB
  become: yes
  tasks:
    - name: Установка nginx на hostB
      apt:
        name: nginx
        state: present  
      notify: Restart nginx

    - name: Копирование шаблона nginx
      template:
        src: vars/default
        dest: /etc/nginx/sites-available/default
      notify: Restart nginx 

    - name: Создание линка в sites-enabled 
      file:
        src: /etc/nginx/sites-available/default
        dest: /etc/nginx/sites-enabled/default
        state: link
      notify: Restart nginx
```
В принципе можно проверить при помощи:
```sh
curl localhost
```

Команда вернет html разметку сайта ya.ru

В следующей задаче будет настроен запрет к веб-серверу со стороны hostA. DNS имя не прокатит, так что придется указать адрес сервера явно. 


```yml
    - name: Запрет обращения к nginx c hostA
      ufw:
        rule: deny
        src: 192.168.3.50 #нужно явно прописать ip-хоста
        port: 80,443
        proto: tcp 
      notify: Restart nginx
```
Ну и хендлерсы.

```yml
  handlers:
  - name: Restart nginx
    service:
      name: nginx
      state: reloaded
```


