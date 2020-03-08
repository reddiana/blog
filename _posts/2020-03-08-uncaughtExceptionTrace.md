---
toc: true
description: "Uncaught Exception Trace"
categories: [python, exception]
---

# Uncaught Exception Trace


- [파이썬 공식 자습서 - 8. 에러와 예외](https://docs.python.org/ko/3/tutorial/errors.html#errors-and-exceptions)
- [파이썬을 이용한 전문적인 오류 처리](https://code.tutsplus.com/ko/tutorials/professional-error-handling-with-python--cms-25950)
- [Catching every single exception with Python](https://dev.to/joshuaschlichting/catching-every-single-exception-with-python-40o3) 


```python
import sys
import traceback

def exception_hook(type, value, tb):
    logging.error('='*80)
    logging.error(f'Type: {type}')
    logging.error(f'Value: {value}')
    t = traceback.format_exception(type, value, tb)
    t = [i.rstrip().split('\n') for i in t]
    for i in t:
        for j in i:
            logging.error(j)
    logging.error('-'*80)

sys.excepthook = exception_hook

1 / 0
```

```less
2020-03-08 17:51:52 root      ERROR ================================================================================
2020-03-08 17:51:52 root      ERROR Type: <class 'IndexError'>
2020-03-08 17:51:52 root      ERROR Value: tuple index out of range
2020-03-08 17:51:52 root      ERROR Traceback (most recent call last):
2020-03-08 17:51:52 root      ERROR   File "d:/temp/uncaughtExceptionTrace.py", line 59, in <module>
2020-03-08 17:51:52 root      ERROR     lumberjack()
2020-03-08 17:51:52 root      ERROR   File "d:/temp/uncaughtExceptionTrace.py", line 52, in lumberjack
2020-03-08 17:51:52 root      ERROR     bright_side_of_death()
2020-03-08 17:51:52 root      ERROR   File "d:/temp/uncaughtExceptionTrace.py", line 56, in bright_side_of_death
2020-03-08 17:51:52 root      ERROR     return tuple()[0]
2020-03-08 17:51:52 root      ERROR IndexError: tuple index out of range
2020-03-08 17:51:52 root      ERROR --------------------------------------------------------------------------------
```

<script src="https://gist.github.com/everlearningemployee/1746cd89615dfebed068345f5505d525.js"></script>
