---
name: springboot-dev
description: 提供 Spring Boot 应用开发的核心知识、项目结构模板、常用 Starter 选型、配置文件管理、注解使用、单元测试与集成测试、打包部署及生产就绪最佳实践。
---

# Spring Boot 开发技能

本技能帮助开发者高效构建 Spring Boot 应用程序，从项目初始化到生产部署，涵盖配置、数据访问、Web 层、安全、测试和监控等关键环节，并提供可复用的代码模板和最佳实践。

## 何时使用

- 初始化一个新的 Spring Boot 项目，选择依赖和版本
- 设计分层架构（Controller、Service、Repository）
- 配置多环境配置文件（application-{profile}.yml/properties）
- 使用 Spring Data JPA / MyBatis 进行数据访问
- 实现统一异常处理、参数校验、全局响应封装
- 编写单元测试和集成测试（@WebMvcTest、@DataJpaTest、@SpringBootTest）
- 打包并部署到服务器、容器（Docker）或云平台
- 开启 Actuator 进行应用监控和健康检查
- 解决常见开发问题（循环依赖、事务失效、懒加载等）

## 核心概念速查

| 概念 | 说明 | 关键注解/类 |
|------|------|------------|
| 自动配置 | 根据 classpath 依赖自动配置 Spring Bean | `@SpringBootApplication`, `@EnableAutoConfiguration` |
| Starter 依赖 | 一组预定义的依赖集合 | `spring-boot-starter-web`, `spring-boot-starter-data-jpa` |
| 配置文件 | 外部化配置（properties / yaml） | `application.yml`, `@ConfigurationProperties`, `@Value` |
| 嵌入式容器 | 内嵌 Tomcat/Jetty/Undertow | 无需部署 WAR，直接运行 JAR |
| Actuator | 生产就绪端点（健康、指标、信息） | `spring-boot-starter-actuator` |
| 分层测试 | 针对不同层级编写测试 | `@SpringBootTest`, `@WebMvcTest`, `@DataJpaTest` |
| 条件注解 | 基于条件注册 Bean | `@ConditionalOnClass`, `@ConditionalOnMissingBean` |

## 项目结构模板

```
src/
├── main/
│   ├── java/com/example/demo/
│   │   ├── DemoApplication.java              # 启动类
│   │   ├── controller/                        # 控制层
│   │   │   └── UserController.java
│   │   ├── service/                           # 业务层
│   │   │   ├── UserService.java
│   │   │   └── impl/
│   │   │       └── UserServiceImpl.java
│   │   ├── repository/                        # 数据访问层
│   │   │   └── UserRepository.java
│   │   ├── entity/                            # 实体类
│   │   │   └── User.java
│   │   ├── dto/                               # 数据传输对象
│   │   │   ├── request/
│   │   │   └── response/
│   │   ├── config/                            # 配置类
│   │   │   └── WebConfig.java
│   │   ├── exception/                         # 异常处理
│   │   │   ├── GlobalExceptionHandler.java
│   │   │   └── BusinessException.java
│   │   └── util/                              # 工具类
│   └── resources/
│       ├── application.yml                    # 主配置文件
│       ├── application-dev.yml                # 开发环境
│       ├── application-prod.yml               # 生产环境
│       ├── static/                            # 静态资源
│       └── templates/                         # 模板文件（若使用 thymeleaf）
└── test/
└── java/com/example/demo/
├── controller/                        # 控制器测试
├── service/                           # 服务层测试
└── repository/                        # 数据层测试
```

## 常用 Starter 选型指南

| Starter | 用途 | 典型场景 |
|---------|------|----------|
| `spring-boot-starter-web` | 构建 Web 应用（含 RESTful API） | 提供 Spring MVC、Tomcat、Jackson |
| `spring-boot-starter-data-jpa` | JPA 持久化（Hibernate） | 对象关系映射、自动分页、审计 |
| `spring-boot-starter-data-redis` | Redis 集成 | 缓存、分布式锁、会话共享 |
| `spring-boot-starter-security` | 安全框架 | 认证授权、CSRF、CORS |
| `spring-boot-starter-validation` | 参数校验 | Hibernate Validator，`@Valid` |
| `spring-boot-starter-actuator` | 生产监控 | 健康检查、指标、环境信息 |
| `spring-boot-starter-test` | 测试依赖 | JUnit 5、Mockito、AssertJ |
| `spring-boot-starter-aop` | 面向切面 | 日志、权限、事务管理 |
| `mybatis-spring-boot-starter` | MyBatis 集成 | 灵活 SQL 映射 |
| `spring-boot-starter-amqp` | RabbitMQ | 消息队列 |
| `spring-boot-starter-mail` | 邮件发送 | 通知、告警 |

## 配置文件最佳实践

### 多环境配置

```yaml
# application.yml
spring:
  profiles:
    active: dev
  application:
    name: demo-service

---
# application-dev.yml
spring:
  config:
    activate:
      on-profile: dev
  datasource:
    url: jdbc:mysql://localhost:3306/dev_db
    username: dev_user
    password: dev_pass
logging:
  level:
    com.example: DEBUG
---
# application-prod.yml
spring:
  config:
    activate:
      on-profile: prod
  datasource:
    url: jdbc:mysql://prod-db:3306/prod_db
    username: ${DB_USER}
    password: ${DB_PASSWORD}
logging:
  level:
    com.example: WARN
```

### 配置绑定

```java
// 使用 @ConfigurationProperties
@Component
@ConfigurationProperties(prefix = "app.storage")
@Data
public class StorageProperties {
    private String location;
    private int maxSize;
}

// 在 application.yml 中配置
app:
  storage:
    location: /var/data
    maxSize: 10485760
```

### 常见配置项

```yaml
server:
  port: 8080
  servlet:
    context-path: /api
  tomcat:
    max-connections: 10000
    threads:
      max: 200

spring:
  jackson:
    date-format: yyyy-MM-dd HH:mm:ss
    time-zone: Asia/Shanghai
  mvc:
    throw-exception-if-no-handler-found: true
  web:
    resources:
      add-mappings: false
```

## 常用注解速查

| 注解 | 作用 | 示例 |
|------|------|------|
| `@SpringBootApplication` | 启动类组合注解 | `@SpringBootApplication` |
| `@RestController` | REST 控制器 | `@RestController` + `@RequestMapping` |
| `@Service` | 业务层 Bean | `@Service public class UserService` |
| `@Repository` | 数据层 Bean | `@Repository public interface UserRepository` |
| `@Autowired` / `@Resource` | 依赖注入 | `@Autowired private UserService userService` |
| `@Value` | 注入配置值 | `@Value("${app.name}") String appName` |
| `@ConfigurationProperties` | 批量绑定配置 | `@ConfigurationProperties(prefix = "app")` |
| `@ConditionalOnProperty` | 条件配置 | `@ConditionalOnProperty(name = "feature.enabled", havingValue = "true")` |
| `@Transactional` | 声明式事务 | `@Transactional(rollbackFor = Exception.class)` |
| `@Valid` + `@RequestBody` | 参数校验 | `public Result create(@Valid @RequestBody UserDto dto)` |
| `@ExceptionHandler` | 全局异常处理 | `@ExceptionHandler(BusinessException.class)` |
| `@RestControllerAdvice` | 统一异常/响应处理 | 结合 `@ExceptionHandler` |
| `@DataJpaTest` | JPA 切片测试 | `@DataJpaTest` |
| `@WebMvcTest` | Controller 层测试 | `@WebMvcTest(UserController.class)` |

## Web 开发模板

### 统一响应封装

```java
@Data
@Builder
public class Result<T> {
    private int code;
    private String message;
    private T data;
    private long timestamp = System.currentTimeMillis();

    public static <T> Result<T> success(T data) {
        return Result.<T>builder().code(200).message("success").data(data).build();
    }

    public static <T> Result<T> error(int code, String message) {
        return Result.<T>builder().code(code).message(message).build();
    }
}
```

### 全局异常处理

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public Result<?> handleValidationException(MethodArgumentNotValidException e) {
        String errorMsg = e.getBindingResult().getAllErrors().stream()
                .map(DefaultMessageSourceResolvable::getDefaultMessage)
                .collect(Collectors.joining("; "));
        return Result.error(400, errorMsg);
    }

    @ExceptionHandler(BusinessException.class)
    public Result<?> handleBusinessException(BusinessException e) {
        return Result.error(e.getCode(), e.getMessage());
    }

    @ExceptionHandler(Exception.class)
    public Result<?> handleException(Exception e) {
        log.error("系统异常", e);
        return Result.error(500, "服务器内部错误");
    }
}
```

### 参数校验

```java
@PostMapping("/user")
public Result<User> createUser(@Valid @RequestBody UserCreateRequest request) {
    // ...
}

public class UserCreateRequest {
    @NotBlank(message = "用户名不能为空")
    @Size(min = 3, max = 20)
    private String username;

    @Email(message = "邮箱格式不正确")
    @NotBlank
    private String email;

    @Pattern(regexp = "^1[3-9]\\d{9}$", message = "手机号格式不正确")
    private String phone;
}
```

## 数据访问模板（Spring Data JPA）

```java
// Entity
@Entity
@Table(name = "users")
@Data
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String username;
    private String email;
    @CreationTimestamp
    private LocalDateTime createdAt;
}

// Repository
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    Optional<User> findByUsername(String username);
    
    @Query("SELECT u FROM User u WHERE u.email LIKE %:domain%")
    List<User> findByEmailDomain(@Param("domain") String domain);
}

// Service
@Service
@Transactional
public class UserService {
    @Autowired
    private UserRepository userRepository;
    
    public User createUser(User user) {
        return userRepository.save(user);
    }
    
    @Transactional(readOnly = true)
    public User findByUsername(String username) {
        return userRepository.findByUsername(username)
            .orElseThrow(() -> new BusinessException("用户不存在"));
    }
}
```

## 测试模板

### 单元测试（Service 层）

```java
@ExtendWith(MockitoExtension.class)
class UserServiceTest {
    @Mock
    private UserRepository userRepository;
    
    @InjectMocks
    private UserService userService;
    
    @Test
    void testFindByUsername_UserExists_ReturnsUser() {
        User mockUser = new User();
        mockUser.setUsername("john");
        when(userRepository.findByUsername("john")).thenReturn(Optional.of(mockUser));
        
        User result = userService.findByUsername("john");
        assertThat(result.getUsername()).isEqualTo("john");
    }
}
```

### 切片测试（Controller 层）

```java
@WebMvcTest(UserController.class)
class UserControllerTest {
    @Autowired
    private MockMvc mockMvc;
    
    @MockBean
    private UserService userService;
    
    @Test
    void testGetUser() throws Exception {
        User user = new User();
        user.setUsername("john");
        when(userService.findByUsername("john")).thenReturn(user);
        
        mockMvc.perform(get("/api/users/john"))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.data.username").value("john"));
    }
}
```

### 集成测试

```java
@SpringBootTest
@AutoConfigureMockMvc
@Transactional   // 测试后回滚
class IntegrationTest {
    @Autowired
    private MockMvc mockMvc;
    
    @Test
    void testCreateAndQueryUser() throws Exception {
        // 发送 POST 创建用户
        // 发送 GET 验证用户存在
    }
}
```

## 生产就绪：Actuator 与监控

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
  endpoint:
    health:
      show-details: always
  metrics:
    export:
      prometheus:
        enabled: true
```

常用端点：
- `/actuator/health` - 健康检查
- `/actuator/info` - 应用信息
- `/actuator/metrics` - 指标（JVM、CPU、请求）
- `/actuator/env` - 环境属性

## 打包与部署

### Maven 构建

```bash
# 打包（包含内置 Tomcat）
mvn clean package

# 运行 JAR
java -jar target/demo-0.0.1-SNAPSHOT.jar --spring.profiles.active=prod

# 使用外部配置文件
java -jar app.jar --spring.config.location=file:/config/application.yml
```

### Docker 镜像

```dockerfile
FROM openjdk:17-jdk-slim
VOLUME /tmp
COPY target/demo.jar app.jar
ENTRYPOINT ["java", "-jar", "/app.jar"]
```

### 环境变量覆盖配置

```bash
java -jar app.jar --server.port=9090 --spring.datasource.url=jdbc:mysql://...
# 或使用环境变量（Spring Boot 支持 relaxed binding）
export SERVER_PORT=9090
export SPRING_DATASOURCE_URL=jdbc:mysql://...
```

## 常见问题与解决方案

| 问题 | 原因 | 解决方案 |
|------|------|----------|
| 循环依赖 | 两个 Bean 互相注入 | 使用 `@Lazy` 或构造器注入，或重新设计依赖关系 |
| `@Transactional` 失效 | 同类内部调用（无代理） | 注入自身或使用 `AopContext.currentProxy()` |
| 懒加载异常 | 在事务外访问延迟加载属性 | 在事务内调用 getter，或使用 `@EntityGraph` 预加载 |
| 配置文件不生效 | profile 未激活或优先级问题 | 检查 `spring.profiles.active`，确认配置位置顺序 |
| 测试启动慢 | 加载了整个上下文 | 使用切片测试（`@WebMvcTest`, `@DataJpaTest`） |
| 端口已被占用 | 其他进程占用 | `lsof -i:8080` 查找并停止，或更改 `server.port` |

## 最佳实践总结

1. **版本管理**：使用 Spring Boot 官方支持的版本（当前稳定版 3.x 需 JDK 17+）。
2. **构造器注入**：优于字段注入（`@Autowired`），便于单元测试和不可变性。
3. **日志规范**：使用 SLF4J + Logback，避免 `System.out`，按环境调整日志级别。
4. **API 版本控制**：通过 URL 路径（`/v1/users`）或请求头实现。
5. **配置外部化**：敏感信息使用环境变量或配置中心，不硬编码。
6. **异步处理**：耗时任务使用 `@Async` 配合线程池，避免阻塞主线程。
7. **安全**：启用 CSRF（针对浏览器），使用 HTTPS，对输入进行校验和过滤。
8. **文档**：集成 SpringDoc OpenAPI（原 Swagger）自动生成 API 文档。
9. **健康检查**：暴露 `/actuator/health` 给容器探针使用。
10. **优雅停机**：配置 `server.shutdown=graceful` 和 `spring.lifecycle.timeout-per-shutdown-phase=30s`。

## 扩展资源

| 主题 | 链接 |
|------|------|
| Spring Boot 官方文档 | [https://spring.io/projects/spring-boot](https://spring.io/projects/spring-boot) |
| Spring Initializr（项目生成） | [https://start.spring.io/](https://start.spring.io/) |
| Spring Boot 参考指南（中文） | [https://springdoc.cn/spring-boot/](https://springdoc.cn/spring-boot/) |
| Spring Data JPA 文档 | [https://spring.io/projects/spring-data-jpa](https://spring.io/projects/spring-data-jpa) |
| Spring Security 官方指南 | [https://spring.io/guides/topicals/spring-security-architecture/](https://spring.io/guides/topicals/spring-security-architecture/) |
| Baeldung Spring Boot 教程 | [https://www.baeldung.com/spring-boot](https://www.baeldung.com/spring-boot) |
