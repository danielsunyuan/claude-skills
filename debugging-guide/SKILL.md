---
name: debugging-guide
description: Python debugging techniques, common bug patterns, and troubleshooting strategies. Use when investigating bugs, analyzing errors, or debugging issues.
allowed-tools: Read, Grep, Glob, Bash
---

# Debugging Guide

Quick reference for Python debugging techniques and common bug patterns.

## Debugging Process

1. **Reproduce** - Can you trigger it consistently?
2. **Isolate** - What's the minimal reproduction?
3. **Hypothesize** - What could cause this?
4. **Test** - Verify one hypothesis at a time
5. **Fix** - Address root cause, not symptoms

## Python Debugger (pdb)

```python
import pdb; pdb.set_trace()  # Breakpoint

# Or use breakpoint() in Python 3.7+
breakpoint()
```

### pdb Commands

| Command | Action |
|---------|--------|
| `n` | Next line |
| `s` | Step into function |
| `c` | Continue execution |
| `l` | List code around current line |
| `p var` | Print variable |
| `pp var` | Pretty print |
| `w` | Show stack trace |
| `u` | Up one frame |
| `d` | Down one frame |
| `q` | Quit debugger |

## Strategic Logging

```python
import logging

logger = logging.getLogger(__name__)

def complex_operation(user_id, data):
    logger.debug(f"Starting operation for user {user_id}")
    logger.debug(f"Input: {data}")

    try:
        result = process(data)
        logger.info(f"Success for user {user_id}")
        return result
    except Exception as e:
        logger.error(f"Failed for user {user_id}", exc_info=True)
        raise
```

### Quick Debug Logging

```python
# Enable debug logging temporarily
import logging
logging.basicConfig(level=logging.DEBUG)
```

## Common Bug Patterns

### 1. Mutable Default Arguments

```python
# BUG: List shared across calls
def add_item(item, items=[]):
    items.append(item)
    return items

# FIX
def add_item(item, items=None):
    if items is None:
        items = []
    items.append(item)
    return items
```

### 2. Reference vs Copy

```python
# BUG: Modifies original
def process(data):
    data.sort()  # In-place!
    return data

# FIX
def process(data):
    return sorted(data)  # New list
```

### 3. Late Binding Closures

```python
# BUG: All functions return 4
funcs = [lambda: i for i in range(5)]

# FIX: Capture value
funcs = [lambda i=i: i for i in range(5)]
```

### 4. Silent Exception Swallowing

```python
# BUG: Hides errors
try:
    risky_operation()
except:
    pass

# FIX: Be specific
try:
    risky_operation()
except ValueError as e:
    logger.warning(f"Invalid value: {e}")
```

### 5. Off-by-One Errors

```python
# BUG: Misses last element
for i in range(len(items) - 1):
    process(items[i])

# FIX
for item in items:
    process(item)
```

### 6. Resource Leaks

```python
# BUG: File might not close
f = open("file.txt")
data = f.read()
f.close()

# FIX: Context manager
with open("file.txt") as f:
    data = f.read()
```

### 7. Race Conditions

```python
# BUG: Check-then-act race
if not os.path.exists(path):
    with open(path, 'w') as f:
        f.write(data)

# FIX: Atomic operation
try:
    with open(path, 'x') as f:
        f.write(data)
except FileExistsError:
    pass
```

## Performance Profiling

### CPU Profiling

```python
import cProfile
import pstats

profiler = cProfile.Profile()
profiler.enable()

# Code to profile
expensive_operation()

profiler.disable()
stats = pstats.Stats(profiler)
stats.sort_stats('cumulative')
stats.print_stats(10)
```

### Memory Profiling

```python
import tracemalloc

tracemalloc.start()

# Code to analyze
create_objects()

snapshot = tracemalloc.take_snapshot()
for stat in snapshot.statistics('lineno')[:10]:
    print(stat)
```

### Line Profiler

```bash
pip install line_profiler

# Add @profile decorator to functions
kernprof -l -v script.py
```

## SQL Debugging

```python
# Enable SQLAlchemy logging
import logging
logging.getLogger('sqlalchemy.engine').setLevel(logging.INFO)
```

```sql
-- Analyze query plan
EXPLAIN ANALYZE
SELECT * FROM users WHERE email = 'test@example.com';

-- Find slow queries (PostgreSQL)
SELECT query, calls, mean_exec_time
FROM pg_stat_statements
ORDER BY mean_exec_time DESC
LIMIT 10;
```

## Stack Trace Analysis

```python
# Preserve exception chain
try:
    user = get_user(user_id)
except UserNotFound as e:
    raise ValueError(f"Invalid user_id: {user_id}") from e
```

### Reading Stack Traces

1. Start from the **bottom** - that's the actual error
2. Work **upward** to find your code
3. Check the **line numbers** in your code
4. Look for **variable values** if shown

## Git Bisect

```bash
# Find the commit that introduced a bug
git bisect start
git bisect bad              # Current commit is broken
git bisect good v1.0.0      # This version worked

# Test each commit, then mark
git bisect good  # or
git bisect bad

# When done
git bisect reset
```

## Production Debugging

### Safe Practices

- Never use pdb in production
- Use structured logging
- Add request IDs for tracing
- Enable debug logging temporarily
- Test fixes in staging first

### Log Analysis

```bash
# Find errors in last hour
grep ERROR app.log | tail -100

# Count error types
grep ERROR app.log | cut -d: -f3 | sort | uniq -c

# Find slow requests
awk '$NF > 1000 {print}' access.log
```

## Debugging Checklist

- [ ] Can you reproduce it consistently?
- [ ] Have you read the full error message?
- [ ] Have you checked the stack trace?
- [ ] What changed recently?
- [ ] Have you checked the logs?
- [ ] Have you verified input data?
- [ ] Have you tested your assumptions?
- [ ] Have you tried explaining it to someone?

## Quick Debugging Tips

```python
# Print type and value
print(f"{type(x)=}, {x=}")

# Inspect object
print(dir(obj))
print(vars(obj))

# Check where code is
import inspect
print(inspect.getfile(some_function))

# Get current stack
import traceback
traceback.print_stack()
```
