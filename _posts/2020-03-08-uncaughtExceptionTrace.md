---
toc: true
description: "Uncaught Exception Trace"
categories: [python]
---

# Uncaught Exception Trace

```python
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

<script src="https://gist.github.com/everlearningemployee/1746cd89615dfebed068345f5505d525.js"></script>