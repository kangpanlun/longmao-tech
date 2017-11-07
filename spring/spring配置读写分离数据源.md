## spring配置读写分离数据源

#### 前言

我先介绍一下本次的主角`AbstractRoutingDataSource`,它是spring提供的一个动态数据源路由，我们可以将自己的数据源装配到他的`Map<Object, Object> targetDataSources`属性中，这里的key是我们自定义的master或slave，用户标记是主库还是存库。value就是具体的数据源实例。在使用时，只要使用注解的方式告诉Mapper应该使用哪个数据源即可。

使用方式例如：

```java
public interface UserMapper {
    @DataSource("master")
    public void addEntity(User user);
    @DataSource("slave")
    public User findEntityById(int id);
}
```

#### 一、先定义要需要的1个interface,3个class

1. DataSource接口，定义注解的

   ```java
   @Retention(RetentionPolicy.RUNTIME)
   @Target(ElementType.METHOD)
   public @interface DataSource {
       String value();
   }
   ```

   ​


2. 自定义`DynamicDataSource` ，继承`AbstractRoutingDataSource`,实现它的`determineCurrentLookupKey()`方法，这个方法是获取需要的数据源的方法。

```java
public class DynamicDataSource extends AbstractRoutingDataSource {
    @Override
    protected Object determineCurrentLookupKey() {
        // TODO Auto-generated method stub
        return DynamicDataSourceHolder.getDataSouce();
    }
}
```



3. `DynamicDataSourceHolder`这个类主要理解`ThreadLocal<String> holder`,它是用于保存本次调用需要使用哪个数据源。这个值是什么时候保存进去的呢？其实需要数第三个类`DataSourceAspect`。

```java
public class DynamicDataSourceHolder {
    public static final ThreadLocal<String> holder = new ThreadLocal<String>();
    public static void putDataSource(String name) {
        holder.set(name);
    }
    public static String getDataSouce() {
        return holder.get();
    }
}
```
4. `DataSourceAspect`这个类里定义了一个切面方法，这个切点切到了Mapper的方法上，根据Mapper方法注解数据源的值，将这个值（master或slave）设置到了`DynamicDataSourceHolder`的`ThreadLocal<String> holder`里。

```java
public class DataSourceAspect {
 public void before(JoinPoint point)
 {
     Object target = point.getTarget();
     String method = point.getSignature().getName();
     Class<?>[] classz = target.getClass().getInterfaces();
     Class<?>[] parameterTypes = ((MethodSignature) point.getSignature())
             .getMethod().getParameterTypes();
     try {
         Method m = classz[0].getMethod(method, parameterTypes);
         if (m != null && m.isAnnotationPresent(DataSource.class)) {
             DataSource data = m
                     .getAnnotation(DataSource.class);
             DynamicDataSourceHolder.putDataSource(data.value());
             System.out.println(data.value());
         }
     } catch (Exception e) {
         // TODO: handle exception
     }
 }
}
```



#### 二、定义需要的Model、Mapper

```java
public interface UserMapper {
    @DataSource("master")
    public void addEntity(User user);
    @DataSource("slave")
    public User findEntityById(int id);
}
```

```java

@Alias("User")
public class User implements Serializable
{
	private String userId = null;
	public Long getUserId()
	{
		return this.userId;
	}
	public void setUserId(Long userId)
	{
		this.userId = userId;
	}
}	
```

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="com.kang.mapper.UserMapper">
    <resultMap id="BaseResultMap" type="User">
        <id column="USER_ID" jdbcType="NUMERIC" property="userId" />
    </resultMap>

    <sql id="Base_Column_List">
        user_id
    </sql>

    <insert id="addEntity" parameterType="User">
        insert into USER (
        <trim prefix="" suffix=")" suffixOverrides=",">
            <if test="userId != null">
                USER_ID,
            </if>
        </trim>
        values (
        <trim prefix="" suffix=")" suffixOverrides=",">
            <if test="userId != null">
                #{userId,jdbcType=VARCHAR},
            </if>
        </trim>
    </insert>

    <select id="findEntityById"  resultMap="BaseResultMap">
        select * from USER where USER_ID = #{userId,jdbcType=NUMERIC};
    </select>
</mapper>
```

#### 三、最后是最重要的，也是最容易出错的application-context.xml

这里面分几个点：

1. 定义component-scan自动扫描的包。
2. 定义两个数据源，master和slave。
3. 定义`DynamicDataSource`的bean，将master和slave装配进来。
4. 配置事物管理
5. 配置`sqlSessionFactory`，注意这里面包含了mapper.xml的加载。
6. 配置`MapperScannerConfigurer`,将自定义的Mapper接口注入到`sqlSessionFactory`
7. 配置自定义的切面`DataSourceAspect`

具体配置参照如下：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:aop="http://www.springframework.org/schema/aop"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans-3.2.xsd
           http://www.springframework.org/schema/aop http://www.springframework.org/schema/aop/spring-aop-3.2.xsd

           http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context-3.2.xsd">
    <context:component-scan base-package="com.kang"/>
   
    <bean id="masterdataSource"
          class="org.springframework.jdbc.datasource.DriverManagerDataSource">
        <property name="driverClassName" value="com.mysql.jdbc.Driver" />
        <property name="url" value="jdbc:mysql://127.0.0.1:3306/pay" />
        <property name="username" value="root" />
        <property name="password" value="kpl123456" />
    </bean>

    <bean id="slavedataSource"
          class="org.springframework.jdbc.datasource.DriverManagerDataSource">
        <property name="driverClassName" value="com.mysql.jdbc.Driver" />
        <property name="url" value="jdbc:mysql://127.0.0.1:3306/read_pay" />
        <property name="username" value="root" />
        <property name="password" value="kpl123456" />
    </bean>

    <bean id="dataSource" class="com.kang.DynamicDataSource">
        <property name="targetDataSources">
            <map key-type="java.lang.String">
                <!-- write -->
                <entry key="master" value-ref="masterdataSource"/>
                <!-- read -->
                <entry key="slave" value-ref="slavedataSource"/>
            </map>

        </property>
        <property name="defaultTargetDataSource" ref="masterdataSource"/>
    </bean>

    <bean id="transactionManager"
          class="org.springframework.jdbc.datasource.DataSourceTransactionManager">
        <property name="dataSource" ref="dataSource" />
    </bean>

    <!-- 配置SqlSessionFactoryBean -->
    <bean id="sqlSessionFactory" class="org.mybatis.spring.SqlSessionFactoryBean">
        <property name="dataSource" ref="dataSource" />
        <!-- 自动配置mapper xml -->
        <property name="mapperLocations" value="classpath:mybatis/*.xml" />
        <!-- 自动配置别名 -->
        <property name="typeAliasesPackage" value="com.kang.model" />
    </bean>

    <!-- 配置数据库注解aop -->
    <aop:aspectj-autoproxy></aop:aspectj-autoproxy>
    <bean id="manyDataSourceAspect" class="com.kang.DataSourceAspect" />
    <aop:config>
        <aop:aspect id="c" ref="manyDataSourceAspect">
            <aop:pointcut id="tx" expression="execution(* com.kang.mapper.*.*(..))"/>
            <aop:before pointcut-ref="tx" method="before"/>
        </aop:aspect>
    </aop:config>
    <!-- 配置数据库注解aop -->

    <!-- DAO接口所在包名，Spring会自动查找其下的类 -->
    <bean class="org.mybatis.spring.mapper.MapperScannerConfigurer">
        <property name="basePackage" value="com.kang.mapper"/>
        <property name="sqlSessionFactoryBeanName" value="sqlSessionFactory"/>
    </bean>
</beans>
```

#### 总结

要想彻底理解这里面的原理，需要很好的理解`AbstractRoutingDataSource`的设计。一定要多动手，多写点demo，这样才能掌握。