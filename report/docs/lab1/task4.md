# Connection


``` py title="connection.py"
from sqlmodel import SQLModel, Session, create_engine
import os
from dotenv import load_dotenv
from models import *

load_dotenv()

db_url = os.getenv("DB_ADMIN")
engine = create_engine(db_url, echo=True)


def init_db():
    SQLModel.metadata.create_all(engine)


def get_session():
    with Session(engine) as session:
        yield session

```

### Alembic

Для начала работы к проекту была подключена система миграции Alembic. Все ее компоненты хранятся в папке `migrations` + в файле `alembic.ini`. Вся информация о базе данных хранится локально и подтягивается системой миграции динамически из файла `.env`.