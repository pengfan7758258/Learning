记录function花费的时间
```python
import time
import functools

def log_time(func):
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        print(f"[START] {func.__name__}")
        start = time.time()
        
        result = func(*args, **kwargs)
        
        end = time.time()
        print(f"[END] {func.__name__} | Elapsed: {end - start:.2f} sec")
        return result
    return wrapper

```