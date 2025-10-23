Режим работы [[Модули|модуля]] [[SQLAlchemy]], при котором, работа с базой данных осуществляется через Объектно-Реляционную модель. В объектно-реляционной модели каждая сущность в БД представляется в виде Python-[[Классы|класса]]. Для упрощения работы с базой данных, ORM вводит абстракцию над базой. 
В основе работы ORM лежит паттерн (unit-of-work) - совокупность всего, что должно взаимодействовать с базой данных и должно быть объединено для одновременной отправки изменений. Изменения отправляются не по одиночке, а в группах, называемых транзакцией. Также, в случае, если какое-то количество запросов бесполезно, ORM и СУБД могут их отбрасывать (например, добавление и удаление одних и тех же данных). В данном случае ORM может "додумывать" за базу данных, чтобы лишний раз не нагружать БД.
Все реляционные базы данных имеют транзакции - группы последовательных операций с БД, они являются основой для мультиверсионности и конкурентного доступа к данным, внутри [[PostgreSQL]] все операции оборачиваются в транзакции. Транзакции помогают обеспечить соответствие работы с базой данных к принципам [[ACID]]. 

В SQLAlchemy вводится понятие сессий - некого конвейера, по которому движутся данные. 
1) Не находящиеся в БД
2) Находящиеся в транзакции
3) Находящиеся в БД

В сессии у объекта может быть 5 состояний:
* Transistent - временный, до добавления в сессию, не имеет ассоциации с БД
* Pending - ожидающий, добавлен в сессию с помощью session.add(), но все еще не находится в БД, но может там появиться в будущем с помощью session.flush(), так как данные прошли интеграцию
* Persistent - находится в сессии и имеет ассоциацию с реальной записью в базе данных, все изменения в отношении этого объекта, отразятся и на записи внутри БД.
* Deleted - удаленный, запись в БД, ассоциированная с этим объектом все еще существует, однако сам объект помечен на удаление и после flush() запись в БД будет также удалена.
* Detached - отсоединенный, состояние объекта, который ранее имел ассоциацию с базой данных, но в настоящий момент отсоединен от сессии и нельзя достоверно утверждать, что данные все еще актуальны.

## **[[Добавление таблиц|Создание таблиц]]**
При создании таблиц
```python
class Base(DeclarativeBase):
  pass

class User(Base):
  __tablename__ = 'users'
  id: Mapped[int] = mapped_column(primary_key=True, autoincrement=True)
  name: Mapped[str] = mapped_column(nullable=False)
  age: Mapped[int]
  
  
Base.metadata.drop_all(engine)
Base.metadata.create_all(engine)
```

В отличие, от [[SQLAlchemy Core|Core]], в ORM нам не нужно отдельно создавать объект метаданных для таблиц. В ORM объект метаданных содержится в классе ```DeclarativeBase```, от которого мы сначала наследуем базовый класс, а потом от базового класса наследуем сами таблицы. После чего, для их создания мы обращаемся **Base.metadata**.

Но для того, чтобы уже именно работать с таблицами, нужна сессия, сессию можно получить двумя способами:
```python
session = Session(bind=engine, expire_on_commit=False)
```
В данном способе мы создаем сессию, через класс Session, привязывая к нему движок, данный способ не является [[Хорошие практики|хорошей практикой]], так как мы не использовали [[Менеджер состояния|контекстный менеджер]] для автоматического открытия транзакции и ее коммита. Лучше делать так:
```python
SessionLocal = sessionmaker(bind=engine, expire_on_commit=False)

with SessionLocal.begin() as session:
  user = User(name="vladimir", age=22)
  session.add(user)
```

Данный контекстный менеджер будет автоматически делать и session.flush() и session.commit(), однако в повествовании ниже будет использоваться именно первый вариант, чтобы видеть различия между стадиями сессий у объекта.

## **[[Заполнение|Создание и добавление]] записей**
Как описано ранее, работа с записями происходит в виде работы с python-объектами. Попробуем создать объект пользователя:
```python
user = User(name="vladimir", age=22)
```

Мы создаем объект через класс User, и обращаемся к колонкам таблицы как к атрибутам класса. Теперь мы можем узнать статус объекта:
```python
insp = inspect(user)
print("is transistent?", insp.transient)
```
```
Вывод:
is transistent? True
```

Только что созданный объект является транзистентным, для того, чтобы добавить объект в сессию, нужно воспользоваться методом **session.add()**.
```python
session.add(user)
print("is pending?", insp.pending, "Is transistent?", insp.transient)
```
```
Вывод:
is pending? True Is transistent? False
```

После того, как объект добавлен в сессию, он становится pending, то есть ожидающим, и перестает быть транзистентным, тем не менее, он все еще не находится в базе данных. Добавление объекта в транзакцию происходит с помощью **session.flush()**:
```python
session.flush() # перемещение данных из сессии в транзакцию
print("is persistent?", insp.persistent, "Is pending?", insp.pending)
```
```
Вывод:
is persistent? True Is pending? False
```

Теперь объект находится в транзакции внутри БД. Но, все изменения все еще возможно откатить обратно, если сделать rollback. Чтобы изменения окончательно перенеслись в базу данных, нужно сделать session.commit().
```python
session.commit() # обьект из тразакции был перемещен в БД
```

После совершения commit, все объекты из транзакции переносятся в БД и появляется ассоциированная с ними запись.
```python
session.delete(user)
print('is deleted?', insp.deleted)
```
```
Вывод:
is deleted? False
```

session.delete() приводит к удалению объекта, однако, поскольку flush не был произведен, данные изменения на базе, пока что, не отображаются. после session.flush() изменения попадут в транзакцию:
```python
session.flush()
print('is deleted?', insp.deleted)
```
```
is deleted? True
```

Но все же, ассоциированная запись с этим объектом еще существует в БД.
```python
session.commit()
insp = inspect(user)
print('is deleted?', insp.deleted, 'Is detached?', insp.detached)
```
```Вывод:
is deleted? True Is detached? False
```

После commit, ассоциированная с объектом запись удаляется и он становится отсоединенным.

## **Коллекции объектов**
В процессе изменения сессий, объекты перемещаются по коллекциям, существуют следующие коллекции:
* session.new - коллекция новых объектов со статусом pending
* session.dirty - коллекция измененных объектов, имеющих статус persistant
* session.deleted - коллекция объектов, помеченных на удаление
* session.identity_map - коллекция всех объектов со статусом persistant

## **[[Выборка данных|Запрос данных и их изменение]]**
Мы можем получить объект User с помощью session.get(). get можно использовать тогда, когда нам известно значение primary_key.
```python
session_user = session.get(User, ident='2')
```

Данный запрос находит объект базы данных с id=2, фактически, этот запрос эквивалентен запросу:
```sql
SELECT user.name, user.id, user.age FROM users WHERE users.id == 2
```

Для того, чтобы нам изменить объект, хранящийся в БД, достаточно всего лишь изменить его атрибут, это аналог UPDATE запроса:
```python
session_user.age = 22
```

Также можно произвести запрос с помощью scalar, который возвращает 1 объект:
```python
user_from_db = session.scalar(select(User).where(User.id == 3))
```

select() это максимально гибкий запрос и он поддерживает множество "функций": where(), join(), order_by(), limit(), offset(), group_by(), having(), и т.д.

### **Relationships (Отношения между таблицами)**
Отношения бывают [[Один к одному (SQL)|один к одному]], [[Один ко многим (SQL)|один ко многим]], и [[Многие ко многим (SQL)|многие ко многим]]. Все эти виды отношений возможно реализовать в [[SQLAlchemy ORM]].

## Один-к-одному
Примером связи один к одному может являться отношение сущностей муж-мена (у каждого мужа может быть только одна жена и у каждой жены может быть только один муж).
Пример:
```python
class Base(DeclarativeBase):
  pass

  
class User(Base):
  __tablename__ = 'users'
  id: Mapped[int] = mapped_column(primary_key=True, autoincrement=True)
  name: Mapped[str] = mapped_column(nullable=False)
  age: Mapped[int]

  address: Mapped["Address"] = relationship(back_populates="user", uselist=False)

  def __repr__(self):
    return f'User: {self.id=}, {self.name=}, {self.age=}'
  

class Address(Base):
  __tablename__ = 'addresses'
  email: Mapped[str] = mapped_column(primary_key=True, nullable=False)
  user_fk: Mapped[int] = mapped_column(ForeignKey('users.id'))

  user: Mapped["User"] = relationship(back_populates="address", uselist=False)

  def __repr__(self):
    return f'Address: {self.email=}, {self.user_fk=}'
```

В данном примере у одного пользователя может быть только один адрес электронной почты. Осуществляется это тем, что, во первых, таблица **"addresses"** имеет поле **"user_fk"**, являющееся внешним ключом с атрибутом **"users.id"**, а во вторых, и в addresses и в users существует атрибут ```address: Mapped["Addreses"] = relationship(backpopulates="user", uselist=False)```. Этот атрибут поясняет, что мы ссылаемся на таблицу Addresses, этот параметр указывается в виде строки, чтобы не возникало циклических импортов имен. После чего после символа =, указывается тип отношения, backpopulates указывает, что мы делаем связь с атрибутом user (не с таблицей users). Параметр uselist указывает, что возможна только связь [[Один к одному (SQL)|один к одному]].

## Один-ко-многим
Примером связи один ко многим может являться отношение сущностей учитель-ученики (у одного учителя много учеников, но у каждого ученика один единственный учитель, если это начальная школа).
В следующем примере показаны таблицы, в которых у одного человека может быть несколько адресов электронной почты:
```python
class Base(DeclarativeBase):
  pass
  

class User(Base):
  __tablename__ = 'users'
  id: Mapped[int] = mapped_column(primary_key=True, autoincrement=True)
  name: Mapped[str] = mapped_column(nullable=False)
  age: Mapped[int]
  
  addresses: Mapped[list["Address"]] = relationship(back_populates="user", uselist=True)

  def __repr__(self):
    return f'User: {self.id=}, {self.name=}, {self.age=}'
  

class Address(Base):
  __tablename__ = 'addresses'
  email: Mapped[str] = mapped_column(primary_key=True, nullable=False)
  user_fk: Mapped[int] = mapped_column(ForeignKey('users.id'))

  user: Mapped["User"] = relationship(back_populates="addresses", uselist=False)

  def __repr__(self):
    return f'Address: {self.email=}, {self.user_fk=}'
```

Описывая данный пример, мы могли бы сказать, что таблица users главенствует над таблицей addresses, так как, это один пользователь может иметь несколько почт, а не наоборот. Соответственно, мы указываем в таблице users атрибут addreses, по первых, как list адресов, а во вторых, параметр uselist = True, показывая этим, что в addresses может быть много адресов. А также, в таблице Address, атрибут user ссылается уже не на атрибут address, а на атрибут addresses, т.е. в множественном числе.

## Многие-ко-многим
Примером связи сущностей многие ко многим может являться отношение родителей и детей (у каждого из родителей может быть несколько детей, а у каждого ребенка есть два родителя).
Пример ниже иллюстрирует систему, в которой каждый адрес электронной почты может принадлежать сразу нескольким сущностям, например, если адрес почты общий корпоративный, то скорее всего у него будет много пользователей:
```python
class Base(DeclarativeBase):
  pass
  

class User(Base):
  __tablename__ = 'users'
  id: Mapped[int] = mapped_column(primary_key=True, autoincrement=True)
  name: Mapped[str] = mapped_column(nullable=False)
  age: Mapped[int]

  addresses: Mapped[list["Address"]] = relationship(back_populates="users", uselist=True, secondary='user_address')

  def __repr__(self):
    return f'User: {self.id=}, {self.name=}, {self.age=}'


class Address(Base):
  __tablename__ = 'addresses'
  email: Mapped[str] = mapped_column(primary_key=True, nullable=False)

  users: Mapped[list["User"]] = relationship(back_populates="addresses", uselist=True, secondary='user_address')
  
  def __repr__(self):
    return f'Address: {self.email=}'
  

class UserAddress(Base): # таблица ассоциации
  __tablename__ = 'user_address'
  user_fk = mapped_column(ForeignKey('users.id'), primary_key=True)
  address_fk = mapped_column(ForeignKey('addresses.email'), primary_key=True)
  
  def __repr__(self):
    return f"UserAddress:{self.user_fk}, {self.address_fk}"
```

Для работы со связью многие ко многим необходимо введение дополнительной **ассоциативной** таблицы, в которой будет осуществляться сопоставление адреса с id пользователя, причем ключом в такой таблице будет не какое-то одно поле, а комбинация одного и другого поля, соответственно, она будет уникальной. 
Также, поскольку теперь таблицы логически равнозначные, в каждой из них в атрибуте, обозначающем отношение указано хранение не одного адреса либо пользователя, а списка адресов или пользователей, также параметр uselist у атрибутов обоих таблиц указан как True. Помимо этого, добавлен параметр secondary, значение которого ссылается на ассоциативную таблицу, обозначая связь многие-ко-многим.

## **Запросы и N+1 проблема**
Проблема N+1 проявляется при запросе данных из нескольких связанных таблиц именно в ORM взаимодействии с БД и заключается она в том, что при попытке обратиться к атрибуту объекта, имеющего связь с внешней таблицей, ORM делает множественное количество запросов в базу данных, излишне загружая ее. 
Для того, чтобы решить данную проблему, существуют стратегии загрузки:
* lazy load - ленивая подгрузка. Подгружает данные из связанных таблиц только при обращении к соответствующему атрибуту. В данной стратегии и проявляется **N + 1 проблема**, применяется по умолчанию
* joined load - вся информация загружается из БД одним запросом, может возвращать много дублирующихся данных. Подходит для небольших таблиц.
* select in load - делает один запрос для загрузки основной таблицы и еще один для загрузки всех связанных сущностей. Рекомендуется для использования по умолчанию, если не нужна агрегация в связанных сущностях. Походит для больших таблиц. 
* subquerry load - делает один запрос, содержащий внутри себя другой подзапрос со связанными сущностями. Решает проблему n+1, однако менее эффективна чем select in load. Используется в специфических случаях.
* raise loading - нужна для выявления n+1 проблемы, в случае ее возникновения выбрасывает исключение. Подходит для отладки. 

Рассмотрим схему в которой можно отчетливо видеть N+1 проблему:
```python
class User(Base):
  __tablename__ = 'users'
  id: Mapped[int] = mapped_column(primary_key=True, autoincrement=True)
  name: Mapped[str] = mapped_column(nullable=False)

  tasks: Mapped[list['Task']] = relationship(back_populates='user', uselist=True, lazy="select")

  def __repr__(self):
    return f"User: {self.id=}, {self.name=}"
  

class Task(Base):
  __tablename__ = 'tasks'
  id: Mapped[int] = mapped_column(primary_key=True, autoincrement=True)
  name: Mapped[str] = mapped_column(nullable=False)
  responsible_fk = mapped_column(ForeignKey('users.id'))

  user: Mapped['User'] = relationship(back_populates='tasks', uselist=False)

  def __repr__(self):
    return f'Task: {self.id=}, {self.name=}, {self.responsible_fk=}'
```

Можно увидеть, что в атрибуте tasks таблицы users указан параметр lazy="select". Это означает, что будет использоваться стратегия "lazy load". Наполним таблицу данными и произведем запрос пользователей и их задач:
```python:
with SessionLocal.begin() as conn:
    users = conn.scalars(select(User)).all()
    for user in users:
        print(user.name)
        for task in user.tasks:
          print("   ", task.name)
```

Мы увидим в выводе, что сначала ORM запросит список всех пользователей, а позже запросит задачи для каждого из пользователей:
```
2025-10-23 12:41:46,819 INFO sqlalchemy.engine.Engine SELECT tasks.id AS tasks_id, tasks.name AS tasks_name, tasks.responsible_fk AS tasks_responsible_fk
FROM tasks
WHERE %(param_1)s::INTEGER = tasks.responsible_fk
2025-10-23 12:41:46,820 INFO sqlalchemy.engine.Engine [generated in 0.00031s] {'param_1': 1}
    Задание 1
    Задание 2
maxim
2025-10-23 12:41:46,821 INFO sqlalchemy.engine.Engine SELECT tasks.id AS tasks_id, tasks.name AS tasks_name, tasks.responsible_fk AS tasks_responsible_fk
FROM tasks
WHERE %(param_1)s::INTEGER = tasks.responsible_fk
2025-10-23 12:41:46,822 INFO sqlalchemy.engine.Engine [cached since 0.002195s ago] {'param_1': 2}
    Задание 5
    Задание 6
nikita
2025-10-23 12:41:46,823 INFO sqlalchemy.engine.Engine SELECT tasks.id AS tasks_id, tasks.name AS tasks_name, tasks.responsible_fk AS tasks_responsible_fk
FROM tasks
WHERE %(param_1)s::INTEGER = tasks.responsible_fk
2025-10-23 12:41:46,823 INFO sqlalchemy.engine.Engine [cached since 0.003856s ago] {'param_1': 3}
    Задание 3
    Задание 4
pavel
2025-10-23 12:41:46,825 INFO sqlalchemy.engine.Engine SELECT tasks.id AS tasks_id, tasks.name AS tasks_name, tasks.responsible_fk AS tasks_responsible_fk
FROM tasks
WHERE %(param_1)s::INTEGER = tasks.responsible_fk
2025-10-23 12:41:46,825 INFO sqlalchemy.engine.Engine [cached since 0.005444s ago] {'param_1': 4}
    Задание 7
    Задание 8
2025-10-23 12:41:46,826 INFO sqlalchemy.engine.Engine COMMIT
```
В данном логе видно, что используя lazy load, для получения информации о 4 пользователях, мы сделали 5 запросов в БД: один запрос чтобы получить всех пользователей и по запросу на каждого пользователя для получения задач. Теперь можно поменять lazy="select" на lazy="selectin":

```
2025-10-23 12:45:12,891 INFO sqlalchemy.engine.Engine BEGIN (implicit)
2025-10-23 12:45:12,892 INFO sqlalchemy.engine.Engine SELECT users.id, users.name
FROM users
2025-10-23 12:45:12,892 INFO sqlalchemy.engine.Engine [generated in 0.00025s] {}
2025-10-23 12:45:12,894 INFO sqlalchemy.engine.Engine SELECT tasks.responsible_fk AS tasks_responsible_fk, tasks.id AS tasks_id, tasks.name AS tasks_name
FROM tasks
WHERE tasks.responsible_fk IN (%(primary_keys_1)s::INTEGER, %(primary_keys_2)s::INTEGER, %(primary_keys_3)s::INTEGER, %(primary_keys_4)s::INTEGER)
2025-10-23 12:45:12,895 INFO sqlalchemy.engine.Engine [generated in 0.00035s] {'primary_keys_1': 1, 'primary_keys_2': 2, 'primary_keys_3': 3, 'primary_keys_4': 4}
vladimir
    Задание 1
    Задание 2
maxim
    Задание 5
    Задание 6
nikita
    Задание 3
    Задание 4
pavel
    Задание 7
    Задание 8
2025-10-23 12:45:12,896 INFO sqlalchemy.engine.Engine COMMIT
```

Теперь, вместо 5 запросов мы видим всего 2.