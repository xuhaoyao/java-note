## **使用 `#{} `占位符**

MyBatis 中最推荐使用的防止 SQL 注入的方法是 **`#{} `占位符**，而不是 **`${}` 占位符**。

- **`#{} `**占位符会使用 **PreparedStatement**，将参数作为 SQL 语句的参数传递，并自动进行参数绑定和转义。
  - MyBatis 使用 `#{} `占位符时，底层实际使用的是 `PreparedStatement`，它能够将输入参数和 SQL 语句进行分离，从而防止 SQL 注入。
- **`${}`** 占位符是直接拼接字符串的方式，不会进行参数转义，存在 SQL 注入风险。

**示例：使用 `#{}`**

```xml
<select id="getUserByName" parameterType="String" resultType="User">
    SELECT * FROM users WHERE username = #{username}
</select>
```

**解释：**

- 在上述 SQL 语句中，`#{username}` 是 MyBatis 的占位符，会使用 **PreparedStatement** 进行参数绑定，防止 SQL 注入。
- 用户输入的 `username` 会被当作参数传递，不会直接插入 SQL 字符串中，避免了 SQL 注入的风险。

**示例：使用 `${}`（不推荐）**

```xml
<select id="getUserByName" parameterType="String" resultType="User">
    SELECT * FROM users WHERE username = '${username}'
</select>
```

**解释：**

在这种情况下，`username` 的值会直接拼接到 SQL 语句中，如果 `username` 包含恶意 SQL 代码，则会发生 SQL 注入。例如，输入 `admin' OR '1'='1`，会执行：

```sql
SELECT * FROM users WHERE username = 'admin' OR '1'='1'
```

## 动态SQL构建

如果需要动态构建 SQL 语句，MyBatis 提供了 `<if>`, `<choose>`, `<trim>`, `<foreach>` 等动态 SQL 元素，避免直接使用字符串拼接。

```xml
<select id="getUserByCondition" parameterType="map" resultType="User">
    SELECT * FROM users
    <where>
        <if test="username != null">
            username = #{username}
        </if>
        <if test="age != null">
            AND age = #{age}
        </if>
    </where>
</select>

```

### 利用前端传过来的字段排序

当我们需要根据前端传递的字段进行排序时，如果直接将用户输入的字段名拼接到 SQL 语句中，可能会导致 **SQL 注入** 问题。攻击者可以通过构造恶意字段名（如 `id; DROP TABLE users`）来执行恶意 SQL 语句。为了防止这种情况，MyBatis 或 SQL 查询构建时需要特别注意处理排序字段。

如果需要使用 MyBatis 动态 SQL，可以结合 `<choose>`, `<when>`, `<otherwise>` 标签，实现基于白名单的排序字段选择。

```xml
<select id="getUsers" resultType="User">
    SELECT * FROM users
    <choose>
        <when test="sortField == 'username'">
            ORDER BY username ${sortOrder}
        </when>
        <when test="sortField == 'age'">
            ORDER BY age ${sortOrder}
        </when>
        <when test="sortField == 'created_at'">
            ORDER BY created_at ${sortOrder}
        </when>
        <otherwise>
            ORDER BY id ${sortOrder}
        </otherwise>
    </choose>
</select>

```

