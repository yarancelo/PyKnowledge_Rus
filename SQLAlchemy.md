Библиотека для работы с реляционными SQL базами данных, такими как [[PostgreSQL]] и т.п. Данная библиотека создает абстракцию над БД, благодаря которой, можно работать и с сырыми запросами, и с [[Классы|Python-обьектами]]. Используется вместе с библиотеками, предоставляющими непосредственный доступ к Базе данных - [[asyncpg]]. [[psycopg]]. Также применяется месте с [[Pydantic]] для валидации данных.

Работать с sqlalchemy возможно в двух режимах: **core** и **ORM**. В режиме **core** осуществляется работа с сырыми запросами и запросами из "отформатированных" данных. В **ORM** работа происходит с абстракцией над базами данных, предоставляющими доступ к данным не через "запросы", а через классы, объекты и атрибуты.

Вот в этой статье описана [[SQLAlchemy Core|работа в Core режиме]], а вот в этой, [[SQLAlchemy ORM|работа в ORM режиме]].
Понятия, общие для обоих "режимов работы":

* ## **Двигатель**
Для того, чтобы подключиться к базе данных, нужен двигатель, двигатель создается следующим образом:
```python
from sqlalchemy import create_engine
from config import settings

engine = create_engine(
  url=settings.DATABASE_URL,
  echo=True
)
```

Из sqlalchemy импортируется функция create_engine(), являющаяся [[Фабрика|фабрикой]] двигателей.
Для того, чтобы передать настройки в двигатель, они импортируются из config.settings. Параметр echo=True, означает, что вся работа с базой данных будет [[Логгирование|логироваться]].

* ## **Настройки для двигателя**
Для подключения в БД, двигателю необходимо знать такие параметры как адрес хоста БД, порт БД, имя БД, имя пользователя, через которого осуществляется работа с БД и пароль данного пользователя в БД. Все эти параметры должны передаваться в двигатель с помощью сформатированной строки. Форматированием строки занимается класс Settings:
```Python
from pydantic_settings import BaseSettings, SettingsConfigDict

class Settings(BaseSettings):
  DB_HOST: str
  DB_PORT: str
  DB_NAME: str
  DB_USER: str
  DB_PASS: str

  @property
  def DATABASE_URL(self):
    return f"postgresql+psycopg://{self.DB_USER}:{self.DB_PASS}@{self.DB_HOST}:{self.DB_PORT}/{self.DB_NAME}"
    
  model_config = SettingsConfigDict(env_file=".env")

settings = Settings()
```

Данный двигатель использует драйвер для работы с БД [[psycopg]], который является еще более низкоуровневой библиотекой для работы с Postgres, помимо [[psycopg]], можно использовать и [[asyncpg]] для работы с БД Postgres асинхронно. Непосредственно сами параметры хранятся в переменных окружения и подтягиваются с помощью [[Pydantic]]:
```
DB_HOST=172.27.152.86
DB_PORT=5432
DB_NAME=mydb
DB_USER=userdb
DB_PASS=password
```

В данном примере адрес хоста задан как IP-адрес, но также это может быть и доменный адрес.

* ## **Метаданные**
Метаданными в SQLAlchemy является информация о таблицах внутри БД. В случае с **core** режимом, метаданные нужно создать и передать в таблицу при ее создании, пример:
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
```


