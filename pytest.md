支持验证某一个异常的抛出

支持内建的参数值访问，例如，tmpdir

有用的连接：
[reporting demo](https://docs.pytest.org/en/latest/example/reportingdemo.html#tbreportdemo)

## Conventions for Python test discovery
* 搜索测试目录：testpaths配置项、当前目录、命令行参数
* 搜索所有的子目录
* 搜索目录下的 test_*.py 或 *_test.py 文件
* 在这些文件中，搜索
  * 以 test_ 开头的class之外的函数
  * 以 Test 开头的class中的以 test_ 开头的函数，注意该class没有 __init__ 方法

## pytest执行返回值
有6个不同的返回值：
* 0：所有的测试用例都被采集且执行成功
* 1：所有的测试用例都被采集但有些执行失败
* 2：测试过程被用户中断
* 3：测试执行过程中出现内部错误
* 4：pytest命令使用错误
* 5：没有采集到测试用例

## Setting breakpoints

## Calling pytest from Python code
加入代码
`pytest.main()`

和命令行执行效果一致，区别是：不再抛出 SystemExit 异常，只是返回 exitcode

同样可以传入参数
```
pytest.main(['-x', 'mytestdir'])
pytest.main(['-qq'], plugins=[MyPlugin()])
```

## 常用命令
```
pytest --fixtures   # show available builtin function arguments
pytest -x           # stop after first failure
pytest --maxfail=2  # stop after 2 failures
pytest test_mod.py  # run tests in a module
pytest testing/     # run tests in a directory
pytest -k "MyClass and not method"  # run tests by keyword expressions
pytest test_mod.py::test_func   # run a specific test within a module
pytest test_mod.py::TestClass::test_method  # run a specific test within a class
pytest -m slow      # run all tests which are decorated with the @pytest.mark.slow decorator
pytest --pyargs pkg.testing     # run tests from packages. This will import pkg.testing and use its filesystem location to find and run test from

# 修改traceback的输出内容
pytest -l   # show local variables
pytest --tb=xxx
pytest --full-trace   # 输出完整bt，包括 Ctrl+C 中断时的执行处，可用于查看测试执行情况

# Dropping to PDB on failures
# 当出现错误时，进入PDB模式。可访问的信息 sys.last_value, sys.last_type, sys.last_traceback
pytest -x --pdb               # 
pytest --maxfail=3 -- --pdb   # 

# Profiling test execution duration
pytest --durations=10   # 列出执行时间最长的10个测试用例

# Disabling plugins
pytest -p no:doctest    # disable loading the plugin doctest
```

# The writing and reporting of assertions in tests
## Asserting with the assert statement
```
def f():
  return 3
def test_function():
 assert f() == 4
 
# 重写报错信息
assert a % 2 ==0, "value was odd, should be even"
```

## Assertions about expected exceptions
使用 pytest.raises 作为 context manager
```
import ptest
def test_zero_division():
  with pytest.raises(ZeroDivisionError):
    1 / 0
```

获取异常的具体信息
```
# excinfo 是一个 ExceptionInfo 实例，包含实际的异常
# 属性有：.type, .value, .traceback
def test_recursion_depth():
  with pytest.raises(RuntimeError) as excinfo:
    def f():
      f()
    f()
  assert 'maximum recursion' in str(excinfo.value)
```
使用match关键字匹配特定的异常值
```
import pytest
def myfunc():
  raise ValueError("Exception 123 raised")
def test_match():
  with pytest.raises(ValueError, match=r'.* 123 .*'):
    myfunc()
```

pytest.mark.xfail 可以设置需要捕获的异常
```
@pytest.mark.xfail(raises=IndexError)
def test_f():
  f()
```
**Notice: pytest.raises 使用在测试特定异常，pytest.mark.xfail 使用在记录未修复的bug或特定情况下出现的bug**

## Assertions about expected warnings
TODO

## Making use of context-sensitive comparisons
```
def test_set_comparison():
  set1 = set("1308")
  set2 = set("8035")
  assert set1 == set2

# 执行pytest，报错信息会给出set1有额外的元素1， set2有额外的元素5
```
Special comparisons are done for a number of cases:
* comparing long strings: a context diff is shown
* comparing long sequences: first failing indices
* comparing dicts: different entries

## Defining your own assertion comparison
添加自定义的、更加明确的解释信息
```
# content of conftest.py
from test_foocompare import Foo
def pytest_assertrepr_compare(op, left, right):
  if isinstance(left, Foo) and isinstance(right, Foo) and op == "==":
    return ['Comparing Foo instances:', '    vals: %s != %s' % (left.val, right.val)]

class Foo(object):
  def __init__(self, val):
    self.val = val
  def __eq__(self, other):
    return self.val == other.val

def test_compare():
  f1 = Foo(1)
  f2 = Foo(2)
  assert f1 == f2
```

## Advanced assertion introspection
TODO

# Pytest API and builtin fixtures
## Helpers for assertions about Exceptions/Warnings
### raises(expected_exception, *args, **kwargs)

参数：
* message: if specified, provides a custom failure message if the exception is not raised
* match: if specified, asserts that the exception matches a text or regex

**当使用 pytest.raises 时，抛出异常语句必须是最后一行；在抛出异常语句之后的代码是不会被执行的。**
```
value = 15
with raises(ValueError) as exc_info:
  if value > 10:
    raise ValueError("value must be <= 10")
  assert exc_info.type == ValueError # this will not execute

# 以下为正确的写法
with raises(ValueError) as exc_info:
  if value > 10:
    raise ValueError("value must be <= 10")
assert exc_info.type == ValueError
```
### deprecated_call(func=None, *args, **kwargs)

用于确保代码段抛出 DeprecationWarning 或 PendingDeprecationWarning
```
import warnings
def api_call_v2():
  warnings.warn('use v3 of this api', DeprecationWarning)
  return 200
  
with deprecated_call():
  assert api_call_v2() == 200
```
