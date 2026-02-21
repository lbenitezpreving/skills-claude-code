# Template: Configuracion de Entorno

## .env

```env
# PostgreSQL
DATABASE_URL=postgresql+asyncpg://postgres:password@localhost:5432/myapp
DATABASE_POOL_SIZE=5
DATABASE_MAX_OVERFLOW=10

# App
APP_ENV=development
SECRET_KEY=cambia-esto-en-produccion
```

## .env.test

```env
DATABASE_URL=postgresql+asyncpg://postgres:password@localhost:5432/myapp_test
```

## .gitignore (añadir si no existe)

```gitignore
.env
.env.test
*.db
```

## src/api/config.py

```python
from pydantic_settings import BaseSettings, SettingsConfigDict


class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env", extra="ignore")

    database_url: str
    database_pool_size: int = 5
    database_max_overflow: int = 10
    app_env: str = "development"
    secret_key: str = "dev-secret"

    @property
    def is_development(self) -> bool:
        return self.app_env == "development"


settings = Settings()
```

> Instalar: `pip install pydantic-settings`
