### 业务背景

为了保护个人隐私信息，业务系统中需要对敏感信息，比如手机号、身份证号、地址等，进行加密存储。

### 项目现状

系统是一个比较标准的 MVC 结构，技术栈为 SpringBoot，MyBatis，MySQL。

### 解决方案

我们的解决方案要达到两个目的：业务无感知，效率要高。因此需要做两个决定

1. 使用什么样的算法对敏感信息进行加解密？AES + BASE64
2. 在哪一层对敏感信息进行加解密？DAO 层

MyBatis 提供了两种机制：插件和 TypeHandler，对比下来 TypeHandler 比较合适，因为它实现简单。

首先需要定义一个工具类 `AesUtil`，为了说明目的只使用 BASE64 算法。

```java
public class AesUtil {
    public static String encrypt(String src) {
        return Base64.getEncoder().encodeToString(src.getBytes(StandardCharsets.UTF_8));
    }

    public static String decrypt(String src) {
        byte[] bytes = Base64.getDecoder().decode(src);
        return new String(bytes, StandardCharsets.UTF_8);
    }
}
```

接下来我们实现 `AesEncryptTypeHandler` 类，它继承了 `BaseTypeHandler`。

```java
public class AesEncryptTypeHandler extends BaseTypeHandler<String> {
    @Override
    public void setNonNullParameter(PreparedStatement ps, int i, String parameter, JdbcType jdbcType) throws SQLException {
        ps.setString(i, AesUtil.encrypt(parameter));
    }

    @Override
    public String getNullableResult(ResultSet rs, String columnName) throws SQLException {
        String columnValue = rs.getString(columnName);
        return AesUtil.decrypt(columnValue);
    }

    @Override
    public String getNullableResult(ResultSet rs, int columnIndex) throws SQLException {
        String columnValue = rs.getString(columnIndex);
        return AesUtil.decrypt(columnValue);
    }

    @Override
    public String getNullableResult(CallableStatement cs, int columnIndex) throws SQLException {
        String columnValue = cs.getString(columnIndex);
        return AesUtil.decrypt(columnValue);
    }
}
```

然后在 `application.yml` 定义 TypeHandler 的包名，这样 MyBatis 才能找到它。

```yml
mybatis:
  type-handlers-package: me.acomma.groot
```

最后在需要加解密的敏感字段，比如在 `name` 字段，配置 TypeHandler

```xml
<resultMap id="userResult" type="me.acomma.groot.domain.user.User">
    ···
    <result property="name" column="name" typeHandler="me.acomma.groot.infrastructure.dao.AesEncryptTypeHandler"/>
    ···
</resultMap>

<insert id="insert" useGeneratedKeys="true" keyProperty="id">
    insert into t_user (
        ···
        name, 
        ···
    )
    values (
        ···
        #{name,typeHandler=me.acomma.groot.infrastructure.dao.AesEncryptTypeHandler}, 
        ···
    )
</insert>

<update id="update">
    update t_user
    set name = #{name,typeHandler=me.acomma.groot.infrastructure.dao.AesEncryptTypeHandler}
    where user_id = #{userId.id,jdbcType=BIGINT}
</update>

<select id="selectByUsername" resultMap="userResult">
    select *
    from t_user
    where username = #{username}
</select>
```

在 `<result>` 配置中，所有的字段都没有加 `jdbcType`，同时 `#{username}` 中也没有加 `jdbcType`，所以上面的配置将对所有 `String` 类型的字段应用 `AesEncryptTypeHandler` 而不是指定的 `name` 这一个字段，因此**所有的字段都应该加上 `jdbcType` 配置**。

另外，可以使用 JavaType 对 `AesEncryptTypeHandler` 施加限制，让它只对特殊的类型生效，其他都使用默认配置。

首先增加一个 JavaType 类 `AesEncrypt`。

```java
@Alias("AESEncrypt")
public class AesEncrypt {
}
```

接下来在 `AesEncryptTypeHandler` 类上增加 `@MappedTypes` 注解。

```java
@MappedTypes(AesEncrypt.class)
public class AesEncryptTypeHandler extends BaseTypeHandler<String> {
    ···
}
```

还需要在 `application.yml` 中配置别名的包名，这样 MyBatis 才能找到它。

```yml
mybatis:
  type-handlers-package: me.acomma.groot
  type-aliases-package: me.acomma.groot
```

`Mapper.xml` 就按上面的配置也没有问题，当然也可以在 `name` 字段的地方加上 `javaType="AesEncrypt"` 配置。


### 确定数据库字段长度

MySQL 数据库字段类型 `VARCHAR(n)` 表示最多可以存 `n` 个字符，在 UTF-8 编码下一个字符最多 `3` 个字节，因此 `VARCHAR(n)` 最多可以存 `3 * n` 个字节。

AES 加密后密文的字节长度（用 `N` 表示）和明文的字节长度（用 `n` 表示）的关系为 `N = (n / 16) * 16 + 16`，其中 `(n / 16)` 表示整除，举几个例子 `(1 / 16) = 0`，`(2 / 16) = 0`，`(15 / 16) = 0`，`(16 / 16) = 1`，`(17 / 16) = 1`。

一个字符串进行 BASE64 编码后，编码后的长度 `N` 与原字符串长度 `n` 的关系为 `N = ⌈n / 3⌉ * 4`，其中符号 `⌈ ⌉` 表示向上取整。

根据上面的说明可以写一个程序来计算 `VARCHAR(n)` 的字段需要扩展到的长度。

```java
public int N(int n) {
    int BYTE_N = n * 3;
    int AES_N = (BYTE_N / 16) * 16 + 16;
    int BASE64_N = (int) Math.ceil(AES_N / 3.0) * 4;
    return BASE64_N;
}
```

举几个计算例子

```
11  -> 64
20  -> 88
600 -> 2412
```

MySQL 的 `TO_BASE64` 函数每 `76` 个字符会增加一个换行符，举个例子

```
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
```

因此实际的长度还要加上换行符的个数。

对于一个有 `n` 个字符的字符串，换行符的个数计算函数为

```java
public int R(int n) {
    int CRLF_N = n / 76;
    return CRLF_N;
}
```

最终的长度为 `N(n) + R(N(n))`。

### 参考资料

1. [Mybatis 字段加解密](https://blog.csdn.net/weixin_43430525/article/details/85783053)，基于 TypeHandler 的处理方式
2. [Mybatis拦截结果集实现字段加密](https://blog.csdn.net/qq_37433657/article/details/105725020)，基于 Plugin 的处理方式
3. [drtrang/typehandlers-encrypt](https://github.com/drtrang/typehandlers-encrypt)，值得一读，自定义 javaType，只处理特殊字段，类型别名 + TypeHandler
4. [mybatis generator 自定义 TypeHandler 对数据库敏感字段进行加解密](https://www.link-nemo.com/Kira/article/detail.do?a=uafqEQnH1oW7RK3WFKP)，与 mybatis generator 结合使用
5. [Mybatis数据库字段加解密1-使用mysql自带加密方法](https://www.jianshu.com/p/a35c39f0886c)，值得一读
6. [Mybatis数据库字段加解密2-使用typeAlias实现](https://www.jianshu.com/p/31994b3f224b)，值得一读，类型别名 + TypeHandler
7. [MySQL + MyBatis + AES 数据库加密的一些坑](https://sq.sf.163.com/blog/article/177903091005186048)，值得一读，字段长度增长问题
8. [mybatis plus 实现敏感数据的加密](https://cloud.tencent.com/developer/article/1557019)
9. [使用拦截器进行数据加解密](https://blog.csdn.net/wizard_rp/article/details/79821671)
10. [使用mybatis的拦截器对实体类上的敏感字段进行加密解密](https://blog.csdn.net/weixin_41605123/article/details/97028210)
11. [辉锅锅 / dbsecurity](https://gitee.com/HuiGuoGuo/dbsecurity)，插件方式
12. [数据库字段加密解密-Mybatis简单实现](https://blog.csdn.net/ming1215919/article/details/112545354)
13. [基于Mybatis层面对敏感字段的加密](https://blog.csdn.net/qq_43207114/article/details/115729399)，插件方式
14. [Mybatis：敏感信息加密存入数据库、解密读出](https://blog.csdn.net/qq_33407429/article/details/112304055)，TypeHandler 方式
15. [Spring Boot Mybatis 优雅解决敏感信息加解密问题](https://ld246.com/article/1587991687126)，值得一读，类型别名 + TypeHandler
16. [使用mybatis的BaseTypeHandler来给敏感字段进行AES加密](https://www.cnblogs.com/java-spring/p/14676670.html)
17. [Mybatis的TypeHandler加解密数据实现](https://www.jb51.net/article/216085.htm)，把要加密的字段用一个类重新包装起来
18. [Jerry.hu/mybatis-cipher](https://gitee.com/Jerry.hu/mybatis-cipher)，插件方式
19. [数据加密存储——已上线应用平滑切换方案](https://juejin.cn/post/6864406731045535752)，和 ShardingSphere
20. [基于mybatis类型转换器实现数据加解密](https://blog.csdn.net/userwyh/article/details/80069276)
21. [Mybatis使用TypeHandler实现数据的加解密转换](https://www.cnblogs.com/wangjuns8/p/8688815.html)
22. [mybatis typehandler 不全局注册](https://segmentfault.com/q/1010000010117276)，值得一读
23. [mybatis 自定义typehandler，转换特定字段](https://blog.csdn.net/lw296196709/article/details/74370355)
24. [用过AES加密吗，为何加密后文件会变长](http://www.myexceptions.net/j2se/1171232.html)
25. [JAVA AES 加密后，结果的长度](https://bbs.csdn.net/topics/320179182)
26. [base64加密后字符串长度](https://blog.csdn.net/dliyuedong/article/details/17686457)
27. [AES密文与明文长度的关系](https://www.cnblogs.com/lori/p/14210066.html)
28. [【IoT】加密与安全：不同模式和填充下 AES 密文的长度](https://blog.csdn.net/liwei16611/article/details/86312599)
