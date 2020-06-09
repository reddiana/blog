---
title: Python Graceful Shutdown
description: "세련되게 살자"
layout: post
categories: [Python]
---

```python
app = flask.Blueprint('app', __name__)

# Graceful Shutdown (signal 처리) 구현
imBusy = []
def graceful_shutdown_handler(signum = None, frame = None):
    logging.info("I am dying.. [%s]" % signum)
    logging.debug("working [%s]" % imBusy)

    while len(imBusy) > 0:
        time.sleep(0.5)        

    logging.info("Good Bye")
    exit(0)

# signal 처리 등록
try:
    signals = [signal.SIGTERM, signal.SIGINT, signal.SIGHUP]
except:    
    # Windows에는 signal.SIGHUP 없음
    signals = [signal.SIGTERM, signal.SIGINT]

for sig in signals:
    signal.signal(sig, graceful_shutdown_handler) 

@app.before_app_request
def imbusy():
    global imBusy
    imBusy.append(threading.currentThread().name)

@app.after_app_request
def solong(res):
    global imBusy
    logging.debug("remove [%s]" % threading.currentThread().name)
    imBusy.remove(threading.currentThread().name)
    return res
```
