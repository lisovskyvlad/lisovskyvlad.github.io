---
layout: post
title: "Переезд с MySQL на PostgreSQL"
date: 2013-12-18 23:00:00 +0400
comments: false
categories: [mysql, postgresql]
---

В настоящее время я работаю в [lakehouse:labs](http://lakehouse.ru/labs/). На наших
ключевых проектах мы постепенно переезжаем с MySQL на PostgreSQL.

К сожалению, упомянутые выше две СУБД не совместимы на уровне дампа, а потому появилась
необходимость в инструменте для миграции. Большую часть кода я пишу на Ruby,
поэтому приоритет отдавался инструментам на этом языке. После изучения возможных
[вариантов](http://wiki.postgresql.org/wiki/Converting_from_other_Databases_to_PostgreSQL#MySQL)
я и мой коллега [Игорь Варавко](http://blog.ivaravko.com/) решили остановиться на геме под
названием taps-taps, под катом я опишу как им пользоваться.

<!-- more -->

Для начала вам нужен ruby версии не ниже 1.9.2, на 1.8.* я не проверял - может быть и заработает.
Процесс установки ruby в этом посте я опущу, но в будущем планирую написать отдельный пост,
посвященный установке ruby. В интернетах можно найти большое количество постов и инструкций,
но многие из них мне не нравятся, либо же они уже не актуальны.

Далее, установим все необходимые гемы:

``` bash Установка гемов
$ gem install taps-taps pg mysql
```

Собственно, ставим сам гем taps-taps, pg и mysql, хочу отметить что не mysql2,
а именно mysql, taps-taps именно его использует для подключения к mysql.

Забавный факт - гем [taps-taps](https://github.com/wijet/taps) на самом деле форк гема
[taps](https://github.com/ricardochimal/taps), автор которого, как это часто бывает в oos, забросил
своё детище и оно за время его отсутствия обрасло issues и pull request на гитхабе.

taps-taps под капотом использует sinatra и rest-client, то есть используется клиент-серверный подход,
именно этот факт заставил попробовать его, а не решение от
[Макса Лапшина](https://github.com/maxlapshin/mysql2postgres).

Чтобы заставить работать эту балалайку, нужно запустить taps сервер и taps клиент.
При запуске клиента, если всё сложилось хорошо, начнётся миграция, причём она идёт посредством
обмена json по http, используется basic http authorization.

Запускаем сервер:

``` bash Запуск taps сервера
$ taps server mysql://db_user:password@localhost/db_name?encoding=utf8 httpuser httppassword
```

В данном случае я предлагаю использовать сервер, который коннектится к mysql, хотя
возможные разные варианты подключения,
в [readme](https://github.com/wijet/taps/blob/master/README.rdoc)
приведены другие варианты использования.

Нужно создать пустую базу данных в PostgreSQL, обычно я это делаю вот так:

```
$ creatdb project_db_dev
```

Запускаем клиент, который будет принимать данные:

``` bash Запуск taps клиента
$ taps pull postgres://db_user:password@localhost/db_name http://httpuser:httppassword@localhost:5000
```

Уточню некоторые вещи:

- db_user - имя пользователя, который имеет доступ к бд
- db_name - имя бд из которой мигрируем и в которую
- httpuser - имя пользователя для http аутентификации
- httppassword - пароль пользователя для http аутентификаци

Если вы всё сделали правильно, то увидите примерно такой вывод к консоле:

``` bash Лог
Receiving schema
Schema:          0% |                                          | ETA:  --:--:--
Schema:          2% |                                          | ETA:  00:00:16
Schema:          5% |==                                        | ETA:  00:00:15
...
Schema:        100% |==========================================| Time: 00:00:16
Receiving data
34 tables, 6,800 records
admins:        100% |==========================================| Time: 00:00:00
...
versions:      100% |==========================================| Time: 00:00:00
Resetting sequences
```

После того как процесс миграции прошел, подключаемся к postgresql и проверяем
приложение, всё ли работает.

Проекты, которые мы переносим, написаны плохо:

- во многих местах используется сырой SQL, хотя стоило бы писать на ActiveRecord
- используются некоторые SQL функции, специфичный для mysql, например DATE_ADD.

Вот такой код:
```ruby
seminars.started_at <= DATE_ADD(#{from},INTERVAL -4 HOUR)
```

Нужно заменить на такой:
```ruby
seminars.started_at <= #{from}::timestamp - INTERVAL '4 HOURS'"
```

Также был найден забавный баг:
```ruby
def partimer_conditions
  cond =  '1'
  cond += " AND user.full_name LIKE ... "
  cond
end
```

Фича в том, что postgres ругается на подобный код:
```
ERROR:  argument of WHERE must be type boolean, not type integer
```

Если же вместо 1 поставить true, то всё будет ок.


Ах да, предыдущие погромисты не знали о ActiveRecord::Relation,
не поняли концепцию [method chaining](https://en.wikipedia.org/wiki/Method_chaining)
и придумали свой конструктор запросов - бывшие PHP программисты, наверное.

Эти и много других отличий между этими СУБД описаны
[тут](http://www.the-art-of-web.com/sql/postgres-mysql/), [тут](http://eax.me/postgresql-vs-mysql/),
[тут](http://www.wikivs.com/wiki/MySQL_vs_PostgreSQL) и много где ещё.

Возможно, кому-то интересна моя мотивация - зачем мигрировать с MySQL на PostgreSQL,
ведь MySQL это проверенное большим количеством проектов и инсталляций решение.
На мой скромный взгляд, MySQL переживает далеко не лучшие времена.
Находящаяся под Oracle, имеющая куча форков и ответвлений, с меняющимися лицензиями -
всё это не вызывает желания пользоваться этой СУБД.

На её фоне PostgreSQL выглядит куда более привлекательнее - огромное количество новых
фич, типа [hstore](http://www.postgresql.org/docs/9.3/static/hstore.html),
поддержка типа [json](http://www.postgresql.org/docs/9.3/static/functions-json.html)
и его индексация, независимые разработчики по всему миру,
надёжность как у танка [Т-90 МС](https://ru.wikipedia.org/wiki/Т-90), большое количество
интересных российских проектов, использующих его в качестве основного хранилища.

Буду рад, если моё решение поможет и вам!
