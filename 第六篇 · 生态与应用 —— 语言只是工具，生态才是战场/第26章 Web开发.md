### 第26章 Web开发：从Spring Boot到FastAPI的极简哲学

> 📖 本章你将学会：
> - 理解 Spring Boot 全家桶与 FastAPI 极简哲学的根本分歧
> - 用 FastAPI 的 Depends 替代 Spring @Autowired，掌握 yield 生命周期
> - 读懂 ASGI 异步网关模型，对比 Servlet 容器的线程模型
> - 用 Pydantic v2 的类型提示驱动数据校验，替代 Bean Validation
> - 利用 OpenAPI 自动文档，零配置生成可交互 API 文档

---

#### 开篇：重装骑士与轻量刺客

你在 Java 世界里写 Web API 已经很多年了。每次起一个新项目，第一件事是什么？打开 [start.spring.io](https://start.spring.io)，勾选 Spring Web、Spring Data JPA、Validation，下载一个 zip，解压，`mvn spring-boot:run`，然后等上三五秒——Spring 容器初始化、扫描 Bean、注册 DispatcherServlet、构建 Tomcat 连接池——服务终于起来了。你想加一个 GET 接口？写 Controller、写 Service、写 Repository、写 Entity、写 DTO，五个文件起步。这还没算 `application.yml` 的数据库配置和 `pom.xml` 的依赖管理。

Spring Boot 像一位重装骑士——盔甲齐全，DI 容器、AOP 切面、Security 过滤链、事务管理器，一应俱全。穿上这身盔甲，你什么都不缺，但穿盔甲要时间，走路也要费力。FastAPI 像一位轻量刺客——一把刀（路由装饰器）、一个盾（Depends 依赖注入），不穿盔甲，快速出击。5 行代码起一个 API，0.1 秒启动，自动生成可交互文档[^fastapi-docs]。这不是偷工减料，而是另一种哲学：**你不需要的东西，不应该出现在你的启动路径里**。

[^fastapi-docs]: FastAPI 官方文档，https://fastapi.tiangolo.com/

Spring Boot 的"重"不是缺陷，而是企业级场景的刚需——当你需要分布式事务、集群会话、消息队列集成时，那身盔甲就是保命的东西。FastAPI 的"轻"也不是万能——当你需要复杂的事务协调、重度 ORM 继承体系时，你会发现 Python 生态的企业级基建确实不如 Spring 成熟。本章的任务不是分高下，而是让 Java 老兵理解：**同一个 Web API 问题，换一种哲学可以有多简洁**。

---

#### Java 回顾

在 Spring Boot 中写一个 RESTful API，典型代码长这样：

```java
// Spring Boot: Controller + Service + Repository 五件套
@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}

@RestController
@RequestMapping("/api/users")
public class UserController {
    @Autowired
    private UserService service;

    @GetMapping("/{id}")
    public UserDTO getUser(@PathVariable Long id) {
        return service.findById(id);
    }

    @PostMapping
    public UserDTO createUser(@Valid @RequestBody CreateUserRequest req) {
        return service.create(req);
    }
}

@Service
public class UserService {
    @Autowired
    private UserRepository repo;

    public UserDTO findById(Long id) {
        return repo.findById(id)
            .map(this::toDTO)
            .orElseThrow(() -> new NotFoundException("User not found"));
    }
}

public interface UserRepository extends JpaRepository<User, Long> {}
```

Spring Boot 的核心特点：注解驱动的隐式装配。`@Autowired` 在背后注入 Bean，`@Valid` 在背后触发 Bean Validation，`@RequestMapping` 在背后注册到 DispatcherServlet 的路由表。你不打开 Spring 文档，很难知道这些魔法是怎么串起来的。这套体系成熟、强大，但"隐式"带来的代价是：**启动慢（IoC 容器初始化）、内存大（全家桶加载）、调试链路长（代理层叠代理）**[^spring-boot-arch]。

[^spring-boot-arch]: Spring Boot 的自动装配和 IoC 容器机制参见 Spring Framework 官方文档，https://docs.spring.io/spring-framework/reference/

> 💡 提示：如果你对 Spring Boot 的 Controller-Service-Repository 分层已经很熟悉，可以跳过本节直接看下一节。

---

#### Python 视角：FastAPI 的极简哲学

FastAPI 由 Sebastián Ramírez（网名 tiangolo）于 2018 年 12 月 5 日创建[^fastapi-wiki]，基于两个核心库构建：Starlette 负责 Web 部分（ASGI 路由、中间件、WebSocket），Pydantic 负责数据部分（类型校验、序列化）。截至 2026 年，FastAPI 在 GitHub 上拥有超过 99k Stars，被 Microsoft、Netflix、Uber、Cisco、Expedia 等公司采用[^fastapi-quotes]。

[^fastapi-wiki]: FastAPI 首次发布日期由 Wikipedia 确认，https://en.wikipedia.org/wiki/FastAPI
[^fastapi-quotes]: FastAPI 官方首页引用的企业用户评价，https://fastapi.tiangolo.com/features/

FastAPI 的设计哲学可以用一句话概括：**Python 类型提示做一切**。你声明的函数参数类型，同时驱动了路由解析、数据校验、JSON 序列化和 OpenAPI 文档生成——一次声明，四种用途。

**同一个 GET 接口，FastAPI 写法**：

```python
from fastapi import FastAPI

app = FastAPI()

@app.get("/api/users/{id}")
def get_user(id: int):
    return {"id": id, "name": "Alice"}
```

没有 `@SpringBootApplication`，没有 `@RestController`，没有 `@Autowired`，没有 `@RequestMapping`。一个装饰器搞定路由，一个类型提示 `id: int` 自动完成路径参数解析和类型校验——如果客户端传 `/api/users/abc`，FastAPI 直接返回 422 错误，附带清晰的错误信息，不需要你写任何校验代码。

Spring Boot 与 FastAPI 的关键差异：

| 维度 | Spring Boot | FastAPI | 差异本质 |
|------|------------|---------|----------|
| 启动模型 | IoC 容器扫描+Bean初始化 | 函数装饰器即时注册 | 反射装配 vs 声明式路由 |
| 依赖注入 | @Autowired 隐式注入 | Depends() 显式声明 | 容器托管 vs 函数参数传值 |
| 数据校验 | Bean Validation @Valid | Pydantic 类型提示 | 注解触发 vs 类型驱动 |
| API 文档 | 需配 swagger-springdoc | 内置 /docs + /redoc | 额外配置 vs 零配置生成 |
| 异步模型 | Servlet 线程池 | ASGI 事件循环 | 一请求一线程 vs 协程并发 |
| 最小代码 | 20行+配置+依赖 | 5行 | 全家桶 vs 微核心 |

<!-- SVG对比图：Spring Boot重启动 vs FastAPI轻启动 -->
<svg width="560" height="400" xmlns="http://www.w3.org/2000/svg" font-family="sans-serif" font-size="12">
  <!-- 标题 -->
  <text x="280" y="24" text-anchor="middle" font-size="14" font-weight="bold">Spring Boot vs FastAPI 启动链路对比</text>

  <!-- 左侧 Spring Boot -->
  <text x="140" y="54" text-anchor="middle" font-weight="bold" fill="#6B7280">Spring Boot 启动（3-10s）</text>
  <rect x="30" y="64" width="220" height="30" rx="4" fill="#FEE2E2" stroke="#EF4444" stroke-width="1"/>
  <text x="140" y="83" text-anchor="middle" fill="#991B1B">IoC 容器初始化</text>
  <rect x="30" y="100" width="220" height="30" rx="4" fill="#FEE2E2" stroke="#EF4444" stroke-width="1"/>
  <text x="140" y="119" text-anchor="middle" fill="#991B1B">扫描 @Component/@Bean</text>
  <rect x="30" y="136" width="220" height="30" rx="4" fill="#FEE2E2" stroke="#EF4444" stroke-width="1"/>
  <text x="140" y="155" text-anchor="middle" fill="#991B1B">代理生成（AOP/事务）</text>
  <rect x="30" y="172" width="220" height="30" rx="4" fill="#FEE2E2" stroke="#EF4444" stroke-width="1"/>
  <text x="140" y="191" text-anchor="middle" fill="#991B1B">DispatcherServlet 注册</text>
  <rect x="30" y="208" width="220" height="30" rx="4" fill="#FEE2E2" stroke="#EF4444" stroke-width="1"/>
  <text x="140" y="227" text-anchor="middle" fill="#991B1B">Tomcat 连接池启动</text>
  <rect x="30" y="244" width="220" height="30" rx="4" fill="#D1FAE5" stroke="#10B981" stroke-width="1.5"/>
  <text x="140" y="263" text-anchor="middle" fill="#065F46" font-weight="bold">就绪 → 接受请求</text>

  <!-- 右侧 FastAPI -->
  <text x="420" y="54" text-anchor="middle" font-weight="bold" fill="#6B7280">FastAPI 启动（0.1s）</text>
  <rect x="310" y="64" width="220" height="30" rx="4" fill="#DBEAFE" stroke="#3B82F6" stroke-width="1"/>
  <text x="420" y="83" text-anchor="middle" fill="#1E40AF">装饰器注册路由表</text>
  <rect x="310" y="100" width="220" height="30" rx="4" fill="#DBEAFE" stroke="#3B82F6" stroke-width="1"/>
  <text x="420" y="119" text-anchor="middle" fill="#1E40AF">生成 OpenAPI Schema</text>
  <rect x="310" y="136" width="220" height="30" rx="4" fill="#D1FAE5" stroke="#10B981" stroke-width="1.5"/>
  <text x="420" y="155" text-anchor="middle" fill="#065F46" font-weight="bold">就绪 → 接受请求</text>

  <!-- 底部对比标注 -->
  <text x="140" y="292" text-anchor="middle" fill="#6B7280">5 层初始化 + 反射代理</text>
  <text x="420" y="292" text-anchor="middle" fill="#6B7280">2 步注册，无反射</text>

  <!-- 差异说明 -->
  <rect x="30" y="310" width="500" height="76" rx="6" fill="#F9FAFB" stroke="#E5E7EB"/>
  <text x="42" y="330" font-size="11" fill="#374151">⚡ FastAPI 无 IoC 容器、无代理生成、无连接池预热</text>
  <text x="42" y="348" font-size="11" fill="#374151">⚡ 路由在装饰器执行时即注册，无需运行时反射扫描</text>
  <text x="42" y="366" font-size="11" fill="#374151">⚡ OpenAPI Schema 在启动时一次性生成，运行时零开销</text>
</svg>

图中可以看到，Spring Boot 的启动链路有 5 层——IoC 容器、Bean 扫描、AOP 代理、Servlet 注册、连接池启动——每一层都需要时间。FastAPI 只有 2 步：装饰器注册路由、生成 OpenAPI Schema。这就是 0.1 秒 vs 5 秒的根源。

---

#### 代码实战：完整 CRUD 对照

**场景描述**：实现一个用户管理 API，包含创建用户、查询单个用户、查询用户列表三个接口，带数据校验和数据库操作。

```java
// === Spring Boot 实现 ===
// 文件1: Entity
@Entity
@Table(name = "users")
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    @Column(nullable = false)
    private String name;
    @Column(nullable = false, unique = true)
    private String email;
    private Integer age;
    // getter/setter 省略
}

// 文件2: DTO（请求校验）
public class CreateUserRequest {
    @NotBlank
    private String name;
    @Email
    @NotBlank
    private String email;
    @Min(0) @Max(150)
    private Integer age;
    // getter/setter 省略
}

// 文件3: Repository
public interface UserRepository extends JpaRepository<User, Long> {
    boolean existsByEmail(String email);
}

// 文件4: Service
@Service
public class UserService {
    @Autowired
    private UserRepository repo;

    public User create(CreateUserRequest req) {
        if (repo.existsByEmail(req.getEmail())) {
            throw new IllegalArgumentException("Email already exists");
        }
        User user = new User();
        user.setName(req.getName());
        user.setEmail(req.getEmail());
        user.setAge(req.getAge());
        return repo.save(user);
    }

    public User findById(Long id) {
        return repo.findById(id)
            .orElseThrow(() -> new NoSuchElementException("Not found"));
    }

    public List<User> findAll() {
        return repo.findAll();
    }
}

// 文件5: Controller
@RestController
@RequestMapping("/api/users")
public class UserController {
    @Autowired
    private UserService service;

    @PostMapping
    public User create(@Valid @RequestBody CreateUserRequest req) {
        return service.create(req);
    }

    @GetMapping("/{id}")
    public User getById(@PathVariable Long id) {
        return service.findById(id);
    }

    @GetMapping
    public List<User> list() {
        return service.findAll();
    }
}
```

5 个文件，约 80 行核心代码（不含 getter/setter），外加 `pom.xml` 依赖和 `application.yml` 配置。

```python
# === FastAPI + SQLModel 实现 ===
# 一个文件搞定
from fastapi import FastAPI, Depends, HTTPException
from sqlmodel import Field, Session, SQLModel, select

# 模型定义：同时是数据库模型 + API校验模型
class User(SQLModel, table=True):
    id: int | None = Field(default=None, primary_key=True)
    name: str = Field(min_length=1)
    email: str = Field(regex=r'^[^@]+@[^@]+\.[^@]+$')
    age: int | None = Field(default=None, ge=0, le=150)

# 创建请求模型（不含 id，防止客户端传入）
class CreateUserRequest(SQLModel):
    name: str = Field(min_length=1)
    email: str = Field(regex=r'^[^@]+@[^@]+\.[^@]+$')
    age: int | None = Field(default=None, ge=0, le=150)

# 数据库依赖：yield 模式管理 Session 生命周期
def get_session():
    with Session(engine) as session:
        yield session

app = FastAPI()

@app.post("/api/users", response_model=User)
def create_user(req: CreateUserRequest, session: Session = Depends(get_session)):
    existing = session.exec(select(User).where(User.email == req.email)).first()
    if existing:
        raise HTTPException(400, "Email already exists")
    user = User(**req.model_dump())
    session.add(user)
    session.commit()
    session.refresh(user)
    return user

@app.get("/api/users/{id}", response_model=User)
def get_user(id: int, session: Session = Depends(get_session)):
    user = session.get(User, id)
    if not user:
        raise HTTPException(404, "User not found")
    return user

@app.get("/api/users", response_model=list[User])
def list_users(session: Session = Depends(get_session)):
    return session.exec(select(User)).all()
```

1 个文件，约 40 行代码，无需额外配置文件。启动命令就一行：`uvicorn main:app --reload`。

**关键差异解读**：

- **模型复用**：Spring Boot 中 Entity（数据库映射）和 DTO（API 校验）是两个类，字段重复声明。SQLModel 让一个类同时具备数据库映射和 API 校验能力[^sqlmodel-docs]，因为它的底层就是 SQLAlchemy + Pydantic 的融合体——Sebastián Ramírez 同时是 FastAPI 和 SQLModel 的作者。
- **校验声明**：Spring Boot 用 `@NotBlank`、`@Email`、`@Min` 等注解。FastAPI 用 `Field(min_length=1)`、`Field(regex=...)`、`Field(ge=0)`——同样是声明式，但直接绑在类型提示上，编辑器能提供更好的补全。
- **依赖注入**：Spring Boot 用 `@Autowired` 隐式注入 Service。FastAPI 用 `Depends(get_session)` 显式声明——你看函数签名就知道这个接口需要数据库 Session，不需要打开 Spring 配置文件。

[^sqlmodel-docs]: SQLModel 官方文档，https://sqlmodel.tiangolo.com/。SQLModel 由 Sebastián Ramírez 创建，结合了 SQLAlchemy 和 Pydantic。

> ⚠️ 注意：Java 老兵最容易踩的坑是在 `async def` 路由函数里调用同步的数据库操作。FastAPI 的 `async def` 在事件循环中执行，同步阻塞调用会卡住整个事件循环。如果你的数据库驱动是同步的（如 SQLModel 默认的同步 Session），路由函数用普通 `def` 而不是 `async def`——FastAPI 会自动把它放到线程池中执行，不会阻塞事件循环。

---

#### 依赖注入深度对比：@Autowired vs Depends

Spring 的 `@Autowired` 和 FastAPI 的 `Depends` 表面功能相似，底层哲学截然不同。

**Spring @Autowired：容器托管，隐式注入**

```java
@Service
public class OrderService {
    @Autowired  // Spring 容器在启动时找到 UserService Bean，注入进来
    private UserService userService;
}
```

你不打开 Spring 配置，不知道 `userService` 从哪来——可能是同一个 JVM 里的 Bean，可能是远程代理，可能是 `@MockBean` 替换的测试桩。这种"隐式"的好处是解耦，代价是调试链路长。

**FastAPI Depends：函数参数，显式声明**

```python
from typing import Annotated
from fastapi import Depends

# 依赖函数：带 yield 的生命周期管理
def get_session():
    with Session(engine) as session:
        yield session  # 请求处理前 yield，处理后执行 finally

# 嵌套依赖：依赖可以依赖其他依赖
def get_current_user(session: Annotated[Session, Depends(get_session)],
                     token: str = Header(...)):
    user = validate_token(token, session)
    if not user:
        raise HTTPException(401, "Invalid token")
    return user

# 路由函数：依赖链一目了然
@app.get("/orders")
def list_orders(user: Annotated[User, Depends(get_current_user)],
                session: Annotated[Session, Depends(get_session)]):
    return session.exec(select(Order).where(Order.user_id == user.id)).all()
```

`Depends` 的核心特性[^fastapi-depends-yield]：

[^fastapi-depends-yield]: FastAPI 官方文档——使用 yield 的依赖项，https://fastapi.tiangolo.com/tutorial/dependencies/dependencies-with-yield/

- **yield 生命周期**：`yield` 之前的代码在请求处理前执行（setup），`yield` 之后的代码在请求处理后执行（teardown）。这是数据库 Session 管理的标准模式——`yield session` 交出 Session，请求结束后自动执行 `finally` 关闭连接。
- **嵌套依赖**：`get_current_user` 依赖 `get_session`，FastAPI 自动解析依赖树，按正确顺序执行。依赖树可以任意深度嵌套。
- **请求内缓存**：同一个请求中，如果多个路由参数都依赖 `get_session`，FastAPI 只调用一次 `get_session`，返回值被缓存。这就像 Spring 的 `@Scope("request")` 单例——但不需要注解，默认行为。
- **scope 参数**：`Depends(get_session, scope="function")` 在路由函数返回后、响应发送前执行 teardown；`scope="request"`（默认）在响应发送后执行 teardown。这对数据库事务的提交时机很关键。

**对比表**：

| 特性 | Spring @Autowired | FastAPI Depends |
|------|-------------------|-----------------|
| 注入方式 | 字段/构造器，容器反射注入 | 函数参数，显式声明 |
| 可见性 | 隐式，需看配置才知道来源 | 显式，函数签名即文档 |
| 生命周期 | Bean scope 控制（singleton/prototype/request） | yield 控制 + scope 参数 |
| 嵌套深度 | 任意，容器递归解析 | 任意，函数递归解析 |
| 请求内缓存 | @Scope("request") | 默认缓存，use_cache=False 可禁用 |
| 测试替换 | @MockBean / @TestPropertySource | app.dependency_overrides |

---

#### ASGI 异步模型 vs Servlet 线程模型

Spring Boot 基于 Servlet 容器（Tomcat 默认），FastAPI 基于 ASGI 服务器（Uvicorn 推荐）。这两种模型的根本差异决定了它们各自的性能特征。

**Servlet 模型：一请求一线程**

Tomcat 为每个请求分配一个线程，线程在请求处理期间被独占。如果处理中有 I/O 等待（数据库查询、远程调用），线程阻塞，什么也不做，但不能被复用。默认线程池 200 个线程，意味着最多 200 个并发请求。

**ASGI 模型：事件循环 + 协程**

Uvicorn 在单个事件循环上运行，每个请求是一个协程。当协程 `await` 一个 I/O 操作时，事件循环切换到其他协程继续工作。一个线程可以处理数百甚至数千个并发连接——只要它们大部分时间在等 I/O[^asgi-spec]。

[^asgi-spec]: ASGI 官方规范文档，https://asgi.readthedocs.io/。ASGI 由 Django Channels 项目（Andrew Godwin 等）于 2016 年创建，作为 WSGI（PEP 3333）的异步继承者。

WSGI 与 ASGI 的核心差异：

| 维度 | WSGI (Spring/Flask/Django) | ASGI (FastAPI/Starlette) |
|------|---------------------------|--------------------------|
| 规范 | PEP 3333 (2003年) | ASGI Spec (2016年) |
| 调用签名 | `app(environ, start_response)` | `async app(scope, receive, send)` |
| 并发模型 | 多线程/多进程 | 事件循环 + 协程 |
| WebSocket | 不支持 | 原生支持 |
| HTTP/2 | 不支持 | 支持 |
| 长连接 | 不支持 | 支持（SSE/WebSocket） |
| 典型服务器 | Tomcat/Gunicorn/uWSGI | Uvicorn/Hypercorn/Daphne |

> 💡 比喻：Servlet 模型像银行柜台——每个窗口一个柜员（线程），客户（请求）排队等柜员。柜员去查资料（I/O）时，窗口闲置，后面的人干等。ASGI 模型像医院分诊台——一个护士（事件循环）同时处理多个病人，病人去做检查（I/O）时护士不干等，立刻处理下一个病人。

**中间件模型对比**：

Spring Boot 用 Servlet Filter 链处理横切逻辑（鉴权、日志、CORS）。FastAPI 用 ASGI 中间件，本质相同但执行方式不同——Filter 链是同步的，一层调一层；ASGI 中间件是异步的，`await call_next(request)` 把控制权交给下一层。

```python
# FastAPI 中间件：请求计时
import time
from fastapi import Request

@app.middleware("http")
async def timing_middleware(request: Request, call_next):
    start = time.perf_counter()
    response = await call_next(request)  # 交给下一层
    duration = time.perf_counter() - start
    response.headers["X-Response-Time"] = f"{duration*1000:.1f}ms"
    return response
```

这和 Spring Boot 的 `OncePerRequestFilter` 功能等价，但写法更简洁——一个装饰器、一个函数，不需要继承基类。

---

#### Pydantic v2：Rust 内核的类型校验引擎

FastAPI 的数据校验由 Pydantic 驱动。Pydantic v2 是一次从底层重写的重大升级——核心校验和序列化引擎用 Rust 编写（`pydantic-core`），速度比 v1（纯 Python）快 5 到 50 倍[^pydantic-v2]。

[^pydantic-v2]: Pydantic v2 官方文档，https://docs.pydantic.dev/latest/。Pydantic v2 的核心用 Rust 重写，由 Samuel Colvin 主导。

Pydantic v1 到 v2 的迁移在 FastAPI 中经历了一个渐进过程[^fastapi-pydantic-migration]：

[^fastapi-pydantic-migration]: FastAPI Pydantic v1→v2 迁移时间线基于 PyPI 发布历史，https://pypi.org/project/fastapi/#history

- **FastAPI 0.100.0**（2023年7月）：首次支持 Pydantic v2，同时兼容 v1
- **FastAPI 0.119.0**：引入 `pydantic.v1` 子模块，允许渐进迁移
- **FastAPI 0.126.0**：正式弃用 Pydantic v1
- **FastAPI 0.128.0**：完全弃用 `pydantic.v1` 子模块
- **FastAPI 0.130.0**：引入 Pydantic Rust JSON 序列化，性能提升约 2 倍
- **当前版本 0.141.1**（2026年7月29日）：完全基于 Pydantic v2[^fastapi-pypi]

[^fastapi-pypi]: FastAPI PyPI 发布历史，https://pypi.org/project/fastapi/

Pydantic v2 的模型定义对 Java 老兵来说非常直观——它就像一个带自动校验的 DTO 类：

```python
from pydantic import BaseModel, Field, EmailStr

class CreateUserRequest(BaseModel):
    name: str = Field(min_length=1, max_length=100)
    email: EmailStr  # 内置邮箱格式校验
    age: int = Field(default=0, ge=0, le=150)
    tags: list[str] = Field(default_factory=list, max_length=10)

# 自动校验，自动报错
req = CreateUserRequest(
    name="Alice",
    email="alice@example.com",
    age=30,
    tags=["vip", "early-adopter"]
)
# 如果 email 格式不对，直接抛 ValidationError
# 如果 age=200，抛 ValidationError: age must be ≤ 150
```

这相当于 Spring Boot 中 `@NotBlank` + `@Email` + `@Min` + `@Max` + `@Size` 的组合，但全部用类型提示和 `Field` 参数表达，不需要注解满天飞。

---

#### OpenAPI 自动文档：零配置生成

Spring Boot 需要引入 `springdoc-openapi` 依赖、配置 `OpenAPI` Bean，才能生成 Swagger 文档。FastAPI 的文档生成是零配置的——启动时自动从路由装饰器和 Pydantic 模型提取信息，生成完整的 OpenAPI 3.1 Schema，挂载到两个可交互界面：

- `/docs`：Swagger UI，可以直接在浏览器里试调 API
- `/redoc`：ReDoc，更适合阅读的文档界面

这不是"额外功能"——它是类型提示驱动设计的自然产物。你已经声明了参数类型和 Pydantic 模型，FastAPI 只是把这些信息翻译成 OpenAPI JSON。运行时零开销，Schema 在启动时一次性生成。

---

#### ⚡ 性能贴士

**性能对比**（基于 TechEmpower 基准测试 Round 22，2024年）[^techempower]：

[^techempower]: TechEmpower Framework Benchmarks。FastAPI 官方文档引用 TechEmpower 基准测试，确认 FastAPI 运行在 Uvicorn 下是最快的 Python 框架之一，仅次于 Starlette 和 Uvicorn 本身。参见 https://fastapi.tiangolo.com/#performance

| 指标 | Spring Boot 3 | FastAPI (async) | Flask (Gunicorn) | 倍数差异 |
|------|--------------|-----------------|-------------------|----------|
| 简单JSON QPS | ~8,000 | ~38,000 | ~28,000 | FastAPI 4.8x Spring |
| 数据库查询 QPS | ~4,000 | ~18,000 | ~12,000 | FastAPI 4.5x Spring |
| P99延迟(100并发) | 25ms | 12ms | 18ms | FastAPI 2.1x Spring |
| 内存占用(空闲) | 250-400MB | 30-50MB | 30-50MB | FastAPI 5-8x 轻 |
| 启动时间 | 3-10s | 0.1s | 0.1s | FastAPI 30-100x 快 |

**测试环境说明**：上述数据为 TechEmpower 基准测试的典型数量级参考。实际性能受硬件、JDK 版本、Python 版本、数据库驱动、网络环境等多种因素影响。建议在目标部署环境上自行基准测试。

FastAPI 性能优势的底层原因：

- **ASGI 协程并发**：I/O 等待时不占线程，一个事件循环处理数百连接。Spring Boot 的线程模型在 I/O 密集场景下线程大部分时间在阻塞等待。
- **Pydantic Rust 内核**：数据校验和序列化在 Rust 层执行，不走 Python 解释器。FastAPI 0.130.0 引入 Pydantic Rust JSON 序列化后，序列化性能再提升约 2 倍。
- **零反射启动**：路由在装饰器执行时注册，不需要运行时扫描注解。OpenAPI Schema 启动时一次性生成，运行时零开销。

**优化建议**：

1. **生产环境用 Gunicorn + Uvicorn Worker**：`gunicorn main:app -w 4 -k uvicorn.workers.UvicornWorker`，多进程 + 每进程事件循环，兼顾并行性和并发性[^uvicorn-deploy]。
2. **同步操作用 `def`，异步操作用 `async def`**：不要在 `async def` 里调用同步阻塞函数，会卡住事件循环。FastAPI 会自动把 `def` 路由放到线程池执行。
3. **数据库用异步驱动**：如果用 `async def` 路由，配 `asyncpg` 或 `databases` 异步驱动，否则数据库操作仍是阻塞的。

[^uvicorn-deploy]: Uvicorn 官方部署文档，https://www.uvicorn.org/deployment/。推荐生产环境使用 Gunicorn 进程管理器 + Uvicorn Worker 类。

> 💡 性能口诀：**IO密集选async，CPU密集丢线程池；同步驱动配def路由，异步驱动配async路由。**

---

#### 踩坑日记

**坑1：async def 里调用同步数据库操作**

```python
# ❌ 错误写法：同步操作阻塞事件循环
@app.get("/users/{id}")
async def get_user(id: int):
    user = session.get(User, id)  # 同步阻塞！整个事件循环卡住
    return user

# ✅ 正确写法1：用 def，FastAPI 自动放线程池
@app.get("/users/{id}")
def get_user(id: int):
    user = session.get(User, id)
    return user

# ✅ 正确写法2：用异步驱动
@app.get("/users/{id}")
async def get_user(id: int):
    user = await async_session.get(User, id)  # 异步非阻塞
    return user
```

Java 老兵习惯 `@Autowired` 注入的 Service 是同步的，在 Spring Boot 的线程模型里没有问题。但 FastAPI 的 `async def` 在事件循环中运行，同步阻塞调用会冻结所有协程。

**坑2：Pydantic v1 → v2 迁移陷阱**

```python
# Pydantic v1 写法（FastAPI 0.126+ 已弃用）
class User(BaseModel):
    name: str

    @validator("name")  # ❌ v2 中改为 field_validator
    def validate_name(cls, v):
        return v.strip()

# Pydantic v2 写法
from pydantic import field_validator

class User(BaseModel):
    name: str

    @field_validator("name")
    @classmethod
    def validate_name(cls, v: str) -> str:
        return v.strip()

# v1: .dict() → v2: .model_dump()
# v1: .parse_obj() → v2: .model_validate()
# v1: Config 类 → v2: model_config 字典
```

如果你从旧版 FastAPI 升级，`pydantic.v1` 子模块可以在过渡期同时使用 v1 和 v2 模型，但最终需要全部迁移到 v2。

**坑3：Depends 的 yield 执行时机**

```python
def get_session():
    session = Session(engine)
    try:
        yield session
        session.commit()  # 在路由函数返回后、响应发送前执行
    except Exception:
        session.rollback()
        raise
    finally:
        session.close()  # 在响应发送后执行（默认 scope="request"）
```

`yield` 之后的代码不是在路由函数 `return` 时立即执行——默认在响应发送给客户端之后才执行 `finally`。如果你需要在响应发送前完成事务提交（比如保证客户端收到 200 时数据已落库），用 `scope="function"`：

```python
@app.get("/users/{id}")
def get_user(id: int, session: Session = Depends(get_session, scope="function")):
    ...
```

---

#### 思维实验：什么时候不该用 FastAPI

FastAPI 的极简哲学不是万能解药。以下场景，Spring Boot（或 Django）可能更合适：

- **重度事务系统**：需要分布式事务（JTA/XA）、跨服务事务协调（Seata），Spring 的 `@Transactional` 体系更成熟。FastAPI 的事务管理全靠手动 `session.commit()` / `session.rollback()`，没有声明式事务。
- **复杂 ORM 继承**：JPA 的 `@ManyToOne` + `@OneToMany` + fetch 策略 + 二级缓存，在复杂领域模型中非常强大。SQLAlchemy 虽然功能不弱，但 Python 生态中缺乏像 Hibernate 那样深度集成的全栈 ORM。
- **传统 CMS / 管理后台**：需要内置 Admin 界面、用户权限管理、内容工作流——Django Admin 开箱即用，Spring Boot 有 Spring Security + Admin 模块。FastAPI 在这方面几乎空白。
- **消息驱动架构**：Spring Integration + Spring Cloud Stream 提供完整的消息驱动企业集成模式。Python 生态有 Celery，但架构层次不同。
- **Java 团队既有基建**：如果公司已有 Spring Cloud 微服务体系（Nacos/Sentinel/Seata），强行用 FastAPI 会割裂技术栈。

> 🤔 思考题：如果 FastAPI 的"轻"来自 ASGI 协程模型，那 Java 有没有类似的异步 Web 框架？下一章我们将进入自动化脚本与 AI 入口，看看 Python 生态在另一个维度的"轻"——脚本自动化和 AI 集成，是如何让 Java 老兵重新思考"工程化"边界的。

---

#### 本章小结

1. **极简哲学**：FastAPI 用类型提示驱动一切——路由解析、数据校验、JSON序列化、文档生成，一次声明四种用途。Spring Boot 用注解+IoC容器驱动一切，功能更全但启动更重。—— 重装骑士穿盔甲要时间，轻量刺客拔刀就能出击。
2. **显式依赖**：`Depends` 是函数参数级别的显式注入，函数签名即依赖文档。`@Autowired` 是容器级别的隐式注入，需要看配置才知道来源。—— 刺客的盾挂在手臂上看得见，骑士的盔甲藏在容器里看不见。
3. **协程并发**：ASGI 事件循环让一个线程处理数百连接，Servlet 线程池一请求一线程。I/O 密集场景 FastAPI 性能碾压，但同步阻塞调用是致命陷阱。—— 护士分诊台一个护士管多人，银行柜台一个柜员等一人。
4. **Rust 内核**：Pydantic v2 用 Rust 重写校验引擎，速度提升 5-50 倍。FastAPI 0.130.0 引入 Rust JSON 序列化后再提速 2 倍。Python 的"慢"正在被 Rust 补上。—— 刺客的刀不是铁打的，是合金锻的。
5. **场景互补**：FastAPI 适合 API 优先、I/O 密集、快速迭代的场景。Spring Boot 适合企业级事务、复杂 ORM、成熟基建的场景。选型取决于场景，不是信仰。—— 刺客不适合攻城，骑士不适合刺探，各有所长。

---

[^fastapi-github]: FastAPI GitHub 仓库，https://github.com/fastapi/fastapi
[^starlette]: Starlette 官方文档，https://www.starlette.io/。Starlette 是 FastAPI 的底层 ASGI 框架。
[^pydantic-github]: Pydantic GitHub 仓库，https://github.com/pydantic/pydantic
[^sqlmodel-github]: SQLModel GitHub 仓库，https://github.com/fastapi/sqlmodel
[^wsgi-pep3333]: PEP 3333 — Python Web Server Gateway Interface v1.0.1，https://peps.python.org/pep-3333/
[^sebastian-typer]: Typer 官方文档，https://typer.tiangolo.com/。Sebastián Ramírez 同时创建了 FastAPI、SQLModel、Typer 和 Asyncer。
