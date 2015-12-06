# [转]两个实用的 python 装饰器: timeout 和 trace

转载: [Python编程（四）：两个实用的Python的装饰器](http://zhuanlan.zhihu.com/auxten/20175869)

## timeout
通过设置时钟信号给函数添加超时终端功能, 不适用于通过 `os.system()` 调用外部程序的情形.

`timeout.py`:
```python
import signal
import functools

class TimeoutError(Exception):
  pass

def timeout(seconds, error_message='Function call timed out'):
  def decorated(func):
    def _handle_timeout(signum, frame):
      raise TimeoutError(error_message)

    def wrapper(*args, **kwargs):
      signal.signal(signal.SIGALRM, _handle_timeout)
      signal.alarm(seconds)

      try:
        result = func(*args, **kwargs)
      finally:
        signal.alarm(0)

      return result
    return functools.wraps(func)(wrapper)
  return decorated

  @timeout(3)
  def slowfunc(sleep_time):
    import time
    time.sleep(sleep_time)

  if '__main__' == __name__:
    slowfunc(1)
    print 'ok'

    try:
      slowfunc(5)
    except TimeoutError:
      print 'timeout'
```


## trace 函数
提供类似 `bash -x` 调试功能, 打印执行的每一行代码.
```python
import os
import sys
import linecache

def trace(f):
  def localtrace(frame, why, arg):
    if 'line' == why:
      # record the file name and line number of every trace
      filepath = frame.f_code.co_filename
      lineno = frame.f_lineno
      print '{filepath}({lineno}): {line}'.format(
        filepath=filepath,
        lineno=lineno,
        line=linecache.getline(filepath, lineno).strip('\r\n'))

    return localtrace

  def globaltrace(frame, why, arg):
    if 'call' == why:
      return localtrace
    return None

  def _f(*args, **kwards):
    sys.settrace(globaltrace)
    result = f(*args, **kwards)
    sys.settrace(None)
    return result

  return _f

@trace
def xxx():
  print 1

  result = ''
  for i in ['1', '2', '3']:
    if i == '3':
      result += i

  print result

if '__main__' == __name__:
  xxx()
```
