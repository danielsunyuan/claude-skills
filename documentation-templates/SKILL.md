---
name: documentation-templates
description: Documentation templates for README, API docs, docstrings, and architecture docs. Use when writing documentation, creating READMEs, or documenting code.
allowed-tools: Read, Grep, Glob
---

# Documentation Templates

Quick reference templates for common documentation needs.

## README Template

```markdown
# Project Name

Brief one-sentence description.

## Overview

2-3 paragraphs explaining what this project does and why.

## Features

- Feature 1: Description
- Feature 2: Description

## Quick Start

\`\`\`bash
pip install project-name
python -m project_name --help
\`\`\`

## Installation

### Prerequisites

- Python 3.11+
- PostgreSQL 13+

### Setup

1. Clone: `git clone https://github.com/org/project.git`
2. Install: `pip install -r requirements.txt`
3. Configure: `cp .env.example .env`
4. Run: `python -m project_name`

## Usage

\`\`\`python
from project import Client
client = Client(api_key="your-key")
result = client.do_something()
\`\`\`

## Configuration

| Variable | Description | Default |
|----------|-------------|---------|
| API_KEY | Your API key | required |
| DEBUG | Debug mode | false |

## Development

\`\`\`bash
pytest tests/
\`\`\`

## License

MIT - See [LICENSE](LICENSE)
```

## Python Docstring (Google Style)

```python
def process_data(data: List[dict], filter_fn: Optional[Callable] = None) -> List[dict]:
    """Process a list of data dictionaries with optional filtering.

    This function processes each item, optionally applying a filter.

    Args:
        data: List of dictionaries to process. Each must have 'id' field.
        filter_fn: Optional callable returning bool. If provided, only
            items where filter_fn returns True are included.

    Returns:
        List of processed dictionaries with 'processed' field added.

    Raises:
        ValueError: If any item is missing 'id' field.
        TypeError: If data is not a list.

    Example:
        >>> data = [{'id': 1, 'value': 10}]
        >>> process_data(data)
        [{'id': 1, 'value': 10, 'processed': True}]
    """
```

## Class Docstring

```python
class DataProcessor:
    """A processor for transforming and enriching data.

    Provides methods for processing data with configurable validation
    and transformation pipelines.

    Attributes:
        config: Configuration dictionary with processing parameters.
        validators: List of validation functions.

    Example:
        >>> processor = DataProcessor(config={'strict': True})
        >>> result = processor.process([{'id': 1}])
    """

    def __init__(self, config: dict):
        """Initialize the DataProcessor.

        Args:
            config: Configuration dictionary. Supported keys:
                - strict (bool): Enable strict validation
                - max_items (int): Max items per batch
                - timeout (int): Timeout in seconds
        """
```

## API Endpoint Documentation

```python
@app.route('/api/users', methods=['POST'])
def create_user():
    """Create a new user.

    ---
    tags:
      - Users
    requestBody:
      required: true
      content:
        application/json:
          schema:
            type: object
            required: [email, username]
            properties:
              email:
                type: string
                format: email
              username:
                type: string
                minLength: 3
    responses:
      201:
        description: User created successfully
      400:
        description: Validation error
      409:
        description: Email already exists
    """
```

## Environment Variable Reference

```markdown
## Environment Variables

### Required

#### `DATABASE_URL`
- **Type**: string
- **Format**: `postgresql://user:pass@host:port/db`
- **Example**: `postgresql://app:secret@localhost:5432/myapp`

### Optional

#### `LOG_LEVEL`
- **Type**: string
- **Values**: `DEBUG`, `INFO`, `WARNING`, `ERROR`
- **Default**: `INFO`
```

## Changelog Entry

```markdown
## [1.2.0] - 2024-01-15

### Added
- OAuth2 authentication support (#123)
- User preference settings (#124)

### Changed
- Improved dashboard loading time (#125)

### Fixed
- Login redirect issue on mobile (#126)

### Deprecated
- Legacy API v1 endpoints (removal in v2.0)

### Security
- Updated dependencies for CVE-2024-XXXX
```

## Error Documentation

```markdown
## Error: "Connection refused"

**Cause**: Database is not running or not accessible.

**Solution**:
1. Check database is running: `docker ps | grep postgres`
2. Verify connection settings in `.env`
3. Check database logs: `docker logs db`

**Related**: [Database Configuration](config.md#database)
```

## Architecture Decision Record (ADR)

```markdown
# ADR-001: Use PostgreSQL for Primary Database

## Status
Accepted

## Context
We need a reliable database for storing user data and application state.

## Decision
Use PostgreSQL as the primary database.

## Consequences

### Positive
- ACID compliance
- Strong ecosystem
- JSON support for flexible schemas

### Negative
- More complex than SQLite
- Requires separate service
```

## How-To Guide Structure

```markdown
# How to Add a New Feature

This guide walks through adding [feature] to the project.

## Prerequisites

- Development environment set up
- Basic understanding of [technology]

## Steps

### 1. Create the Component

\`\`\`python
# src/components/my_feature.py
class MyFeature:
    pass
\`\`\`

### 2. Register It

Add to registry in `src/__init__.py`

### 3. Test It

\`\`\`bash
pytest tests/test_my_feature.py
\`\`\`

## Troubleshooting

### Feature not appearing
Check that it's registered in the registry.

## Next Steps
- Add configuration options
- Write documentation
```

## Writing Tips

### Style
- **Active voice**: "The system processes" not "is processed"
- **Present tense**: "returns" not "will return"
- **Short sentences**: One idea per sentence
- **Simple words**: "use" not "utilize"

### Structure
- Overview → Details → Advanced
- Use headers, lists, code blocks
- Include working examples
- Link related topics

### Code Examples
- Complete and runnable
- Realistic scenarios
- Commented where needed
- Actually tested
