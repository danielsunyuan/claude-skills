---
name: flask-patterns
description: Flask application structure, Blueprint organization, and Python backend patterns. Use when building Flask apps, organizing Python web projects, or implementing backend service architecture.
allowed-tools: Read, Grep, Glob
---

# Flask Backend Patterns

Quick reference for Flask application structure and Python backend patterns.

## Project Structure

```
app/
├── api/
│   ├── __init__.py
│   ├── users.py
│   ├── auth.py
│   └── resources.py
├── models/
│   ├── __init__.py
│   ├── user.py
│   └── resource.py
├── services/
│   ├── __init__.py
│   ├── user_service.py
│   └── resource_service.py
└── schemas/
    ├── __init__.py
    └── user_schema.py
```

## Blueprint Organization

```python
# app/api/__init__.py
from flask import Blueprint

api = Blueprint('api', __name__, url_prefix='/api/v1')

from . import users, auth, resources
```

```python
# app/api/users.py
from flask import jsonify, request
from . import api
from app.services.user_service import UserService

@api.route('/users', methods=['GET'])
def list_users():
    users = UserService.get_all()
    return jsonify([u.to_dict() for u in users])

@api.route('/users/<int:user_id>', methods=['GET'])
def get_user(user_id):
    user = UserService.get_by_id(user_id)
    if not user:
        return jsonify({"error": "Not found"}), 404
    return jsonify(user.to_dict())
```

## Service Layer Pattern

```python
# app/services/user_service.py
from typing import List, Optional
from sqlalchemy.orm import Session
from app.models.user import User

class UserService:
    def __init__(self, db: Session):
        self.db = db

    def get_by_id(self, user_id: int) -> Optional[User]:
        return self.db.query(User).filter(User.id == user_id).first()

    def get_all(self) -> List[User]:
        return self.db.query(User).all()

    def create(self, user_data: dict) -> User:
        user = User(**user_data)
        self.db.add(user)
        self.db.commit()
        self.db.refresh(user)
        return user

    def update(self, user_id: int, user_data: dict) -> Optional[User]:
        user = self.get_by_id(user_id)
        if not user:
            return None
        for key, value in user_data.items():
            setattr(user, key, value)
        self.db.commit()
        return user
```

## Dependency Injection

```python
from functools import lru_cache
from typing import Generator

@lru_cache()
def get_settings() -> Settings:
    return Settings()

def get_db() -> Generator:
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()
```

## Error Handling

```python
from flask import jsonify
from werkzeug.exceptions import HTTPException

class APIError(Exception):
    def __init__(self, message: str, status_code: int = 400, details: dict = None):
        self.message = message
        self.status_code = status_code
        self.details = details or {}

@app.errorhandler(APIError)
def handle_api_error(error):
    return jsonify({
        "error": error.message,
        "details": error.details
    }), error.status_code

@app.errorhandler(HTTPException)
def handle_http_error(error):
    return jsonify({
        "error": error.description
    }), error.code
```

## Request Validation (Pydantic)

```python
from pydantic import BaseModel, EmailStr, validator
from typing import Optional

class UserCreate(BaseModel):
    email: EmailStr
    username: str
    password: str

    @validator('username')
    def username_valid(cls, v):
        if len(v) < 3:
            raise ValueError('Username must be at least 3 characters')
        return v

class UserUpdate(BaseModel):
    email: Optional[EmailStr] = None
    username: Optional[str] = None
```

## Configuration Pattern

```python
import os

class Config:
    SECRET_KEY = os.getenv('SECRET_KEY', 'dev-key-change-me')
    SQLALCHEMY_DATABASE_URI = os.getenv('DATABASE_URL')
    SQLALCHEMY_TRACK_MODIFICATIONS = False

class DevelopmentConfig(Config):
    DEBUG = True

class ProductionConfig(Config):
    DEBUG = False

config = {
    'development': DevelopmentConfig,
    'production': ProductionConfig,
    'default': DevelopmentConfig
}
```

## Application Factory

```python
from flask import Flask
from flask_sqlalchemy import SQLAlchemy

db = SQLAlchemy()

def create_app(config_name='default'):
    app = Flask(__name__)
    app.config.from_object(config[config_name])

    db.init_app(app)

    from app.api import api as api_blueprint
    app.register_blueprint(api_blueprint)

    return app
```
