# Настройка сервера и развертывание Django проекта

Маленькая шпаргалка с одним из вариантов, как можно развернуть свой домашний проект. 
(Обычно, такая работа выполняется программистом не так часто, потому создал этот файл, что бы не потерять)

На старте мы имеем виртуальную машину с установленной серверной операционной системой.
Нижеприведенный текст описывает работу с Ubuntu 20.04.1 LTS 


## Настройка сервера

### Создаем не рутованного пользователя

Если ранее не был создан пользователь в системе, то необходимо его создать.

0. Создать пользователя: `useradd [options] user_name`
	> ```sudo useradd {user_name}```
0. Задать пароль пользователя: `passwd user_name`
	> ```sudo passwd {user_name}```
0. Добавить пользователя в группу с правами привилегированных команд: `usermod -aG sudo user_name`
	> ```usermod -aG sudo {user_name}```

Далее работаем только с созданного нами пользователя. Лучше вообще запретить вход с других пользователей.
Незабываем, при создании имени пользователя и пароля придерживаться уникальности (для имени и пароля) и сложности (для пароля).

### Настройка аутентификации по SSH

Для работы нам потребуется openSSH. Если openSSH уже установлен в системе, то устанавливать не нужно. Для проверки, установлен ли он в системе можно воспользоваться командой `dpkg-query -l` которая выведет список всех установленных в системе программ.

0. Устанавливаем OpenSSH: `sudo apt install openssh-server`
0. Запуск процесса (демон) на сервере: `sudo systemctl start ssh`
	> [Как в Linux пользоваться командой systemctl](https://www.dmosk.ru/miniinstruktions.php?mini=systemctl)
0. Добавляем в автозапуск: `sudo systemctl enable ssh`
0. На клиенте, с которого планируется вход на сервер, генерируем новые ключи: `sudo ssh-keygen -t rsa`
0. Создаем файл `authorized_keys` в каталоге `/home/{user_name}/.ssh/`. В указанный файл копируем наш публичный ключ
	> `mkdir -p /home/{user_name}/.ssh`
	> `touch /home/{user_name}/.ssh/authorized_keys`
	> `vim /home/{user_name}/.ssh/authorized_keys`
0. Дополнительно можно настроить права доступа к файлу и каталогу
	* изменяем владельца файла (-R - рекурсивно): `chown -R user_name[:user_group] <path_to_folder>` 
	* читать, писать и исполнять только для владельца: `chmod 700 {path_to_folder}`
	* читать и писать может только владелец: `chmod 600 {path_to_file}`
	> `chown -R {user_name} /home/{user_name}/.ssh`
	> `chmod 700 /home/{user_name}/.ssh`
	> `chmod 600 /home/{user_name}/.ssh/authorized_keys`
0. Перезапускаем shh: `sudo systemctl restart ssh`


Для запрета аутентификации для рута и запрета аутентификации через пароль нам необходимо внести изменения в файл /etc/ssh/sshd_config.

0. Отключить аутентификацию под суперпользователем
	> `PermitRootLogin no`
0. Предоставляем доступ к подключению только для пользователя `AllowUsers` или группы пользователей `AllowGroups`
	> `AllowUsers {user_name}`

0. Дополнительно: 
	- Изменить порт: `Port`
	- Изменение времени ожидания авторизации: `LoginGraceTime`
	- Ограничение авторизации по интерфейсу `ListenAddress`

>[Практические советы, примеры и туннели SSH](https://hackertarget.com/ssh-examples-tunnels/)

### Запуск файрвола
 
 Тут пока пусто
 
 TODO Дополнить материал

> Дополнительно можно включить файрвол `ufw`



## Установка и настройка PostgreSQL

0. Скачиваем PostgreSQL
    ```
    wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - ; \
    RELEASE=$(lsb_release -cs) ; \
    echo "deb http://apt.postgresql.org/pub/repos/apt/ ${RELEASE}"-pgdg main | sudo tee  /etc/apt/sources.list.d/pgdg.list ; \
    sudo apt update ; \
    sudo apt -y install postgresql-13 ; \
    ```
0. Настраиваем русский язык (Если ранее не выбрали)
    ```
    sudo localedef ru_RU.UTF-8 -i ru_RU -fUTF-8 ; \
    export LANGUAGE=ru_RU.UTF-8 ; \
    export LANG=ru_RU.UTF-8 ; \
    export LC_ALL=ru_RU.UTF-8 ; \
    sudo locale-gen ru_RU.UTF-8 ; \
    sudo dpkg-reconfigure locales
    ```
    ```
    sudo vim /etc/profile
        export LANGUAGE=ru_RU.UTF-8
        export LANG=ru_RU.UTF-8
        export LC_ALL=ru_RU.UTF-8
    ```

0. Меняем пароль для пользователя postgres
	> `sudo passwd postgres`

0. Входим пользователем postgres в postgres и создаем базу данных с именем {db_name}
    ```
    su - postgres
    export PATH=$PATH:/usr/lib/postgresql/13/bin
    createdb --encoding UNICODE {db_name} --username postgres
    exit
    ```

0. Создаем пользователя базы данных и даем ему привилегии

	- Заходим под пользователем postgres: `sudo -u postgres psql`
	- Создаем нового пользователя: `create user {user_name} with password '{user_pass}';`
	- Разрешаем пользователю создавать БД: `ALTER USER {user_name} CREATEDB;`
	- Назначаем привилегии на БД: `GRANT ALL PRIVILEGES ON DATABASE {database_name} TO {user_name};`
	- Переходим в созданную БД: `\c {database_name}`
	- Назначаем привилегии на БД:
		```GRANT ALL ON ALL TABLES IN SCHEMA public to {user_name};
		GRANT ALL ON ALL SEQUENCES IN SCHEMA public to {user_name};
		GRANT ALL ON ALL FUNCTIONS IN SCHEMA public to {user_name};```
	- `\q`
	- `exit`


## Установка интерпретатора, клонирование проекта и другие настройки


[Материал позаимствован из видеоролика Alexey Goloburdin](https://github.com/alexey-goloburdin/debian-set-up-for-django "Ссылка на github с резюме по видеоролику")


### Устанавливаем python


0. Скачиваем необходимую версию python
	> `wget https://www.python.org/ftp/python/3.9.0/Python-3.9.0.tgz ; \`

0. Распаковываем архив 
	> `tar xvf Python-3.9.* ; \`

0. Собираем из исходника	
	>`cd Python-3.9.0 ; \`
	>`mkdir ~/.python ; \`
	>`./configure --enable-optimizations --prefix=/home/{user_name}/.python ; \`
	>`make -j8 ; \`
	>`sudo make altinstall`
	

0. Обновляем pip
    > `	sudo /home/{user_name}/.python/bin/python3.9 -m pip install -U pip`

### Клонируем наш код
0. Создаем папку, в которой будет располагаться наш код
    > `mkdir ~/code`
    > `cd code`

0. Создаем и активируем виртуальное окружение
    > `~/.python/bin/python3.9 -m venv env`
    > `. ./env/bin/activate`

0. Клонируем наш репозиторий
    > `git clone <url>` 


### Настраиваем gunicorn

0. Устанавливаем gunicorn
    > `pip install gunicorn`

0.  Создаем файл gunicorn_config.py (в директории с settings) и задаем настройки:
	> `command = '/home/{user_name}/code/env/bin/gunicorn'`
	> `pythonpath = '/home/{user_name}/code/test_project'`
	> `bind = '127.0.0.1:8001'`
	> `workers = 5`
	> `user = '{user_name}'`
	> `limit_request_fields = 32000`
	> `limit_request_field_size = 0`
	> `raw_env = 'DJANGO_SETTINGS_MODULE=test_project.settings'`

0. В директории, в которой лежит файл manage.py, создаем файл скрипт для старта gunicorn
	> `mkdir bin`
	> `vim /bin/start_gunicorn.sh`
	> Вставляем  ```
	#!/bin/bash
	source /home/{user_name}/code/env/bin/activate
	exec gunicorn -c "/home/{user_name}/code/test_project/test_project/gunicorn_config.py" test_project.wsgi```
	> Навешиваем права `chmod +x bin/start_gunicorn.sh`

Пробуем запустить.

Если все работает хорошо, то теперь необходимо настроить проксирование запросов с nginx. Открываем файл sudo vim /etc/nginx/sites-enabled/default и вставляем:
```
        location / {
                #try_files $uri $uri/ =404;
                proxy_pass http://127.0.0.1:8001;
                proxy_set_header X-Forwarded-Host $server_name;
                proxy_set_header X-Real-IP $remote_addr;
                add_header P3P 'CP="ALL DSP COR PSAa PSDa OUR NOR ONL UNI COM NAV"';
                add_header Access_Control-Allow-Origin *;
        }
```

### Настраиваем SUPERVISOR

В папке ect/supervisor/conf.d/ создаем файл <name_project>.conf
    > `sudo vim /ect/supervisor/conf.d/test_project.conf`
```
[program:www_gunicorn]
        command=/home/{user_name}/code/test_project/bin/start_gunicorn.sh
        user={user_name}
        process_name=%(program_name)s
        numprocs=1
        autostart=true
        autorestart=true
        redirect_stderr=true
 ```

 Запускаем supervisor: `sudo service supervisor start`

