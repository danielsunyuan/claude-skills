---
name: pytest-patterns
description: Pytest fixtures, parametrization, mocking patterns, and test organization. Use when writing tests, setting up fixtures, or improving test coverage.
allowed-tools: Read, Grep, Glob
---

# Pytest Patterns

Quick reference for pytest testing patterns and best practices.

## Test Structure (AAA)

```python
def test_create_user_with_valid_data():
    # Arrange
    user_data = {"username": "test", "email": "test@example.com"}

    # Act
    user = create_user(user_data)

    # Assert
    assert user.username == "test"
    assert user.id is not None
```

## Fixtures

### Basic Fixture

```python
# conftest.py
import pytest

@pytest.fixture
def sample_user():
    return User(username="testuser", email="test@example.com")

# test_users.py
def test_user_display_name(sample_user):
    assert sample_user.display_name == "testuser"
```

### Fixture Scopes

```python
@pytest.fixture(scope="function")  # Default: new for each test
@pytest.fixture(scope="class")     # Once per test class
@pytest.fixture(scope="module")    # Once per module
@pytest.fixture(scope="session")   # Once per test session
```

### Database Session Fixture

```python
@pytest.fixture(scope="function")
def db_session(engine):
    connection = engine.connect()
    transaction = connection.begin()
    Session = sessionmaker(bind=connection)
    session = Session()

    yield session

    session.close()
    transaction.rollback()
    connection.close()
```

### Factory Fixture

```python
@pytest.fixture
def user_factory(db_session):
    def _create_user(**kwargs):
        defaults = {"username": "test", "email": "test@example.com"}
        defaults.update(kwargs)
        user = User(**defaults)
        db_session.add(user)
        db_session.commit()
        return user
    return _create_user

def test_multiple_users(user_factory):
    user1 = user_factory(username="alice")
    user2 = user_factory(username="bob")
    assert user1.id != user2.id
```

## Parametrization

### Basic

```python
@pytest.mark.parametrize("input,expected", [
    ("valid@email.com", True),
    ("invalid.email", False),
    ("no@domain", False),
])
def test_email_validation(input, expected):
    assert is_valid_email(input) == expected
```

### Multiple Parameters

```python
@pytest.mark.parametrize("x", [1, 2])
@pytest.mark.parametrize("y", [10, 20])
def test_multiply(x, y):
    # Runs 4 tests: (1,10), (1,20), (2,10), (2,20)
    assert multiply(x, y) == x * y
```

### IDs for Clarity

```python
@pytest.mark.parametrize("email,valid", [
    pytest.param("user@example.com", True, id="valid-email"),
    pytest.param("invalid", False, id="no-at-sign"),
    pytest.param("@example.com", False, id="no-local-part"),
])
def test_email_validation(email, valid):
    assert is_valid_email(email) == valid
```

## Mocking

### Basic Mock

```python
from unittest.mock import Mock, patch

def test_user_creation_sends_email():
    with patch('app.services.email.send_email') as mock_send:
        create_user({"email": "test@example.com"})

        mock_send.assert_called_once_with(
            to="test@example.com",
            subject="Welcome!"
        )
```

### Mock Return Values

```python
@patch('app.services.api.fetch_data')
def test_process_data(mock_fetch):
    mock_fetch.return_value = {"status": "ok", "data": [1, 2, 3]}

    result = process_external_data()

    assert result == [1, 2, 3]
```

### Mock Side Effects

```python
@patch('app.services.api.fetch_data')
def test_retry_on_failure(mock_fetch):
    mock_fetch.side_effect = [
        ConnectionError("Failed"),
        {"status": "ok"}
    ]

    result = fetch_with_retry()
    assert mock_fetch.call_count == 2
```

### pytest-mock

```python
def test_with_mocker(mocker):
    mock_send = mocker.patch('app.email.send')
    mock_send.return_value = True

    result = notify_user("test@example.com")

    assert result is True
    mock_send.assert_called_once()
```

## API Testing

```python
@pytest.fixture
def client(app):
    return app.test_client()

def test_create_user_endpoint(client, auth_headers):
    response = client.post(
        "/api/users",
        json={"username": "test", "email": "test@example.com"},
        headers=auth_headers
    )

    assert response.status_code == 201
    data = response.get_json()
    assert data["username"] == "test"
```

## Async Testing

```python
import pytest

@pytest.mark.asyncio
async def test_async_operation():
    result = await async_fetch_data()
    assert result is not None
```

## Markers

```python
# Mark slow tests
@pytest.mark.slow
def test_slow_operation():
    pass

# Skip test
@pytest.mark.skip(reason="Not implemented yet")
def test_future_feature():
    pass

# Skip conditionally
@pytest.mark.skipif(sys.version_info < (3, 10), reason="Requires Python 3.10+")
def test_new_syntax():
    pass

# Expected failure
@pytest.mark.xfail(reason="Known bug #123")
def test_buggy_feature():
    pass
```

## pytest.ini Configuration

```ini
[pytest]
testpaths = tests
python_files = test_*.py
python_classes = Test*
python_functions = test_*
addopts =
    --verbose
    --strict-markers
    --cov=app
    --cov-report=term-missing
    --cov-fail-under=80
markers =
    slow: marks tests as slow
    integration: integration tests
    unit: unit tests
```

## Test Organization

```
tests/
├── conftest.py           # Shared fixtures
├── unit/
│   ├── test_models.py
│   ├── test_services.py
│   └── test_utils.py
├── integration/
│   ├── test_api.py
│   └── test_database.py
└── fixtures/
    └── data.py
```

## Running Tests

```bash
# Run all tests
pytest

# Verbose output
pytest -v

# Run specific file
pytest tests/test_users.py

# Run specific test
pytest tests/test_users.py::test_create_user

# Run by marker
pytest -m "not slow"
pytest -m integration

# Stop on first failure
pytest -x

# Show print statements
pytest -s

# Parallel execution
pytest -n auto

# Coverage report
pytest --cov=app --cov-report=html
```

## Debugging Tests

```bash
# Drop into debugger on failure
pytest --pdb

# Drop into debugger at start
pytest --pdb-first

# Show local variables
pytest -l

# Increase verbosity
pytest -vv
```

## Common Patterns

### Test Naming

```python
# Pattern: test_<what>_<condition>_<expected>
def test_create_user_with_duplicate_email_raises_error():
    pass

def test_login_with_invalid_password_returns_401():
    pass
```

### Exception Testing

```python
def test_invalid_email_raises_error():
    with pytest.raises(ValueError) as exc_info:
        validate_email("invalid")

    assert "Invalid email" in str(exc_info.value)
```

### Approximate Comparisons

```python
def test_calculation():
    result = calculate_percentage(1, 3)
    assert result == pytest.approx(0.333, rel=1e-2)
```
