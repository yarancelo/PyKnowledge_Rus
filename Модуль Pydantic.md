Внешний [[Модули|модуль]] для языка Python. Представляет из себя набор инструментов для удобной валидации данных. Широко применяется в таких модулях как [[SQLAlchemy]], особенно при работе с [[SQLAlchemy ORM|ORM]], а также в [[Модуль FastAPI|FastAPI]].l

Pydantic расширяет функционал [[Data-классы|Data-классов]]. [[Классы]] в терминологии Pydantic называются схемами (schema) и для создании схемы ее нужно унаследовать от BaseModel:
```python
from pydantic import BaseModel

class ProductSchema(BaseModel):
  name: str
  quantity: int
  price: float

boots = ProductSchema(name="boots", quantity=40, price=54.99)
print(boots)
```
```
вывод:
name='boots' quantity=40 price=54.99
```

Объявленная таким образом схема автоматически генерирует ```__init__``` метод и переопределять его не нужно. В случае, если мы передадим в схему все значения в виде строк, то они будут валидированны и приведены к нужном типу:
```python
boots = ProductSchema(name="boots", quantity="40", price="54.99")
print(boots)
```
```
Вывод:
name='boots' quantity=40 price=54.99
```

То есть, значения количества и цены, переданные в качестве строки были приведены к целому числу и к цене автоматически.

## **Валидация полей с декораторами**
### 1) @field_validator
Декоратор ```field_validator``` служит для проверки корректности заполнения полей модели pydantic. Помимо валидации может быть использован для преобразования данных перед их сохранением в модели. Пример использования:
```python
  @field_validator("name", mode="before")
  def validate_name(cls, v):
    if isinstance(v, int):
      return str(v)
    elif isinstance(v, str):
      return v
    else:
      raise ValueError("Имя должно быть строкой или числом")
```

Декоратор принимает два параметра: название валидируемого поля и режим валидации. Режим валидации указывает на то, будет валидировано значение до создания объекта модели или после. 
Данный валидатор будет принимать имя только если оно является [[Строка|строкой]] или [[Целочисленный тип int|типом int]], иначе выбросит исключение.

### 2) @model_validator
Аналогичен @field_validator за исключением того, что позволяет валидировать значения атрибутов всей модели. Пример:
```python
@model_validator(mode='after')
def check_age(self):
    today = date.today()
    age = today.year - self.birth_date.year - (
      (today.month, today.day) < (self.birth_date.month, self.birth_date.day)
    )

    if age < 18:
      raise ValueError ("age undeer 18")
    if age > 120:
      raise ValueError ("age greater 120")
    return self
```

### 3) @computed_field
Данный декоратор позволяет создать динамически-вычисляемый атрибут. Пример:
```python
@computed_field
def full_name(self) -> str:
	return f"{self.name} {self.surname}"
	
print(user.full_name)
```

## **Валидация полей c Field()**
Pydanic предлагает широкий функционал для валидации полей, то есть, проверки, подходящие ли значения в них переданы или нет. 
```python
class SomeClass(BaseModel):
  id: int
  name: str | None # name: Optional[str]

  model_config = ConfigDict(extra="forbid")
```

В данной схеме имя может быть представлено строкой или отсутствовать, либо быть None. Параметр model_config принимает ConfigDict() и настройками. extra="forbid" означает, что схема не будет принимать словари, имеющие лишние аргументы:
```python
data_correct = {
  "id": 10,
  "name": "Hello"
}

data_incorrect = {
  "id": 10,
  "name": "Hello",
  "extra": "not"
}

object_1 = SomeClass(**data_correct)
object_2 = SomeClass(**data_incorrect)
print(object_1)
print(object_2)
```
```
Вывод:
 1 validation error for SomeClass extra
 Extra inputs are not permitted [type=extra_forbidden, input_value='not', input_type=str]
```

Также в Pydantic имеется возможность использовать более сложные валидаторы для полей с помощью функции Field:
```python
bio: str | None = Field(max_length=100)
```

В аргументы функции Field() передаются условия фильтрации, например - max_length=100 (максимальная длина строки 100 символов). Вообще Field позволяет добавлять к полям моделей метаданные и настройки, которые используются для валидации. На основе этих параметров работают некоторые другие фреймворки, например, [[Модуль FastAPI|FastAPI]]. Примеры того, что также можно передавать в Field:
* default - устанавливает значение по умолчанию
* default-factory - функция, возвращающая значение по умолчанию
* alias - альтернативное поле для сериализации и десериализации.
* title - заголовок поля для документации.
* description - описание поля для документации.
* exclude - исключает поле из сериализации.
* repr - определяет, будет ли поле включено в строковое представление модели.

Валидация через Field():
```python
class Product(BaseModel):   
  price: float = Field(gt=0, description="Цена должна быть больше нуля")    
  name: str = Field(min_length=2, max_length=50, description="Название продукта должно быть от 2 до 50 символов")  
```

* gt, ge, lt, le - Greater than (Field(gt=0) - значение поля должно быть больше нуля). Greter or equal (Field(ge=1) - значение поля должно быть больше либо равным единице). Less than и Less or Equal по смыслу аналогичны.
* min_length, max_length - задает минимальную и максимальную длину поля.
* regex - регулярные выражения

Дополнительные примеры:
#### 1) Пример использования regex
```python
class User(BaseModel):    
	email: str = Field(regex=r"[^@]+@[^@]+\.[^@]+", 
	description="Электронная почта должна быть в корректном формате")    
	phone_number: str = Field(regex=r"^\+\d{1,3}\s?\d{4,14}$", description="Номер телефона должен быть в формате +123456789")
```

Данные регулярные выражения позволяют валидировать номера телефонов и адреса почт. Однако, регулярные выражения имеют очень высокую когнитивную сложность, и для валидации адресов почт можно использовать специальный тип EmailStr:
```python
class UserSchema(BaseModel):
  email: EmailStr
  bio: str | None = Field(max_length=100)
  model_config = ConfigDict(extra="forbid")
```

#### 2) Пример получения динамического значения с default factory:
Иногда требуется генерировать значения полей динамически. Для этого используется параметр default_factory, который принимает функцию для генерации значения.
```python
from uuid import uuid4
class Item(BaseModel):    
	id: str = Field(default_factory=lambda: uuid4().hex)
```

В данном примере, каждое поле id будет получать уникальный идентификатор при создании нового объекта модели Item.

#### 3) Пример с alias
При использовании alias, это поле будет иметь одно имя в вашем python коде и другое имя при сериализации, например, в json.
```python
class User(BaseModel):    
	username: str = Field(alias="user_name")
```

В данном примере, поле username будет сериализоваться под именем user_name.

## **Декларативные аннотации с Annotated**
Возможно использовать Annotated тогда, когда хочется держать тип и метаданные в одной аннотации. Аннотирование именно таким образом считается [[Хорошие практики|хорошей практикой]] в python. Также Annotated помогает избежать конфликтов при наследовании, улучшает подсвеичваемость кода в IDE, поддерживает декларативный стиль написания кода, поддерживает написание нескольких валидаторов через цепочку.
Синтаксис:
```python
from typing import Annotated
from pydantic import BaseModel, Field

class Product(BaseModel):
	name: Annotated[str, Field(min_length=2, max_length=30)]
	price: Annotated[float, Float(gt=0)]
```

## **Конфигурация моделей**
Конфигурация позволяет применять общие свойства к всем полям модели
```python
from pydantic import BaseModel, ConfigDict

class MyModel(BaseModel):    
	model_config = ConfigDict(from_attributes=True)
```

Основные опции:
* from_attributes=True - позволяет создавать объект модели напрямую из атрибутов [[Классы|Python-объектов]] (например, когда поля модели совпадают с атрибутами другого объекта). Чаще всего применяется для преобразования моделей ORM к моделям Pydantic.
* str_to_lower, str_to_upper - преобразование всех строк модели в нижний или верхний регистр.
* str_strip_whitespace - следует ли удалять начальные и конечные пробелы для типов str (аналог [[Строка|метода strip у строки]]). 
* str_min_length, str_max_length - задает минимальную и максимальную длину строки для всех [[Строка|строковых]] полей.
* use_enum_values - следует ли заполнять модели значениями, выбранными из перечислений, вместо того, чтобы использовать необработанные значения. Часто требуется при работе с моделями ORM в которых колонки определены как перечисления ([[enum]]).

config_dict делает конфигурацию моделей более явной, удобной и гибкой, позволяет определять параметры моделей напрямую через словарь, избегая необходимости создания вложенных классов, что упрощает процесс настройки и делает его более интуитивно понятным. Это особенно полезно при работе с большими и сложными моделями данных, где требуется высокая степень кастомизации.

## **Связка Pydantic и [[SQLAlchemy]]**
При работе с SQLAlchemy в ORM стиле мы сталкиваемся с тем, что данные возвращаются в виде объектов моделей таблиц, что не всегда удобно для дальнейшей обработки. Разработчики обычно предпочитают работать с данными в формате [[Модуль json|JSON]] или Python-[[Словарь|словарей]]. 
Принцип интегрирования [[SQLAlchemy ORM]] и Pydantic:
1) Описание модели таблицы в [[SQLAlchemy]]
2) Создание Pydantic-модели для работы с полученными данными
3) Запрос данных из [[PostgreSQL|БД]]
4) Преобразование объекта таблицы [[SQLAlchemy ORM]] к Pydantic-объекту
5) Использование методов model_dump или model_dump_json для получения данных в нужном формате.

В новых версиях Pydantic, разработчики рекомендуют использовать следующий стиль:
```python
def get_user(user_id: int):    
	with Session() as session:        
		user = session.select(UserORM).filter_by(id = user_id).scalar_one_or_none()                
		# Шаг 4: Преобразование объекта SQLAlchemy в Pydantic        
		if user:            
						user_pydantic = UserPydantic.model_validate(user)                        # Шаг 5: Получение данных в нужном формате            
			return user_pydantic.model_dump()        
		return None
```

