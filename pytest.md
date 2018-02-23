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

# content of test_foocompare.py
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

## Comparing floating point numbers
### approx(expected, rel=None, abs=None, nan_ok=False)
TODO
## Raising a specific test outcome
通常情况下，可以使用 mark 来达到同样的效果

* fail(msg='', pytrace=True)
* skip(msg='', **kwargs)
* importorskip(modname, minversion=None)
* xfail(reason='')
* exit(msg)

## Fixtures and requests
### fixture(scope='function', params=None, autouse=False, ids=None, name=None)
本身是一个修饰器，用于定义一个fixture function

# pytest fixtures: explicit, modular, scalable
脚手架(fixture)为每个测试用例提供可靠的、可重复执行的初始化设置
## Fixtures as Function arguments
fixture可以作为测试函数的输入参数
```
import pytest

# 生成fixture
@pytest.fixture
def smtp():
  import smtplib
  return smtplib.SMTP("smtp.gmail.com", 587, timeout=5)
  
# 作为参数传入，pytest查找并调用由 @pytest.fixture 修饰的smtp函数
def test_ehlo(smtp):
  response, msg = smtp.ehlo()
  assert response == 250
```

查看可使用的fixture
```
pytest --fixtures test_simplefactory.py
```
## conftest.py: sharing fixture functions
如果有多个测试用例需要使用同一个fixture，那么可以将这个fixture放入 conftest.py 文件中。
不需要使用import，pytest会自动搜索并引入。

fixture的发现机制：
1. test class
2. test module
3. conftest.py
4. builtin 或 第三方plugin

## Sharing test data
几种方式：
1. 使用fixture引入测试数据，这个将使用pytest的自动cache功能
2. 将测试用数据文件统一放在 tests 目录下，相应的插件有：pytest-datadir 和 pytest-datafiles

## Scope: sharing a fixture instance across tests in a class, module or session
有些fixture会使用网络连接，例如stmp。这些操作十分耗时。此时，我们可以使用fixture中的scope参数。
```
# content of conftest.py
import pytest
import smtplib

@pytest.fixture(scope=“module”)
def smtp():
  return smtplib.SMTP(“smtp.gmail.com”, 587, timeout=5)
```
与conftest.py同一目录下或子目录下的测试用例都可以不用import便能使用smtp

## Fixture finalization / executing teardown code
当fixture离开它的scope时，pytest支持执行特定代码。
```
import smtplib
import pytest

@pytest.fixture(scope=“module”)
def smtp():
  smtp = smtplib.SMTP(“smtp.gmail.com”, 587, timeout=5)
  yield smtp	# 将原来的return改用yield语句
  # 以下语句在fixture消亡时执行
  print(“teardown smtp”)
  smtp.close()

另外一种方式，使用with语句
@pytest.fixture(scope=“module”)
def smtp():
  with smtplib.SMTP(“smtp.gmail.com”, 587, timeout=5) as smtp:
    yield smtp
```

** Note that if an exception happens during the setup code (before the yield keyword), the teardown code (after the yield) will not be called. **

另外一种方式是使用 addfinalizer 方法注册
```
@pytest.fixture(scope=“module”)
def smtp(request):
  smtp = smtplib.SMTP(“smtp.gmail.com”, 587, timeout=5)
  def fin():
    print (“teardown smtp”)
    smtp.close()
  request.addfinalizer(fin)
  return smtp
```

addfinalizer 相对于yield有2个特征：
1. 使用注册多个finalizer 函数
2. 无论在setup时是否出现异常，finalizer函数都会被执行

## Fixtures can introspect the requesting test context
使用request对象，fixture可以获取测试用例的context
```
@pytest.fixture(scope=“module”)
def smtp(request):
  # 获取测试用例所在的module中的smtpserver的值
  server = getattr(request.module, “smtpserver”, “smtp.gmail.com”)
  smtp = smtplib.SMTP(server, 587, timeout=5)
  yield smtp
  print (“finalizing %s (%s)” % (smtp, server))
  smtp.close()
```

## Parametrizing fixtures
使用参数化后的fixture时，测试用例将会被多次执行。此时，测试用例并不感知。
```
@pytest.fixture(scope=“module”, params=[“smtp.gmail.com”, “mail.python.org”])
def smtp(request):
  smtp = smtplib.SMTP(request.param, 587, timeout=5)
  yield smtp
  print (“finalizing %s” % smtp)
  smtp.close()
```

pytest会为实际执行的测试用例分配一个id，这个id可以是自动生成，也可以是自定义
```
import pytest
@pytest.fixture(params=[0, 1], ids=[’spam’, ‘ham’])
def a(request):
  return request.param

def test_a(a):
  pass

# 用于生成id的函数
# fixture_value由fixture在初始化时传入，也就是params中的元素
def idfn(fixture_value):
  if fixture_value == 0:
    return “eggs”
  else:
    return None		# 返回None时，pytest将使用自动生成id的方式

@pytest.fixture(params=[0, 1], ids=idfn)
def b(request):
  return request.param

def test_b(b):
  pass
```

## Modularity: using fixtures from a fixture function
在定义fixture时，我们可以调用其他fixture
```
import pytest

class App(object):
  def __init__(self, smtp):
    self.smtp = smtp

@pytest.fixture(scope=“module”)
def app(smtp):	# smtp 为一个fixture
  return App(smtp)

def test_smtp_exists(app):
  assert app.smtp
```

**需要注意不同fixture的scope**

## Automatic grouping of tests by fixture instances
fixture的scope决定了fixture实例的生存周期
scope=function时，每个实例执行完成后就会消亡
scope=module时，fixture实例只有在多个使用到的测试用例都执行完成后才会消亡

## Using fixtures from classes, modules or projects
**class级别**
```
# content of conftest.py
import pytest
import tempfile
import os

@pytest.fixture()
def cleandir():
  newpath = tempfile.mkdtemp()
  os.chdir(newpath)

# content of test_setenv.py
import os
import pytest

# 类中每一个方法都会使用cleandir fixture
@pytest.mark.usefixtures(“cleandir”)
class TestDirInit(object):
  def test_cwd_starts_empty(self):
    assert os.listdir(os.getcwd()) == []
    with open(“myfile”, “w”) as f:
      f.write(“hello”)

  def test_cwd_again_starts_empty(self):
    assert os.listdir(os.getcwd()) == []
```
**module级别**
```
pytestmark = pytest.mark.usefixtures(“cleandir”)
```
必须命名为 pytestmark

**project级别**
```
# content of pytest.ini
[pytest]
usefixtures = cleandir
```
## Autouse fixtures (xUnit setup on steroids)
TODO
同上述的class级别

## Overriding fixtures on various levels
### Override a fixture on a folder (conftest) level
### Override a fixture on a test module level
### Override a fixture with direct test parametrization
```
tests/
  __init__.py

  conftest.py
    # content of tests/conftest.py
    import pytest
    @pytest.fixture
    def username():
      return ‘username’
    @pytest.fixture
    def other_username(username):
      return ’other-‘ + username

  test_something.py
    # content
    import pytest
    # 直接覆盖fixture
    @pytest.mark.parametrize(‘username’, [‘directly-overriden-username’])
    def test_username(username):
      assert username == ‘directly-overriden-username’

    # 注意此处，fixture实例的传入参数username
    @pytest.mark.parametrize(‘username’, [‘directly-overriden-username-other’])
    def test_username_other(other_username):
      assert other_username == ‘other-directly-overriden-username-other’
```

### Override a parametrized fixture with non-parametrized one and vice versa

# Monkeypatching/mocking modules and environments
用于设置一些全局变量，或是更加复杂的测试环境

## Simple example: monkeypatching functions
```
import os.path
# 一个假设的测试用代码
def getssh():
  return os.path.join(os.path.expanduser(“~admin”), ‘.ssh’)

def test_mytest(monkeypatch):
  def mockreturn(path):
    return ‘/abc’
  monkeypatch.setattr(os.path, ‘expanduser’, mockreturn)
  x = getssh()
  assert x == ‘/abc/.ssh’
```

## example: preventing “requests” from remote operations
```
# content of conftest.py
import pytest
@pytest.fixture(autouse=True)
def no_requests(monkeypatch):
  monkeypatch.delattr(“requests.sessions.Session.request”)
```
在这里所有试图调用request方法都将返回错误
**注意：不建议对builtin函数做patch，可能会造成pytest的内部错误。例如open、compile等**

## Method reference of the monkeypatch fixture
class MonkeyPatch
方法有：
* setattr(target, name, value=<notset>, raising=True)
* delattr(target, name=<notset>, raising=True)
* setitem(dic, name, value)
* delitem(dic, name, raising=True)
* setenv(name, value, prepend=None)
* delenv(name, raising=True)
* syspath_prepend(path)
* chdir(path)
* undo()

# Temporary directories and files
## The ‘tmpdir’ fixture
tmpdir fixture 是一个py.path.local对象
```
# content of test_tmpdir.py
import os
def test_create_file(tmpdir):
  p = tmpdir.mkdir(“sub”).join(“hello.txt”)
  p.write(“content”)
  assert p.read() == “content”
  assert len(tmpdir.listdir()) == 1
  assert 0
```

## The ‘tmpdir_factory’ fixture
tmpdir_factory 是一个 session-scoped fixture。可以用于需要多次使用的情况，节省时间花销。
```
# contents of conftest.py
import pytest

@pytest.fixture(scope=‘session’)
def image_file(tmpdir_factory):
  img = compute_expensive_image()
  fn = tmpdir_factory.mktemp(‘data’).join(‘img.png’)
  img.save(str(fn))
  return fn

# contents of test_image.py
def test_histogram(image_file):
  img = load_image(image_file)
  # compute and test histogram
```
包含的方法：

* TempdirFactory.mktemp(basename, numbered=True) create a subdirectory of the base temporary directory and return it.
* TempdirFactory.getbasetemp() return base temporary directory

## The default base temporary directory
默认情况下，临时目录创建在系统的临时目录下。命名如 pytest-NUM

可设置临时目录
```
pytest —basetemp=mydir
```

# Capturing of the stdout/stderr output
## Default stdout/stderr/stdin capturing behaviour
在测试执行中，任何在stdout、stderr上的输出都会被捕获。并会被记录在traceback中。

另外，stdin被设置为null对象，在运行自动测试时不能进行交互式输入

## Setting capturing methods or disabling capturing
有2种方式：

* file descriptor (FD) level capturing (default): All writes going to the operating system file descriptors 1 and 2 will be captured.
* sys level capturing: Only writes to Python files sys.stdout and sys.stderr will be captured. No capturing of writes to filedescriptors is performed.

```
pytest -s			# disable all capturing
pytest —capture=sys	# replace sys.stdout/stderr with in-mem files
pytest —capture=fd	# also point filedescriptors 1 and 2 to temp files
```

## Using print statements for debugging
获取stdout/stderr的一个主要目的是用于debug

## Accessing captured output from a test function
使用到的fixture：capsys, capsysbinary, capfd, capfdbinary

```
def test_myoutput(capsys):
  print(“hello”)
  sys.stderr.write(“world\n”)
  captured = capsys.readouterr()
  assert captured.out == “hello\n”
  assert captured.err == “world\n”
  print(“next”)
  captured = capsys.readouterr()
  assert captured.out == “next\n”
```

# Warnings Capture
TODO

# Doctest integration for modules and test files
TODO

# Marking test functions with attributes
使用 pytest.mark helper 可以为测试用例设置相关属性

内建的marker有

* skip		- always skip a test function
* skipif		- 在某条件下跳过测试用例
* xfail		- produce an “expected failure” outcome if a certain condition is met
* parametrize	- 执行多次调用为同一个测试用例

## API reference for mark related objects
TODO

# Skip and xfail: dealing with tests that cannot succeed
对于意料中的无法通过的测试用例，我们标记这些测试用例，这样pytest会过滤这些测试用例并在小结中展示出来。同时测试结果通过。


