---
layout: post
title: Автоматический деплой NodeJS приложения
date:   2019-01-01 20:06:00 +0530
categories: NodeJS GitHub
---

На данный момент я занимаюсь разработкой своего проекта [TryCode][1] и для деплоя, я использую два разных сервера. Каждый раз мне приходится пушить изменения на GitLab, потом заходить на SSH, делать `git pull` и перезапускать нодовское приложение.

В целлом, на все эти действия уходит около 2-3 минут. Однако, когда часто вносишь изменения и заливаешь их на прод, это всё начинает надоедать.

Хочу отметить, что я слышал про Docker и на момент написания этой статьи узнал ещё про какие-то сервисы, для автоматического деплоя. Например, [DeployBot][2]. Скажу честно, было лень изучать докер и настраивать его. Я решил написать несколько скриптов и NPM-команд, для автоматического деплоя и сборки приложения на проде.

###  [#][3] Задача

Требуется оптимизировать процесс деплоя приложения на прод и сохранить драгоценные 3 минуты моей жизни.

###  [#][3] Решение

Для начала, надо разобраться с NPM-скриптами (командами). В моём случае, для деплоя на продакшн нужно:

1. Очищать последнюю сборку приложения.
2. Запускать сборку приложения.
3. Пушить всё в репозиторий.
4. Зайти на SSH-сервер, перейти в директорию `/var/www/trycode.pw` и выполнить `git pull`.
5. Рестарнуть запущенное приложение на проде с помощью [pm2][4].

Советую вам разбить на команды все действия, а потом выполнять их поочерёдно. В моём случае, я написал следующие NPM-скрипты.

С помощью NPM-команды `prod-deploy`, происходит всё перечисленное выше. Команда `prod-build` запускает нодовский скрипт `prebuild.js`, который хранится в отдельной папке `scripts`.

Содержимое скрипта выглядит так:


    const exec = require('child_process').exec;

    async function run() {
      await exec('rm -rf ./src/.next');
      await exec('npm run build');
    }

    run();

Выполняется команда удаления папки с собранным приложением (в вашем случае это может быть `build`, `dist`) и запускается сборка приложения.

Далее происходит деплой всех изменений на GitLab с помощью NPM-команды `deploy`. Команда поочередно выполняет добавление изменений, коммит и пуш.

После пуша изменений на GitLab, запускается нодовский скрипт `pull.js`, который подключается к SSH-серверу и скачивает последние изменения.


    const node_ssh = require('node-ssh');
    const ssh = new node_ssh();

    ssh
      .connect({
        host: '144.88.19.48',
        username: 'root',
        privateKey: '/Users/archakov/.ssh/id_rsa',
      })
      .then(() => {
        ssh.execCommand('git pull origin master', { cwd: '/var/www/trycode.pw' }).then(function() {
          console.log('✅ Changes successfully pulled!');
          ssh.execCommand('yarn', { cwd: '/var/www/trycode.pw' }).then(function() {
            console.log('✅ Dependecies installed');
            ssh.execCommand('pm2 restart server', { cwd: '/var/www/trycode.pw' }).then(function() {
              console.log('? Server restarted!');
              process.exit();
            });
          });
        });
      });

    // ИЛИ с async / await

    const node_ssh = require('node-ssh');
    const ssh = new node_ssh();

    async function run() {
      await ssh.connect({
        host: '144.88.19.48',
        username: 'root',
        privateKey: '/Users/archakov/.ssh/id_rsa',
      });
      await ssh.execCommand('git pull origin master', { cwd: '/var/www/trycode.pw' });
      console.log('✅ Changes successfully pulled!');
      await ssh.execCommand('yarn', { cwd: '/var/www/trycode.pw' });
      console.log('✅ Dependecies installed.');
      await ssh.execCommand('pm2 restart server', { cwd: '/var/www/trycode.pw' });
      console.log('? Server restarted!');
        process.exit();
    }

    run();

Для подключения к SSH-серверу, используется библиотека `node-ssh`. [Создайте ssh-ключ][5] для подключения к серверу без пароля и укажите путь к этому файлу в свойстве `privateKey`.

* **host** — указываем адрес SSH-сервера.
* **username** — логин на вашем SSH-сервере.
* **privateKey** — SSH-ключ для подключения к серверу без пароля. (также можно указать пароль по желанию, заменив на `password` и в значении указав пароль).

После успешного подключения, скрипт переходит в папку `/var/www/trycode.pw` и выполняет ряд команд:

1. Скачивание изменений с репозитория.
2. Установка новых зависимостей командой `yarn` (альтернатива `npm install`).
3. Рестарт нодовского приложения.

Работает всё отлично и деплоить приложение на прод уже не так больно. Перфектли!

P.S.: Буду рад, если расскажете в комментах или поделитесь ссылкой, как вы решаете подобный деплой или как сделать то же самое на докере.

--
[Source](https://archakov.im/post/avtomaticheskij-deploj-nodejs-prilozheniya.html "Автоматический деплой NodeJS приложения")

[1]: https://archakov.im/post/trycode-collaborative-online-editor.html
[2]: https://archakov.im/deploybot.com
[3]:
[4]: https://pm2.io/runtime
[5]: https://www.google.com/search?q=создать+ssh+ключ


