# 附录C Python标准库速览

## C.1 常用标准库

| 模块 | 用途 | Java对应 |
|------|------|---------|
| `os` | 操作系统接口 | `java.lang.System` + `java.nio.file` |
| `sys` | 解释器环境 | `System.getProperties()` |
| `pathlib` | 路径操作 | `java.nio.file.Path` |
| `collections` | 数据结构增强 | Guava Collections |
| `itertools` | 迭代器工具 | Stream API |
| `functools` | 高阶函数 | Function/Consumer |
| `typing` | 类型注解 | 泛型系统 |
| `asyncio` | 异步编程 | CompletableFuture/NIO |
| `json` | JSON处理 | Jackson/Gson |
| `re` | 正则表达式 | `java.util.regex` |
| `logging` | 日志 | SLF4J/Logback |
| `unittest` | 单元测试 | JUnit |
| `dataclasses` | 数据类 | Lombok @Data |
| `enum` | 枚举 | `java.lang.Enum` |
| `datetime` | 日期时间 | `java.time` |
| `subprocess` | 进程管理 | `ProcessBuilder` |

## C.2 常用第三方库

| 库 | 用途 | Java对应 |
|---|------|---------|
| `pandas` | 数据处理 | 无直接对应 |
| `numpy` | 数值计算 | 无直接对应 |
| `fastapi` | Web框架 | Spring Boot |
| `flask` | 轻量Web | Spark Java |
| `sqlalchemy` | ORM | Hibernate/JPA |
| `pytest` | 测试框架 | JUnit |
| `mypy` | 类型检查 | javac编译器 |
| `pydantic` | 数据验证 | Bean Validation |
| `celery` | 任务队列 | Spring Batch |
| `redis` | Redis客户端 | Jedis/Lettuce |
| `httpx` | HTTP客户端 | OkHttp/HttpClient |
| `beautifulsoup4` | HTML解析 | Jsoup |
