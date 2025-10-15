Библиотека для работы с реляционными SQL базами данных, такими как [[PostgreSQL]] и т.п. Данная библиотека создает абстракцию над БД, благодаря которой, работа ведется не с сырыми запросами, а с [[Классы|Python-обьектами]]. Используется вместе с библиотеками, предоставляющими непосредственный доступ к Базе данных - [[asyncpg]]. [[psycopg]]. Также применяется месте с [[pydantic]] для валидации данных.

Ниже в статье будет описан пример работы
зависимости:
```python
from sqlalchemy.ext.asyncio import create_async_engine, async_session_maker, AsyncSession
from sqlalchemy.orm import Session, sessionmaker
from sqlalchemy import URL, create_engine, text
from config import settings
```


Для того, чтобы подключиться к базе данных, нужно начать с создания движка:
```python
engine = create_engine(
	url=settings.DATABASE_URL_psycopg,
	echo=True,
)
```