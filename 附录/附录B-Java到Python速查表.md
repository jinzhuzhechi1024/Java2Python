# 附录B Java→Python速查表

## B.1 语法对照

| 功能 | Java | Python |
|------|------|--------|
| 变量声明 | `int x = 42;` | `x = 42` |
| 常量 | `final int N = 10;` | `N = 10`（全大写约定） |
| 字符串 | `String s = "hi";` | `s = "hi"` |
| 列表 | `List<String> list = new ArrayList<>();` | `list = []` |
| 字典 | `Map<String, Integer> map = new HashMap<>();` | `d = {}` |
| 集合 | `Set<String> set = new HashSet<>();` | `s = set()` |
| 元组 | 无 | `t = (1, 2, 3)` |

## B.2 控制流对照

| 功能 | Java | Python |
|------|------|--------|
| if | `if (x > 0) { ... }` | `if x > 0: ...` |
| for循环 | `for (int i=0; i<n; i++)` | `for i in range(n):` |
| for-each | `for (String s : list)` | `for s in list:` |
| while | `while (x > 0)` | `while x > 0:` |
| switch | `switch(x) { case 1: ... }` | `match x: case 1: ...`（3.10+） |

## B.3 类与OOP对照

| 功能 | Java | Python |
|------|------|--------|
| 类定义 | `public class User { }` | `class User:` |
| 构造函数 | `public User() { }` | `def __init__(self):` |
| this | `this.name` | `self.name` |
| 继承 | `class Dog extends Animal` | `class Dog(Animal):` |
| 接口 | `interface Shape { }` | `class Shape(Protocol):` |
| 抽象类 | `abstract class Shape` | `class Shape(ABC):` |
| 静态方法 | `static void hello()` | `@staticmethod def hello()` |
| getter | `getName()` | `@property def name(self):` |
| 注解 | `@Override` | `@decorator` |

## B.4 异常对照

| 功能 | Java | Python |
|------|------|--------|
| 捕获 | `try { } catch (Exception e) { }` | `try: ... except Exception: ...` |
| 抛出 | `throw new RuntimeException()` | `raise RuntimeError()` |
| finally | `finally { }` | `finally:` |
| 自定义 | `class MyEx extends Exception` | `class MyEx(Exception):` |

## B.5 集合操作对照

| 功能 | Java | Python |
|------|------|--------|
| 过滤 | `stream().filter()` | `[x for x in lst if cond]` |
| 映射 | `stream().map()` | `[f(x) for x in lst]` |
| 排序 | `stream().sorted()` | `sorted(lst)` |
| 分组 | `Collectors.groupingBy` | `itertools.groupby` / dict推导 |
| 聚合 | `stream().reduce()` | `functools.reduce()` |

## B.6 Java 21+ 新特性与 Python 对照

Java 21（2023年9月 LTS）引入了多项语法革新，其中不少在 Python 中已有对应物。

| 功能 | Java 21+ | Python | 说明 |
|------|----------|--------|------|
| 记录类 | `record Point(int x, int y) {}` | `@dataclass` | Python 3.7+ 用 `@dataclass` 自动生成 `__init__`/`__repr__`/`__eq__`[^pep-557] |
| 密封类 | `sealed interface Shape permits Circle, Square` | 无直接对应 | Python 动态类型无此概念，可用 `Literal` 类型或运行时 `isinstance` 检查近似 |
| 模式匹配 | `if (obj instanceof String s)` | `match` 语句 + 类型模式 | Python 3.10+ 的 `match-case` 支持类型模式匹配[^pep-634] |
| Switch表达式 | `switch(x) { case 1 -> "one"; }` | `match x: case 1: return "one"` | Python `match` 语句（PEP 634，3.10+） |
| 文本块 | `"""multi-line"""` | `"""multi-line"""` | Python 三引号字符串自古就有 |
| 虚拟线程 | `Thread.startVirtualThread()` | `asyncio` 协程 | 不同模型：Java 虚拟线程是轻量级线程，Python 协程是协作式调度 |
| Record Pattern | `case Point(int x, int y) -> ...` | `case Point(x, y):` | Python `match` 的解构模式 |

[^pep-557]: PEP 557: Data Classes, Eric V. Smith, 2017-06-02, https://peps.python.org/pep-0557/ — Python 3.7 引入 `@dataclass` 装饰器。

[^pep-634]: PEP 634: Structural Pattern Matching, Brandt Bucher, 2020-09-08, https://peps.python.org/pep-0634/ — Python 3.10 引入 `match` 语句，支持类型模式和解构模式。
