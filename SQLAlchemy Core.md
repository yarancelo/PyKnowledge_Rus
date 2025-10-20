[[SQLAlchemy]] Core представляет собой низкоуровневый API для доступа для [[Работа с БД|работы с базой данных]] (в отличие, от [[SQLAlchemy ORM|ORM]]), это инструмент для конструирования [[PostgreSQL|SQL]] запросов через Python-объекты, без написания чистого SQL. В Core явно описываются таблицы и колонки и выполняются команды ([[Работа с БД|select, insert, update, delete]]). 

### **[[Добавление таблиц|Создание таблиц]] в Core**
```python
from sqlalchemy import ForeignKey, String, Table, MetaData, Column, BigInteger

metadata = MetaData()

user_table = Table(
  'users',
  metadata,
  Column('pk_user_id', BigInteger, primary_key=True),
  Column('user_name', String(30), unique=True),
  Column('first_name', String(30)),
  Column('second_name', String(30))
)

email_table = Table(
  'email_addresses',
  metadata,
  Column('pk_email_id', BigInteger, primary_key=True),
  Column('fk_user_id', ForeignKey('users.pk_user_id', ondelete='CASCADE')),
  Column('email', String(), nullable=False)
)
```

В данном коде создаются две таблицы: user_table и email_table. Таблица объявляется с помощью класса Table(), импортированная из sqlalchemy. Первым аргументом передается название таблицы, вторым, метаданные, которые должны быть одни и те же для всех таблиц.
Колонки создаются также, передавая внутрь аргументы - классы Column().
Классы Column() нужны для создания колонок, формат колонки передается в аргументах:
Column(<название колонки>, <тип данных>, <констреинты>). Типы данных импортируются также из sqlalchemy.

Для создания таблиц, нужно вызвать соответствующий метод у метаданных:
```python
with engine.connect() as connection:
  metadata.drop_all(engine) # удалить все таблицы
  metadata.create_all(engine) # создать все таблицы
```

Также, для работой с базой, [[Хорошие практики|хорошей практикой]] является проведение всех операций с БД через [[Менеджер состояния|контекстный менеджер]], для того, чтобы происходило автоматическое открытие и закрытие соединения и даже в том случае, если произойдет ошибка, соединение будет должным образом закрыто.

### **[[Выборка данных|Запросы]] в Core**
Запросы в Core происходят с использованием statement'ов - выражений, содержащих определённое действие:
```python
statement = insert(user_table).values(
  user_name='nikitay',
  first_name='Nikita',
  second_name='Timchenko'
)
```

Вот так может быть представлено выражение по вставке значений в таблицу user_table, созданную выше. При этом, значения передаются в виде аргументов к функции. После того, как выражение создано, оно может быть скомпилировано под определенный диалект - ```pg_statement = statement.compile(engine, postgresql.dialect())```, либо сразу передано к исполнению:
```python
with engine.begin() as connection:
  result = connection.execute(pg_statement)
  print(result.insert_primary_key) # возвращаем значение ключа
```

Разница между ```engine.begin() и engine.connect()``` состоит в том, что engine.begin() автоматически начинает транзакцию и в случае, если не возникнут никакие ошибки, при закрытии соединения совершает commit.
Передаваемый запрос исполняется в методе connection.execute(), в который в качестве аргумента передается выражение, после чего данный метод возвращает результат:
```
INSERT INTO users (user_name, first_name, second_name) VALUES (%(user_name)s, %(first_name)s, %(second_name)s) RETURNING users.pk_user_id

Вывод в консоль:
(1,)
```

### **Вставка в таблицу**
[[Заполнение|Вставка]] в таблицу с помощью SQLAlchemy Core осуществляется с помощью ключевого слова insert, которое передается в запрос:
```python
statement = insert(user_table).values(
  user_name='kirillator',
  first_name='Kirill',
  second_name='Goncharov'
)

with engine.begin() as connection:
  connection.execute(statement)
```

Другой вариант сделать вставку - вставка большого количества строк одновременно:
```python
statement = insert(user_table)

with engine.begin() as connection:
  result = connection.execute(
    statement,
    parameters=[
      {'user_name': 'main_cocker',
       'first_name': 'vladimir',
       'second_name': 'leontiy'
      },
      {'user_name': 'est_cho_420',
       'first_name': 'maxim',
       'second_name': 'kruglove'
      },
      {'user_name': 'nikitay',
       'first_name': 'nikita',
       'second_name': 'timchenko'
      }
    ]
    )
```

В данном случае, insert statement задается, фактически, пустым, и все данные вносятся непосредственно в запрос через аргумент parameters. Данный режим вставки называется bulk insert.

### **Запрос данных в Core**
Запросы в SQLAlchemy производится с помощью ключевого слова select:
```python
with engine.begin() as connection: # выборка по критерию OR (или то или то)
  result = connection.execute(
    select(user_table.c.user_name, user_table.c.first_name).where(
      or_(user_table.c.user_name.startswith("c"),
          user_table.c.first_name.contains('i'))
      )
  )
  print(result.all())
```

Данный запрос получает данные по колонкам user_name и first_name с условием user_name начинается с "c" или first_name содержит "i".
Следующий же запрос показывает как в том же запросе использовать условие "И". Как мы видим, при указании двух условий, "И" используется по умолчанию:
```python
with engine.begin() as connection: # выборка по критерию AND (и то и то)
  result = connection.execute(
    select(user_table.c.user_name, user_table.c.first_name).where(
      user_table.c.user_name.startswith("c"),
      user_table.c.first_name.contains('i')
      )
  )
  print(result.all())
```

Для того, чтобы с данными можно было удобно работать, можно возвращать из БД данные в формате словарей:
```python
with engine.begin() as connection: # выборка с помощью оператора IN
  result = connection.execute(
    select(user_table.c.user_name, user_table.c.first_name).where(
      user_table.c.pk_user_id.in_(list(range(1,5)))
      )
  )

  #print(result.all()) # Возващает список из кортежей со значениями
  print(result.mappings().all()) # Возваращает список словарей с названиями полей и со значениями
```
```
Вывод:
[{'user_name': 'main_cocker', 'first_name': 'vladimir'}, {'user_name': 'est_cho_420', 'first_name': 'maxim'}, {'user_name': 'nikitay', 'first_name': 'nikita'}, {'user_name': 'sleepy_dev', 'first_name': 'alexey'}]
```

Также при осуществлении запроса к колонке, можно либо задать, либо изменить ее отображаемое имя:
```python
with engine.begin() as connection: # выборка имени и фамилии как full_name с помощью label
  result = connection.execute(
    select(user_table.c.pk_user_id, (user_table.c.first_name + " " + user_table.c.second_name).label("full_name")).where(
      user_table.c.pk_user_id.in_(list(range(1,4)))
      )
  )
  for res in result: # result может быть представлено как namedtuple, tuple, .mappings()
    print(res.pk_user_id, res.full_name) # выводит данные построчно
```
```
Вывод:
1 vladimir leontiy
2 maxim kruglove
3 nikita timchenko
```

В данном примере мы создали новую колонку для вывода с именем full_name и вывели ее построчно.

### JOIN-запросы
JOIN-запросы могут быть осуществлены, если есть связь между двумя и более таблицами. 
```python
with engine.begin() as connection: # выборка имени и фамилии как full_name с помощью label
  result = connection.execute(
    select(
      email_table.c.email.label("current_email"),
      (user_table.c.first_name + " " + user_table.c.second_name).label("full_name")
    ).where(
      user_table.c.pk_user_id < 5
    ).join_from(user_table, email_table, user_table.c.pk_user_id == email_table.c.fk_user_id)
    # в join_from последний аргумент можно не указывать, sqlalhemy автомтатически понимает, какая колонка для какой является foreign_key
  )
  print(result.mappings().all())
```
```
Вывод:
[{'current_email': 'vladimir.leontiy@gmail.com', 'full_name': 'vladimir leontiy'}, {'current_email': 'maxim.kruglove@mail.ru', 'full_name': 'maxim kruglove'}, {'current_email': 'nikita.timchenko@yandex.ru', 'full_name': 'nikita timchenko'}, {'current_email': 'alexey.borisov@outlook.com', 'full_name': 'alexey borisov'}]
```

Одного и того же результата можно добиться как с помощью JOIN FROM, так и с помощью просто JOIN:
```python
with engine.begin() as connection: # выборка имени и фамилии как full_name с помощью label
  result = connection.execute(
    select(
      email_table.c.email.label("current_email"),
      (user_table.c.first_name + " " + user_table.c.second_name).label("full_name")
    ).where(
      user_table.c.pk_user_id < 5
    ).join(email_table, isouter=True) # left outer join
    # в join достаточно указать только какую таблицу мы соединяем и все
  )
  print(result.mappings().all())
```

Полученный результат можно отсортировать с помощью ORDER BY:
```python
with engine.begin() as connection: # выборка имени и фамилии как full_name с помощью label
  result = connection.execute(
    select(
      email_table.c.email.label("current_email"),
      (user_table.c.first_name + " " + user_table.c.second_name).label("full_name")
    ).where(
      user_table.c.pk_user_id < 5
    ).join(
      email_table,
      isouter=True
    ).order_by(
      email_table.c.email.desc() # сортировка значений по адресам
    )
  )
  print(result.mappings().all())
```

Также сортировка возможна по full_name, то есть, только по колонке, созданной для вывода:
```python
with engine.begin() as connection: # выборка имени и фамилии как full_name с помощью label
  result = connection.execute(
    select(
      email_table.c.email.label("current_email"),
      (user_table.c.first_name + " " + user_table.c.second_name).label("full_name")
    ).where(
      user_table.c.pk_user_id < 5
    ).join(
      email_table,
      isouter=True
    ).order_by(
      "full_name" # сортировка значений по созданному параметру full_name
    )
  )
  print(result.mappings().all())
```

### **DML в Core**
### Update
Update-запрос в SQLAlchemy Core:
```python
with engine.begin() as connection:
  connection.execute(
    update(user_table).where(user_table.c.first_name == 'vladimir').values(first_name='Vladimir') # у пользователя с именем vladimir поменять имя на leontiy
  )
  result = connection.execute(
    select(user_table).where(user_table.c.pk_user_id == 1) # вывести пользователя с id == 1
  )
  print(result.mappings().all()) # [{'pk_user_id': 1, 'user_name': 'main_cocker', 'first_name': 'Vladimir', 'second_name': 'leontiy'}]
```

Update-запросы производятся с помощью ключевого слова update(), куда во внутрь передаются условия обновления. Помимо одиночного изменения данных, возможен bulk-update данных с помощью пустого statement и bindparam():
```python
with engine.begin() as connection:
  statement = update(user_table).where(user_table.c.first_name == bindparam('old_name')).values(first_name=bindparam('new_name'))
  # bind используется чтобы создавать переменные, которые можно будет вставить позже в блоке execute
  connection.execute
  (
    statement,
    [
      {'old_name': 'maxim', 'new_name': 'Maxim'},
      {'old_name': 'nikita', 'new_name': 'Nikita'}
    ]
  )
  result = connection.execute(
    select(user_table.c.first_name).where(user_table.c.pk_user_id < 5)
  )
  for res in result:
    print(res.first_name)
```

### Delete
Важный момент в удалении заключается в том, что удаляя зависимые сущности, таблицы должны поддерживать каскадное удаление ON DELETE CASCADE. Удаление данных из таблиц, где есть зависимости, возможно только в этом режиме.
Обычное удаление по параметру:
```python
with engine.begin() as connection:
  connection.execute(
    delete(user_table).where(user_table.c.pk_user_id > 15) # удаляем строки с значением id > 15
  )
  result = connection.execute(
    select(user_table.c.pk_user_id, user_table.c.user_name).where(user_table.c.pk_user_id > 12) # выбираем строки с значением id > 12
  )
  for res in result:
    print(f'{res.pk_user_id}: {res.user_name}')
```

Также возможно проводить delete по критерию из другой таблицы:
```python
with engine.begin() as connection:
  delete_statement = (
    delete(user_table)
    .where(user_table.c.pk_user_id == email_table.c.fk_user_id)
    .where(email_table.c.email == 'sergey.egorov@mail.ru')
  )
  connection.execute(delete_statement)
  result = connection.execute(
    select(user_table.c.pk_user_id, user_table.c.user_name).where(
      user_table.c.pk_user_id > 7,
      user_table.c.pk_user_id < 12
    )
  )
  for res in result:
    print(f'{res.pk_user_id}: {res.user_name}')
```

Также, при удалении данных из таблицы, можно с помощью returning из запроса возвращать некоторую информацию, например, информацию о удаленных элементах:
```python
with engine.begin() as connection:
  result = connection.execute(
    delete(user_table).where(
      user_table.c.pk_user_id > 4,
      user_table.c.pk_user_id < 7
    )
  )
  print(f'Deleted {result.rowcount} rows')

```
```
Вывод:
Deleted 2 rows
```
```python
with engine.begin() as connection:
  result = connection.execute(
    delete(user_table)
    .where(user_table.c.pk_user_id.in_([4]))
    .returning(user_table.c.pk_user_id, user_table.c.user_name)
  )
  print(result.mappings().all())
```
```
Вывод:
[{'pk_user_id': 4, 'user_name': 'sleepy_dev'}]
```