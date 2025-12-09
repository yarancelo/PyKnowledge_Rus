Чтобы создать таблицу в базе данных [[PostgreSQL]], для начала нужно импортировать нужные типы для SQL таблиц:
```python
from sqlalchemy import Table, Column, Integer, String, MetaData

metadata_obj = MetaData()
```

Объект metadata_obj будет содержать данные обо всех созданных нами таблицах.
```python
workers_table = Table(
	"workers", # Название таблицы
	metadata_obj, # Объект метадаты для записи информации
	
	Column("id", Integer, primary_key=True), # Параметр первичного ключа нужно указать хотя бы одному из стобцов
	Column("username", String),
)

def create_tables():
	metadata_obj.drop_all(sync_engine)
	metadata_obj.create_all(sync_endgine)
```

Сначала мы объявили таблицу, а потом, с помощью функции create_tables, создали таблицы, при этом, удалив предыдущие, если они были созданы.


