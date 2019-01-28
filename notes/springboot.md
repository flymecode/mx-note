### Springboot 2.0

#### 数据源配置

在我们访问数据库的时候，需要先配置一个数据源，下面分别介绍以下几种不同的数据库配置方式。

首先，为了连接数据库需要引入jdbc的支持，在`pom.xml`中引入配置：

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-jdbc</artifactId>
</dependency>/
```

#### 连接生产数据源

以MySQL数据库为例，先引入MySQL连接的依赖包，在`pom.xml`中加入：

```xml
<dependency>
    <groupId>mysql</groupId>
    <artifactId>mysql-connector-java</artifactId>
    <version>5.1.21</version>
</dependency>
```

在`src/main/resources/application.properties`中配置数据源信息

```xml
spring.datasource.url=jdbc:mysql://localhost:3306/test
spring.datasource.username=dpuser
spring.datasource.password=dbpass
spring.datasource.driver-class-name=com.mysql.jdbc.Driver
```

#### 连接JNDI数据源

```xml
spring.datasource.jndi-name=java:jboss/datasources/customers
```

### 使用JdbcTemplate操作数据库

Spring的JdbcTemplate是自动配置的，你可以直接使用`@Autowired`来注入到你自己的bean中来使用。

举例：我们在创建`User`表，包含属性`name`、`age`，下面来编写数据访问对象和单元测试用例。

```java
public interface UserService {

    /**
     * 新增一个用户
     * @param name
     * @param age
     */
    void create(String name, Integer age);

    /**
     * 根据name删除一个用户高
     * @param name
     */
    void deleteByName(String name);

    /**
     * 获取用户总量
     */
    Integer getAllUsers();

    /**
     * 删除所有用户
     */
    void deleteAllUsers();

}
```

- 通过JdbcTemplate实现UserService中定义的数据访问操作

```java
@Service
public class UserServiceImpl implements UserService {

    @Autowired
    private JdbcTemplate jdbcTemplate;

    @Override
    public void create(String name, Integer age) {
        jdbcTemplate.update("insert into USER(NAME, AGE) values(?, ?)", name, age);
    }

    @Override
    public void deleteByName(String name) {
        jdbcTemplate.update("delete from USER where NAME = ?", name);
    }

    @Override
    public Integer getAllUsers() {
        return jdbcTemplate.queryForObject("select count(1) from USER", Integer.class);
    }

    @Override
    public void deleteAllUsers() {
        jdbcTemplate.update("delete from USER");
    }
}
```

- 创建对UserService的单元测试用例，通过创建、删除和查询来验证数据库操作的正确性。

```java
@RunWith(SpringJUnit4ClassRunner.class)
@SpringApplicationConfiguration(Application.class)
public class ApplicationTests {

	@Autowired
	private UserService userSerivce;

	@Before
	public void setUp() {
		// 准备，清空user表
		userSerivce.deleteAllUsers();
	}

	@Test
	public void test() throws Exception {
		// 插入5个用户
		userSerivce.create("a", 1);
		userSerivce.create("b", 2);
		userSerivce.create("c", 3);
		userSerivce.create("d", 4);
		userSerivce.create("e", 5);

		// 查数据库，应该有5个用户
		Assert.assertEquals(5, userSerivce.getAllUsers().intValue());

		// 删除两个用户
		userSerivce.deleteByName("a");
		userSerivce.deleteByName("e");

		// 查数据库，应该有5个用户
		Assert.assertEquals(3, userSerivce.getAllUsers().intValue());

	}

}
```

*上面介绍的JdbcTemplate只是最基本的几个操作，更多其他数据访问操作的使用请参考：[JdbcTemplate API](https://docs.spring.io/spring/docs/current/javadoc-api/org/springframework/jdbc/core/JdbcTemplate.html)*

通过上面这个简单的例子，我们可以看到在Spring Boot下访问数据库的配置依然秉承了框架的初衷：简单。我们只需要在pom.xml中加入数据库依赖，再到application.properties中配置连接信息，不需要像Spring应用中创建JdbcTemplate的Bean，就可以直接在自己的对象中注入使用。

转自[程序员DD](http://blog.didispace.com/springbootdata1/)



### Springboot 的异常处理

在日常web开发中发生了异常，往往是需要通过一个统一的异常处理来保证客户端能够收到友好的提示。

##### 单个控制器异常

@ExceptionHandler可用于控制器中，表示处理当前类的异常。

```
@ExceptionHandler(Exception.class)
public String exceptionHandler(Exception e) {
    log.error("---------------->捕获到局部异常", e);
    return "index";
}
```

##### 全局异常

如果单使用@ExceptionHandler，只能在当前Controller中处理异常。但当配合@ControllerAdvice一起使用的时候，就可以摆脱那个限制了。



可通过@ExceptionHandler的参数进行异常分类处理。



**步骤：**

第一步、自定义异常

```
public class MyException extends Exception {


    public MyException(String message) {
        super(message);
    }
}
```



第二步、定义全局处理器，主要是@ControllerAdvice和@ExceptionHandler注解的运用

```
@Slf4j
@ControllerAdvice
public class GlobalExceptionHandler {
    @ExceptionHandler(value = Exception.class)
    public ModelAndView defaultErrorHandler(HttpServletRequest req, Exception e)  {
        log.error("------------------>捕捉到全局异常", e);
        
        ModelAndView mav = new ModelAndView();
        mav.addObject("exception", e);
        mav.addObject("url", req.getRequestURL());
        mav.setViewName("error");
        return mav;
    }


    @ExceptionHandler(value = MyException.class)
    @ResponseBody
    public R jsonErrorHandler(HttpServletRequest req, MyException e)  {
        //TODO 错误日志处理
        return R.fail(e.getMessage(), "some data");
     }

```

### Mybatis配置多数据源
