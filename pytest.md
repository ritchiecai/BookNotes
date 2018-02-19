支持验证某一个异常的抛出

支持内建的参数值访问，例如，tmpdir

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

