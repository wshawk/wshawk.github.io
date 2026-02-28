---
title: Java接口参数校验
categories:
  - 后端
tags: 
  - Java
  - 参数校验
abbrlink: d842f1av
date: 2025-06-08 15:20:34
---
# Java接口参数校验

后端服务接口接收参数后，一般都需要根据业务规则进行参数的校验，没有接触过Validation的Java新手可能使用如下述代码来校验参数。

```
@RestController
@RequestMapping("/user")
public class UserController {
	
	@Resource
	private UserService userService;

	@PostMapping("/register")
    public ServerResult<String> registerUser(@RequestBody UserRegisterDTO userRegisterDTO) {
    	String username = userRegisterDTO.getUsername();
    	String password = userRegisterDTO.getPassword();

        if (username == null || username.isEmpty()) {
            throw new ParamInvalidException("用户名不能为空");
        }

        if (password == null || password.isEmpty()) {
            throw new ParamInvalidException("密码不能为空");
        }

        if (password.length() < 6) {
             throw new ParamInvalidException("密码长度至少为6位");
        }

        userService.register(userRegisterDTO)

        return ServerResult.success("用户注册成功");
    }
}

```

复杂的验证规则和多个参数的情况，手动编写校验逻辑会变得冗长、难以维护和复用。引入现代的校验框架如`Spring Validation`可以帮助解决这些问题，提供更高效、统一和可维护的参数校验方案。



## Bean Validation规范

- JSR303/JSR-349/JSR-380: JSR303(`Bean Validation`)是一项标准,只提供规范不提供实现，规定一些校验规范即校验注解，如@Null，@NotNull，@Pattern，位于javax.validation.constraints包下。JSR-349(`Bean Validation 1.1`)是其的升级版本，添加了一些新特性。JSR-380(`Bean Validation 2.0`,[JSR380标准](https://link.juejin.cn?target=https%3A%2F%2Fwww.jcp.org%2Fen%2Fegc%2Fview%3Fid%3D380) )对其规范进一步扩展和增强。
- `hibernate validation`：`hibernate validation`是对这个规范的实现，并增加了一些其他校验注解，如@Email，@Length，@Range等等
- `Spring validation`：`spring validation`对`hibernate validation`进行了二次封装，在springmvc模块中添加了自动校验，并将校验信息封装进了特定的类中

Bean Validation的主页：[beanvalidation.org](https://link.juejin.cn?target=http%3A%2F%2Fbeanvalidation.org%2F)
 Bean Validation的参考实现：[github.com/hibernate/h…](https://link.juejin.cn?target=https%3A%2F%2Fgithub.com%2Fhibernate%2Fhibernate-validator)

相关版本兼容性

| Bean Validation | Hibernate Validation | JDK  | Spring Boot |
| --------------- | -------------------- | ---- | ----------- |
| 1.1             | 5.4 +                | 6+   | 1.5.x       |
| 2.0             | 6.0 +                | 8+   | 2.0.x       |
| 3.0             | 7.0 +                | 9+   | 2.0.x       |

3.0后`Bean Validation`改名为`Jakarta Bean Validation 3.0`了。 如果你的项目版本是jdk1.8的，不要使用`hibernate-validator 7.0`的版本，它里面的依赖的`jakarta.validation-api:3.0`是需要jdk1.9的部分支持的。

## Spring Validation注解

Spring Validation建立在Java Bean Validation（JSR 380）的基础上，为开发人员提供了一组注解和工具，用于定义和执行数据验证规则。它允许开发人员在应用程序中定义验证规则，并使用这些规则来验证输入数据、请求参数、领域对象等。

@Validated：可以用在类型、方法和方法参数上。但是不能用在成员属性（字段）上

@Valid：可以用在方法、构造函数、方法参数和成员属性（字段）上

常用注解标签如下：

| 标签                      | 说明                                                         |
| ------------------------- | ------------------------------------------------------------ |
| @Null                     | 限制只能为null                                               |
| @NotNull                  | 限制必须不为null                                             |
| @AssertFalse              | 限制必须为false                                              |
| @AssertTrue               | 限制必须为true                                               |
| @DecimalMax(value)        | 限制必须为一个不大于指定值的数字                             |
| @DecimalMin(value)        | 限制必须为一个不小于指定值的数字                             |
| @Digits(integer,fraction) | 限制必须为一个小数，且整数部分的位数不能超过integer，小数部分的位数不能超过fraction |
| @Future                   | 限制必须是一个将来的日期                                     |
| @Max(value)               | 限制必须为一个不大于指定值的数字                             |
| @Min(value)               | 限制必须为一个不小于指定值的数字                             |
| @Past                     | 限制必须是一个过去的日期                                     |
| @Pattern(value)           | 限制必须符合指定的正则表达式                                 |
| @Size(max,min)            | 限制字符长度必须在min到max之间                               |
| @Past                     | 验证注解的元素值（日期类型）比当前时间早                     |
| @NotEmpty                 | 验证注解的元素值不为null且不为空（字符串长度不为0、集合大小不为0） |
| @NotBlank                 | 验证注解的元素值不为空（不为null、去除首位空格后长度为0），不同于@NotEmpty，@NotBlank只应用于字符串且在比较时会去除字符串的空格 |
| @Email                    | 验证注解的元素值是Email，也可以通过正则表达式和flag指定自定义的email格式 |

## hibernate-validator 校验

- pom引入hibernate-validator

```xml
<dependency>
    <groupId>org.hibernate.validator</groupId>
    <artifactId>hibernate-validator</artifactId>
    <version>6.2.0.Final</version>
</dependency>
```

- 创建一个Java Bean,我们校验一下用户名跟年龄

```java
public class User {

    @NotBlank(message = "用户名不能为空")
    private String username;

    @Min(value = 18, message = "年龄不能小于18岁")
    private int age;

    // 构造函数、Getter 和 Setter 方法
}
```

- 执行校验

```java
public class ValidatorTest {

    public static void main(String[] args) {
        // 创建校验器
        ValidatorFactory factory = Validation.buildDefaultValidatorFactory();
        Validator validator = factory.getValidator();

        // 创建用户对象
        User user = new User();
        user.setUsername("");
        user.setAge(16);

        // 执行校验
        Set<ConstraintViolation<User>> violations = validator.validate(user);

        // 处理校验结果
        if (!violations.isEmpty()) {
            for (ConstraintViolation<User> violation : violations) {
                System.out.println(violation.getMessage());
            }
        } else {
            System.out.println("校验通过");
        }
    }
}
```

## Spring Validation 校验

### Spring Validation引入

#### 添加pom依赖

`Spring Validation`校验包被独立成了一个`starter`组件，引入如下依赖:

```xml
dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-validation</artifactId>
</dependency>
```

如果是`spring-boot-starter-web`不用引入了，`spring-boot-starter-web` 集成了`spring-boot-starter-validation`，默认可以不加`spring-boot-starter-validation`，它同时也集成了`hibernate-validator`

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

#### hibernate Validator校验器

这里我们定义hibernate校验器用于校验参数

```java
@Configuration
@EnableAutoConfiguration
public class HibernateValidatorConfiguration {
	@Bean
	public MethodValidationPostProcessor methodValidationPostProcessor() {
		MethodValidationPostProcessor processor = new MethodValidationPostProcessor();
		processor.setValidator(validator());
		processor.setProxyTargetClass(true);
		return processor;
	}

	@Bean
	public Validator validator() {
		return Validation
				.byProvider(HibernateValidator.class)
				.configure()
				.addProperty("hibernate.validator.fail_fast", "true")
				.buildValidatorFactory()
				.getValidator();
	}
}
```

#### 全局异常处理

每个`Controller`方法中都如果都写一遍对校验结果信息的处理，使用起来还是很繁琐。可以通过全局异常处理的方式统一处理校验异常。

```java
@ControllerAdvice
@Slf4j
public class GlobalExceptionHandler {
    @ExceptionHandler(value = ConstraintViolationException.class)
	@ResponseBody
	public ApiResult defaultInsuranceExceptionHandler(ConstraintViolationException e) {
		log.error("校验错误:{}", e.getMessage());
		return ApiResult.error("error", e.getMessage());
	}


	@ExceptionHandler(value = MissingServletRequestParameterException.class)
	@ResponseBody
	public ApiResult handleMissingServletRequestParameter(MissingServletRequestParameterException ex) {
		String error = ex.getParameterName() + " 参数为空";
		return ApiResult.error("error", error);
	}
}
```

### Controller接口Bean校验

首先，假设我们有一个用户注册的请求对象 `UserRegistrationRequest`，其中包含用户名和密码字段。

```java
@Data
public class UserRegistrationRequest {
    @NotBlank(message = "用户名不能为空")
    private String username;

    @NotBlank(message = "密码不能为空")
    @Size(min = 6, message = "密码长度至少为6位")
    private String password;
}
```

我们使用了两个注解进行参数校验：

- `@NotBlank`：该注解用于验证字段不能为空或空格，并可以通过 `message` 属性指定验证失败时的错误消息。
- `@Size`：该注解用于验证字段的长度，我们指定了密码的最小长度为6，并通过 `message` 属性定义了验证失败时的错误消息。

接下来我们定义一个Controller接口

```java
@RestController
public class UserController {

    @PostMapping("/register")
    public ResponseEntity<String> registerUser(@Valid @RequestBody UserRegistrationRequest request) {
        // 处理用户注册逻辑
        return ResponseEntity.ok("用户注册成功");
    }
}
```

### @RequestParam 参数校验

首先需要将MethodValidationPostProcessor设置成cglib代理

```arduino
processor.setProxyTargetClass(true);
```

定义一个接口，我们需要对ID进行校验，`1<=id<=400`

```java
@Validated
public interface ValidClient {

	@GetMapping("/valid")
	String queryById(@RequestParam("id") @Min(1) @Max(400) Integer id);

}
```

controller实现

```java
@RestController
public class ValidController implements ValidClient {

	@Override
	public String queryById(Integer id) {
		return "id:" + id;
	}
}
```

### 分组校验

还是看上面那个Controller接口校验的例子，如果我们注册的时候需要校验用户名和密码，重置密码的时候只校验密码该怎么校验呢？这个时候就用到了分组校验了。

- 首先，我们需要定义一个新的分组，用于更新场景中的验证，例如

  ```
  UpdateGroup
  ```

  。

  ```java
  public interface UpdateGroup {
  }
  ```

- 接下来，我们需要在

  ```
  UserRegistrationRequest
  ```

  类的

  ```
  username
  ```

  字段上使用

  ```
  @Validated
  ```

  注解，并通过

  ```
  groups
  ```

  属性指定要应用的验证分组。

  ```java
  public class UserRegistrationRequest {
  
      @NotBlank(message = "用户名不能为空", groups = {RegistrationGroup.class})
      private String username;
  
      @NotBlank(message = "密码不能为空")
      @Size(min = 6, message = "密码长度至少为6位")
      private String password;
  
      // 其他字段和方法
  }
  ```

在上述示例中，我们将`@Validated`注解应用到`username`字段，并通过`groups`属性指定了`RegistrationGroup.class`，这意味着在注册场景中会进行校验。

- 我们可以使用

  ```
  @Validated
  ```

  注解来指定不要应用任何验证分组。

  ```java
  @RestController
   public class UserController {
  
       @PostMapping("/register")
       public ResponseEntity<String> registerUser(@Validated(RegistrationGroup.class) @RequestBody UserRegistrationRequest request) {
           // 处理用户注册逻辑
           return ResponseEntity.ok("用户注册成功");
       }
  
       @PostMapping("/update")
       public ResponseEntity<String> updateUser(@Validated(UpdateGroup.class) @RequestBody UserRegistrationRequest request) {
           // 处理用户更新逻辑
           return ResponseEntity.ok("用户更新成功");
       }
   }
  ```

在上述示例中，我们在`updateUser`方法中使用`@Validated(UpdateGroup.class)`来指定在更新场景中只执行`UpdateGroup`分组的验证规则。由于`username`字段上没有指定`groups`属性，所以在更新场景中将不会对`username`字段进行校验。



### @Valid和@Validated的区别

| 注解       | 来源                                                         | 位置                                         | 是否支持分组 | 是否支持嵌套验证 |
| ---------- | ------------------------------------------------------------ | -------------------------------------------- | ------------ | ---------------- |
| @Valid     | Java EE提供的标准注解，它是**JSR 303**规范的一部分，主要用于Hibernate Validation等场景。 | 用在方法、构造函数、方法参数和成员属性上     | 不支持       | 支持             |
| @Validated | 是Spring框架特有的注解，属于Spring的一部分，也是**JSR 303**的一个变种 | 用在类、方法和方法参数上，但不能用于成员属性 | 支持         | 不支持           |



## 自定义注解

Spring 的 validation 为我们提供了许多特性，几乎可以满足日常开发中绝大多数参数校验场景了。但是，一个好的框架一定是方便扩展的。有了扩展能力，就能应对更多复杂的业务场景，下面我们自定义一个日期格式校验的注解

### 定义注解接口

```java
@Documented
@Constraint(validatedBy = DateFormatValidator.class)
@Target({ElementType.METHOD, ElementType.FIELD, ElementType.ANNOTATION_TYPE, ElementType.CONSTRUCTOR, ElementType.PARAMETER, ElementType.TYPE_USE})
@Retention(RetentionPolicy.RUNTIME)
public @interface DateFormat {
 	//默认错误消息
 	String message() default "时间格式错误";

 	//分组
 	Class<?>[] groups() default {};

 	//默认日期格式
 	String formatter() default "yyyy-MM-dd";

 	//负载
 	Class<? extends Payload>[] payload() default {};

 	@Target({ElementType.METHOD, ElementType.FIELD, ElementType.ANNOTATION_TYPE, ElementType.CONSTRUCTOR, ElementType.PARAMETER, ElementType.TYPE_USE})
 	@Retention(RetentionPolicy.RUNTIME)
 	@Documented
 	@interface List {
 		DateFormat[] value();
 	}
 }
```

### 定义校验器

```java
public class DateFormatValidator implements ConstraintValidator<DateFormat, String> {
    protected String dateFormatter;

    @Override
    public void initialize(DateFormat constraintAnnotation) {
            this.dateFormatter = constraintAnnotation.formatter();
    }

    @Override
    public boolean isValid(String value, ConstraintValidatorContext constraintValidatorContext{
            if (StringUtils.isNotBlank(value)) {
                    try {
                            DateUtils.parseDate(value, dateFormatter);
                    } catch (Exception e) {
                            return false;
                    }
                    return true;
            }
            return false;
    }
}
```

自定义校验注解使用起来和官方注解没有区别，在需要的字段上添加相应注解即可。

```java
public class RequestParam {
    @DateFormat(message = "日期输入错误")
    private String beginDate;

    // 其他字段和方法
}
```



## 组合注解

将原有的多个校验注解组合成为一个校验注解，这样免去了进行个多个注解的麻烦。

```java
@Documented
@Target({METHOD,FIELD,ANNOTATION_TYPE,CONSTRUCTOR,PARAMETER})
@Retention(RUNTIME)
// Hibernate Validation中对JSR 303中所有的校验注解的校验逻辑进行了实现
// 此处不需要声明校验器 
@Constraint(validatedBy = {}) 
@Max(150)
@Min(1)
@Repeatable(ValidateGroup.List.class)
// 默认情况下，组合注解中的一个或多个子注解校验失败的情况下，会分别触发子注解各自错误报告，如果想要使用
// 组合注解中定义的错误信息，则添加该注解。添加之后只要组合注解中有至少一个子注解校验失败，则会生成组合
// 注解中定义的错误报告，子注解的错误信息被忽略。
@ReportAsSingleViolation 
public @interface ValidateGroup {
//  属性覆盖注解，其属性constraint用于指定要覆盖的属性所在的子注解类型，name用于指定要覆盖的属性的名称
//  使用当前组合注解的min属性覆盖Min子注解的value属性
    @OverridesAttribute(constraint = Min.class, name = "value") long min() default 0;
    @OverridesAttribute(constraint = Max.class,name = "value") long max() default 150L;
 
    String message() default "组合注解校验不正确";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
 
    @Documented
    @Target({METHOD,FIELD,ANNOTATION_TYPE,CONSTRUCTOR,PARAMETER})
    @Retention(RUNTIME)
    public @interface List{
        ValidateGroup[] value();
    }
}
```

当有多个属性需要覆盖的时候可以使用`@OverridesAttribute.List`。举例如下：

```java
    @OverridesAttribute.List( {
        @OverridesAttribute(constraint=Size.class, name="min"),
        @OverridesAttribute(constraint=Size.class, name="max") } )
    int size() default 5;
```





## 在Service中进行参数校验

```java
@Service
@AllArgsConstructor
public class TestService {
    // 注入validator
    private final SpringValidatorAdapter validator;

    public R<String> add(Param param) {
        // 获取validator对指定校验组的校验结果
        Set<ConstraintViolation<Param>> violations = validator.validate(param, param.getAddValidationGroups());

        if (!violations.isEmpty()) {
            String message = violations.stream().map(ConstraintViolation::getMessageTemplate)
                    .collect(Collectors.joining(";"));
            return R.fail(message);
        }

        return R.success("success");
    }
}
```



## 参考

- [半亩方塘立身](https://juejin.cn/post/7245230318423310373)
- [Spring基础系列-参数校验](https://www.cnblogs.com/V1haoge/p/9953744.html)

