---
name: sqlalchemy-patterns
description: SQLAlchemy ORM patterns, relationships, and database best practices. Use when working with SQLAlchemy models, database queries, or designing database schemas.
allowed-tools: Read, Grep, Glob
---

# SQLAlchemy Patterns

Quick reference for SQLAlchemy ORM usage and database patterns.

## Model Definition

```python
from sqlalchemy import Column, Integer, String, DateTime, Boolean, ForeignKey
from sqlalchemy.orm import relationship, declarative_base
from sqlalchemy.sql import func

Base = declarative_base()

class User(Base):
    __tablename__ = 'users'

    id = Column(Integer, primary_key=True, index=True)
    email = Column(String(255), unique=True, nullable=False, index=True)
    username = Column(String(100), unique=True, nullable=False)
    hashed_password = Column(String(255), nullable=False)
    is_active = Column(Boolean, default=True)
    created_at = Column(DateTime(timezone=True), server_default=func.now())
    updated_at = Column(DateTime(timezone=True), onupdate=func.now())

    # Relationships
    orders = relationship("Order", back_populates="user")

    def __repr__(self):
        return f"<User(id={self.id}, email={self.email})>"
```

## Relationships

### One-to-Many
```python
class User(Base):
    __tablename__ = 'users'
    id = Column(Integer, primary_key=True)
    orders = relationship("Order", back_populates="user")

class Order(Base):
    __tablename__ = 'orders'
    id = Column(Integer, primary_key=True)
    user_id = Column(Integer, ForeignKey('users.id'), nullable=False)
    user = relationship("User", back_populates="orders")
```

### Many-to-Many
```python
# Association table
user_roles = Table(
    'user_roles',
    Base.metadata,
    Column('user_id', Integer, ForeignKey('users.id'), primary_key=True),
    Column('role_id', Integer, ForeignKey('roles.id'), primary_key=True)
)

class User(Base):
    __tablename__ = 'users'
    id = Column(Integer, primary_key=True)
    roles = relationship("Role", secondary=user_roles, back_populates="users")

class Role(Base):
    __tablename__ = 'roles'
    id = Column(Integer, primary_key=True)
    name = Column(String(50), unique=True)
    users = relationship("User", secondary=user_roles, back_populates="roles")
```

### One-to-One
```python
class User(Base):
    __tablename__ = 'users'
    id = Column(Integer, primary_key=True)
    profile = relationship("Profile", back_populates="user", uselist=False)

class Profile(Base):
    __tablename__ = 'profiles'
    id = Column(Integer, primary_key=True)
    user_id = Column(Integer, ForeignKey('users.id'), unique=True)
    user = relationship("User", back_populates="profile")
```

## Loading Strategies

```python
from sqlalchemy.orm import joinedload, selectinload, lazyload

# Eager load (one query with JOIN)
users = db.query(User).options(joinedload(User.orders)).all()

# Eager load (separate SELECT IN query) - better for collections
users = db.query(User).options(selectinload(User.orders)).all()

# Lazy load (default, N+1 problem)
users = db.query(User).all()
for user in users:
    print(user.orders)  # Separate query for each user!
```

## Common Queries

### Basic CRUD
```python
# Create
user = User(email="test@example.com", username="test")
db.add(user)
db.commit()
db.refresh(user)

# Read
user = db.query(User).filter(User.id == 1).first()
users = db.query(User).all()

# Update
user.email = "new@example.com"
db.commit()

# Delete
db.delete(user)
db.commit()
```

### Filtering
```python
# Equals
db.query(User).filter(User.email == "test@example.com")

# Not equals
db.query(User).filter(User.email != "test@example.com")

# Like
db.query(User).filter(User.email.like("%@gmail.com"))
db.query(User).filter(User.email.ilike("%@GMAIL.com"))  # case-insensitive

# In
db.query(User).filter(User.id.in_([1, 2, 3]))

# Between
db.query(User).filter(User.created_at.between(start_date, end_date))

# IS NULL / IS NOT NULL
db.query(User).filter(User.deleted_at.is_(None))
db.query(User).filter(User.deleted_at.isnot(None))

# AND / OR
from sqlalchemy import and_, or_
db.query(User).filter(and_(User.is_active == True, User.role == "admin"))
db.query(User).filter(or_(User.role == "admin", User.role == "moderator"))
```

### Ordering & Limiting
```python
# Order by
db.query(User).order_by(User.created_at.desc())
db.query(User).order_by(User.username.asc())

# Limit & Offset
db.query(User).limit(10).offset(20)

# First / One
user = db.query(User).first()  # Returns None if not found
user = db.query(User).one()    # Raises if not exactly one
user = db.query(User).one_or_none()  # Returns None or raises if multiple
```

### Aggregations
```python
from sqlalchemy import func

# Count
count = db.query(func.count(User.id)).scalar()

# Sum, Avg, Min, Max
total = db.query(func.sum(Order.amount)).scalar()
avg = db.query(func.avg(Order.amount)).scalar()

# Group by
from sqlalchemy import func
results = db.query(
    User.role,
    func.count(User.id)
).group_by(User.role).all()
```

### Joins
```python
# Implicit join (via relationship)
db.query(User).join(User.orders).filter(Order.status == "completed")

# Explicit join
db.query(User, Order).join(Order, User.id == Order.user_id)

# Left outer join
db.query(User).outerjoin(User.orders)
```

## Session Management

```python
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

engine = create_engine(DATABASE_URL, pool_pre_ping=True)
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)

# Context manager pattern
def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()

# Usage in Flask
@app.route('/users')
def list_users():
    with SessionLocal() as db:
        return db.query(User).all()
```

## Transactions

```python
# Automatic (default)
db.add(user)
db.commit()  # or db.rollback()

# Manual control
try:
    db.begin()
    db.add(user)
    db.add(order)
    db.commit()
except Exception:
    db.rollback()
    raise
```

## Soft Deletes

```python
class SoftDeleteMixin:
    deleted_at = Column(DateTime(timezone=True), nullable=True)

    def soft_delete(self):
        self.deleted_at = func.now()

class User(Base, SoftDeleteMixin):
    __tablename__ = 'users'
    # ...

# Query non-deleted only
db.query(User).filter(User.deleted_at.is_(None))
```

## Timestamps Mixin

```python
class TimestampMixin:
    created_at = Column(DateTime(timezone=True), server_default=func.now())
    updated_at = Column(DateTime(timezone=True), onupdate=func.now())

class User(Base, TimestampMixin):
    __tablename__ = 'users'
    # ...
```

## Index Best Practices

```python
from sqlalchemy import Index

class User(Base):
    __tablename__ = 'users'

    id = Column(Integer, primary_key=True)
    email = Column(String(255), index=True)  # Single column
    first_name = Column(String(100))
    last_name = Column(String(100))

    # Composite index
    __table_args__ = (
        Index('ix_user_name', 'first_name', 'last_name'),
    )
```

## N+1 Prevention

```python
# BAD - N+1 queries
users = db.query(User).all()
for user in users:
    print(user.orders)  # Query per user!

# GOOD - Eager loading
users = db.query(User).options(selectinload(User.orders)).all()
for user in users:
    print(user.orders)  # No additional queries
```
