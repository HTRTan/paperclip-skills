---
name: mybatis-schema
description: 提供 MyBatis Mapper XML 文件审查、ResultMap 设计、类型别名管理、动态 SQL 编写及注解与 XML 选型权衡的专业知识。包含代码模板、常见反模式识别及性能优化建议。
---

# MyBatis Schema 技能

本技能帮助您系统化地审查和优化 MyBatis 持久层代码，确保 Mapper XML 的可读性、可维护性及性能，并在注解与 XML 之间做出合理选择。

## 何时使用

- 审查 Mapper XML 文件中的 SQL 语句、ResultMap、参数映射
- 设计复杂的结果映射（关联、集合、鉴别器）
- 决定使用注解还是 XML 实现某个 Mapper 方法
- 编写或优化动态 SQL（`<if>`, `<foreach>`, `<choose>` 等）
- 配置类型别名、TypeHandler 或插件
- 排查 N+1 查询、延迟加载、结果映射错误
- 建立团队 MyBatis 使用规范

## 核心概念速查

| 概念 | 说明 | 关键要点 |
|------|------|----------|
| Mapper XML | 定义 SQL 映射的 XML 文件 | namespace 对应 Mapper 接口全限定名 |
| ResultMap | 定义数据库列与 Java 属性的映射关系 | 支持关联（association）、集合（collection）、鉴别器（discriminator） |
| 参数映射 | 将 Java 参数传递给 SQL | `#{}` 预编译（安全），`${}` 字符串替换（有注入风险） |
| 动态 SQL | 根据条件动态生成 SQL 片段 | `<if>`, `<choose>`, `<where>`, `<set>`, `<foreach>`, `<trim>` |
| SQL 片段 | 可复用的 SQL 片段 | `<sql id="...">` + `<include refid="..."/>` |
| TypeHandler | 自定义 Java 类型与 JDBC 类型的转换 | 处理枚举、JSON 等特殊类型 |
| 缓存 | 一级缓存（Session 级别）、二级缓存（Mapper 级别） | 二级缓存需谨慎，易产生脏读 |

## Mapper XML 基础结构

### 标准 XML 模板

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper
  PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
  "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="com.example.mapper.UserMapper">

    <!-- 结果映射 -->
    <resultMap id="BaseResultMap" type="User">
        <id column="id" property="id" />
        <result column="username" property="username" />
        <result column="email" property="email" />
        <result column="created_at" property="createdAt" jdbcType="TIMESTAMP"/>
    </resultMap>

    <!-- 基础列列表，用于复用 -->
    <sql id="Base_Column_List">
        id, username, email, created_at
    </sql>

    <!-- 查询单个 -->
    <select id="selectById" resultMap="BaseResultMap" parameterType="long">
        SELECT <include refid="Base_Column_List" />
        FROM users
        WHERE id = #{id}
    </select>

    <!-- 插入并返回自增主键 -->
    <insert id="insert" parameterType="User" useGeneratedKeys="true" keyProperty="id">
        INSERT INTO users (username, email, created_at)
        VALUES (#{username}, #{email}, #{createdAt})
    </insert>

    <!-- 更新 -->
    <update id="updateById" parameterType="User">
        UPDATE users
        SET username = #{username},
            email = #{email}
        WHERE id = #{id}
    </update>

    <!-- 删除 -->
    <delete id="deleteById" parameterType="long">
        DELETE FROM users WHERE id = #{id}
    </delete>
</mapper>
```

## ResultMap 设计规范

### 1. 基础映射

- **主键必须使用 `<id>`**，而非 `<result>`，以优化 MyBatis 的对象标识和缓存。
- **建议明确 `jdbcType`**，尤其是在列可为空的情况下，避免某些驱动无法推断类型。

### 2. 关联映射（一对一）

```xml
<resultMap id="OrderWithUserMap" type="Order" extends="BaseResultMap">
    <!-- 本表列 -->
    <result column="order_no" property="orderNo"/>
    <!-- 关联对象 -->
    <association property="user" javaType="User" resultMap="com.example.mapper.UserMapper.BaseResultMap"/>
    <!-- 或内联写法 -->
    <association property="user" javaType="User">
        <id column="user_id" property="id"/>
        <result column="username" property="username"/>
    </association>
</resultMap>
```

### 3. 集合映射（一对多）

```xml
<resultMap id="UserWithOrdersMap" type="User" extends="BaseResultMap">
    <collection property="orders" ofType="Order" resultMap="com.example.mapper.OrderMapper.BaseResultMap"/>
    <!-- 或内联，并指定嵌套查询（慎用，可能 N+1） -->
    <collection property="orders" ofType="Order" column="id" select="com.example.mapper.OrderMapper.selectByUserId"/>
</resultMap>
```

**注意**：使用 `select` 属性会导致 N+1 查询问题，除非开启延迟加载且业务可接受。通常推荐使用 JOIN 一次查询，再通过 `<collection>` 的嵌套结果映射自动填充集合。

### 4. 鉴别器（Discriminator）

根据列值映射不同的子类：

```xml
<resultMap id="VehicleMap" type="Vehicle">
    <id column="id" property="id"/>
    <discriminator javaType="int" column="vehicle_type">
        <case value="1" resultMap="CarMap"/>
        <case value="2" resultMap="TruckMap"/>
    </discriminator>
</resultMap>
```

## 动态 SQL 最佳实践

### 1. `<if>` 与 `<where>` 配合避免多余的 AND/OR

```xml
<select id="findByCondition" resultMap="BaseResultMap">
    SELECT <include refid="Base_Column_List"/>
    FROM users
    <where>
        <if test="username != null and username != ''">
            AND username LIKE CONCAT('%', #{username}, '%')
        </if>
        <if test="email != null">
            AND email = #{email}
        </if>
        <if test="status != null">
            AND status = #{status}
        </if>
    </where>
</select>
```

### 2. `<set>` 用于动态更新

```xml
<update id="updateSelective" parameterType="User">
    UPDATE users
    <set>
        <if test="username != null">username = #{username},</if>
        <if test="email != null">email = #{email},</if>
        <if test="updatedAt != null">updated_at = #{updatedAt},</if>
    </set>
    WHERE id = #{id}
</update>
```

### 3. `<foreach>` 处理 IN 条件或批量插入

```xml
<!-- 批量插入 -->
<insert id="batchInsert">
    INSERT INTO users (username, email) VALUES
    <foreach collection="list" item="user" separator=",">
        (#{user.username}, #{user.email})
    </foreach>
</insert>

<!-- IN 条件 -->
<select id="selectByIds" resultMap="BaseResultMap">
    SELECT <include refid="Base_Column_List"/>
    FROM users
    WHERE id IN
    <foreach collection="ids" item="id" open="(" close=")" separator=",">
        #{id}
    </foreach>
</select>
```

### 4. `<choose>` 多分支条件

```xml
<select id="findByType" resultMap="BaseResultMap">
    SELECT * FROM orders
    <where>
        <choose>
            <when test="type == 'pending'">
                AND status = 0
            </when>
            <when test="type == 'completed'">
                AND status = 1
            </when>
            <otherwise>
                AND status IN (0,1)
            </otherwise>
        </choose>
    </where>
</select>
```

### 5. `<bind>` 创建绑定变量

```xml
<select id="searchUsers" resultMap="BaseResultMap">
    <bind name="pattern" value="'%' + keyword + '%'"/>
    SELECT * FROM users WHERE username LIKE #{pattern}
</select>
```

### 6. `<trim>` 自定义前缀/后缀

```xml
<trim prefix="SET" suffixOverrides=",">
    <if test="name != null">name = #{name},</if>
    <if test="age != null">age = #{age},</if>
</trim>
```

## 类型别名与 TypeHandler

### 类型别名配置

在 `mybatis-config.xml` 中：

```xml
<typeAliases>
    <!-- 扫描包，默认别名是类名首字母小写 -->
    <package name="com.example.entity"/>
    <!-- 显式指定别名 -->
    <typeAlias type="com.example.entity.User" alias="User"/>
</typeAliases>
```

### 自定义 TypeHandler（如 JSON 字段）

```java
@MappedTypes(Address.class)
@MappedJdbcTypes(JdbcType.VARCHAR)
public class AddressTypeHandler extends BaseTypeHandler<Address> {
    @Override
    public void setNonNullParameter(PreparedStatement ps, int i, Address parameter, JdbcType jdbcType) throws SQLException {
        ps.setString(i, JSON.toJSONString(parameter));
    }
    @Override
    public Address getNullableResult(ResultSet rs, String columnName) throws SQLException {
        String json = rs.getString(columnName);
        return JSON.parseObject(json, Address.class);
    }
    // 其他方法...
}
```

注册：

```xml
<typeHandlers>
    <typeHandler handler="com.example.handler.AddressTypeHandler"/>
</typeHandlers>
```

## 注解 vs XML：权衡与选型

| 维度 | 注解 | XML |
|------|------|-----|
| 简单 CRUD | ✅ 简洁，代码即文档 | ❌ 需维护 XML 文件 |
| 动态 SQL | ❌ 难写（需用 `@SelectProvider` 等） | ✅ 原生支持，清晰 |
| 复杂结果映射 | ❌ `@Results` 冗长难读 | ✅ XML 结构清晰 |
| 可读性/维护性 | 简单语句好；复杂语句乱 | 分离 SQL 与 Java，便于 DBA 审查 |
| 复用 SQL 片段 | ❌ 无法跨方法复用 | ✅ `<sql>` + `<include>` |
| 调试 | 直接看接口 | 需定位 XML 文件 |
| 性能 | 无差别 | 无差别 |

### 选型建议

- **简单、固定 SQL**（单表 CRUD，无动态条件） → 注解。
- **动态 SQL、复杂结果映射、多表关联** → XML。
- **团队规范**：建议统一使用 XML（便于集中管理 SQL 和审查），或在简单场景下混用但保持一致性。
- **示例混合使用**：

```java
public interface UserMapper {
    // 注解用于简单查询
    @Select("SELECT * FROM users WHERE id = #{id}")
    User selectById(Long id);
    
    // XML 用于复杂动态 SQL
    List<User> findByCondition(UserQuery query);
}
```

## 常见反模式与问题

| 反模式 | 表现 | 正确做法 |
|--------|------|----------|
| `SELECT *` 且无 ResultMap | 映射不精确，性能浪费 | 明确列出列，使用 ResultMap |
| 在 `<if>` 中直接拼接字符串未用 `CONCAT` | 导致 SQL 注入风险或语法错误 | 使用 `CONCAT('%', #{keyword}, '%')` 或 `<bind>` |
| 滥用 `${}` | SQL 注入漏洞 | 尽量使用 `#{}`，仅动态表名/列名可用 `${}` 并做好白名单校验 |
| 关联查询的 `<collection>` 未使用 `ofType` | 运行时类型错误 | 指定 `ofType` |
| 二级缓存开启却未设置 `useCache="false"` | 脏数据 | 只对纯查询、不常更新的表开启，并设置合适的刷新间隔 |
| 参数为 `Map` 传递 | 代码可读性差，重构困难 | 使用 POJO 或 `@Param` 注解明确参数名 |
| 大表分页使用内存分页 | 性能极差 | 使用数据库 `LIMIT` 或分页插件（如 PageHelper） |

## 性能优化要点

1. **避免 N+1 查询**：使用 JOIN 一次性加载关联数据，配合 `resultMap` 的嵌套结果映射。
2. **开启延迟加载**（`lazyLoadingEnabled=true`），但谨慎使用，避免在事务外访问懒加载属性。
3. **使用 `fetchSize`** 控制结果集每次拉取的行数（针对大结果集）。
4. **批量操作使用 `batch` 执行器**：
   ```java
   SqlSession sqlSession = sqlSessionFactory.openSession(ExecutorType.BATCH);
   try {
       UserMapper mapper = sqlSession.getMapper(UserMapper.class);
       for (User user : users) {
           mapper.insert(user);
       }
       sqlSession.commit();
   } finally {
       sqlSession.close();
   }
   ```
5. **合理设置 `jdbcTypeForNull`** 避免 Oracle 等数据库报错。
6. **使用 `useGeneratedKeys` 获取自增主键**，比执行额外 `SELECT LAST_INSERT_ID()` 更高效。

## 审查清单

在审查 MyBatis Mapper 代码时，逐项确认：

- [ ] **XML namespace** 是否与 Mapper 接口全限定名一致？
- [ ] **SQL 语句** 中是否使用了 `#{}` 而非 `${}`（除表名/列名动态场景）？
- [ ] **ResultMap** 是否为主键列使用了 `<id>`？
- [ ] **动态 SQL** 是否有条件遗漏导致 `<where>` 中出现无意义 AND/OR？是否使用 `<set>` 而非直接写 SET？
- [ ] **`<foreach>`** 的 `collection` 是否正确对应参数名（若参数为 `@Param("ids")` 则应为 `ids`）？
- [ ] **关联映射** 是否存在 N+1 隐患？是否可改用 JOIN 一次性加载？
- [ ] **SQL 片段** 是否过度拆分影响可读性？
- [ ] **缓存配置** 是否合理？二级缓存是否在合适的 mapper 上开启？
- [ ] **参数传递** 是否使用 POJO 或 `@Param`，避免 `Map` 裸传？
- [ ] **批量操作** 是否使用了 `ExecutorType.BATCH`？
- [ ] **TypeHandler** 是否正确注册且线程安全？

## 故障排查速查表

| 现象 | 可能原因 | 解决方案 |
|------|----------|----------|
| 查询返回 null，但数据库有值 | ResultMap 列名与字段不匹配 | 使用 `AS` 别名或开启 `mapUnderscoreToCamelCase` |
| 更新/插入成功但主键未回填 | 未配置 `useGeneratedKeys` 或 `keyProperty` | 添加相应属性 |
| 动态 SQL 条件始终不生效 | 参数为 null 或空字符串判断错误 | 检查 test 表达式，如 `test="name != null and name != ''"` |
| `#{xxx}` 报错：没有 getter/setter | 参数类型不是 Map 也不是 POJO | 使用 `@Param` 注解或传 Map |
| 延迟加载在 session 关闭后失败 | 在事务外访问懒加载属性 | 调整事务边界，或提前调用 `getXxx()` 加载 |
| 批量插入性能差 | 未开启批处理，或 foreach 拼接的 SQL 过长 | 使用 `ExecutorType.BATCH`，或分多组执行 |
| `TypeHandler` 不生效 | 未注册或 Java 类型不匹配 | 检查 `@MappedTypes` 或配置文件 |

## 扩展资源

| 主题 | 链接 |
|------|------|
| MyBatis 官方文档 | [https://mybatis.org/mybatis-3/](https://mybatis.org/mybatis-3/) |
| XML 映射文件详解 | [https://mybatis.org/mybatis-3/sqlmap-xml.html](https://mybatis.org/mybatis-3/sqlmap-xml.html) |
| 动态 SQL | [https://mybatis.org/mybatis-3/dynamic-sql.html](https://mybatis.org/mybatis-3/dynamic-sql.html) |
| MyBatis 与 Spring 集成 | [https://mybatis.org/spring/](https://mybatis.org/spring/) |
| MyBatis-Plus 增强工具（简化 CRUD） | [https://baomidou.com/](https://baomidou.com/) |

---