---
title: Spring Bean
categories:
  - 后端
tags:
  - Spring
  - 参数校验
abbrlink: cb377744
date: 2025-09-05 23:30:31
---

# Spring Validation 自定义注解案例

## 通用枚举类校验

### @EnumValid.java

```java
import javax.validation.Constraint;
import javax.validation.Payload;
import java.lang.annotation.Documented;
import java.lang.annotation.ElementType;
import java.lang.annotation.Repeatable;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Repeatable(EnumValid.List.class)
@Target({ElementType.FIELD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Constraint (validatedBy = { IntegerEnumValidator.class, StringEnumValidator.class, ListStringEnumValidator.class })
public @interface EnumValid {
    /**
     * 提示信息
     * @return
     */
    String message() default "Invalid enum value";

    /**
     * 枚举类
     * @return
     */
    Class<? extends Enum<?>> enumClass();

    /**
     * 校验分组
     * @return
     */
    Class<?>[] groups() default {};

    Class<? extends Payload>[] payload() default {};

    /**
     * 获取枚举code的方法名 <br>
     * 默认为 getType
     * @return
     */
    String method() default "getType";

    /**
     * 默认必填
     * @return
     */
    boolean required() default true;

    @Documented
    @Target({ElementType.FIELD})
    @Retention(RetentionPolicy.RUNTIME)
    @interface List
    {
        EnumValid[] value();
    }
}
```



### BaseEnumValidator.java

```java
import java.lang.reflect.InvocationTargetException;
import java.lang.reflect.Method;
import java.util.HashSet;
import java.util.List;
import java.util.Set;

public class BaseEnumValidator {

    public static boolean isInEnum(EnumValid annotation, String value) throws
            NoSuchMethodException, InvocationTargetException, IllegalAccessException {
        Object[] objects = getEnumConstants(annotation);
        Method method = getEnumMethod(annotation);
        for (Object o : objects) {
            if (value.equals(method.invoke(o))) {
                return true;
            }
        }
        return false;
    }

    public static boolean isInEnum(EnumValid annotation, Integer value) throws
            NoSuchMethodException, InvocationTargetException, IllegalAccessException {
        Object[] objects = getEnumConstants(annotation);
        Method method = getEnumMethod(annotation);
        for (Object o : objects) {
            if (value.equals(method.invoke(o))) {
                return true;
            }
        }
        return false;
    }

    public static boolean isInEnumByListString(EnumValid annotation, List<String> valueList) throws Exception {
        Object[] objects = getEnumConstants(annotation);
        Method method = getEnumMethod(annotation);

        Set<String> set = new HashSet<>();
        for (Object o : objects) {
            set.add((String) method.invoke(o));
        }

        for (String value : valueList) {
            if (!set.contains(value)) {
                return false;
            }
        }
        return true;
    }

    public static boolean isInEnumByListInteger(EnumValid annotation, List<Integer> valueList) throws Exception {
        Object[] objects = getEnumConstants(annotation);
        Method method = getEnumMethod(annotation);

        Set<Integer> set = new HashSet<>();
        for (Object o : objects) {
            set.add((Integer) method.invoke(o));
        }

        for (Integer value : valueList) {
            if (!set.contains(value)) {
                return false;
            }
        }
        return true;
    }

    private static Object[] getEnumConstants(EnumValid annotation) {
        return annotation.enumClass().getEnumConstants();
    }

    private static Method getEnumMethod(EnumValid annotation) throws NoSuchMethodException {
        return annotation.enumClass().getMethod(annotation.method());
    }
}
```



### IntegerEnumValidator.java

```java
import lombok.extern.slf4j.Slf4j;

import javax.validation.ConstraintValidator;
import javax.validation.ConstraintValidatorContext;

@Slf4j
public class IntegerEnumValidator implements ConstraintValidator<EnumValid, Integer> {
    private EnumValid annotation;

    @Override
    public void initialize(EnumValid constraintAnnotation) {
        this.annotation = constraintAnnotation;
    }

    @Override
    public boolean isValid(Integer value, ConstraintValidatorContext constraintValidatorContext) {
        if (value == null) {
            return !annotation.required();
        }
        try {
            return BaseEnumValidator.isInEnum(annotation, value);
        } catch (Exception e) {
            log.warn("EnumValidator catch an exception: ", e);
            return false;
        }
    }
}
```



### StringEnumValidator.java

```java
import lombok.extern.slf4j.Slf4j;

import javax.validation.ConstraintValidator;
import javax.validation.ConstraintValidatorContext;
import java.util.Objects;

@Slf4j
public class StringEnumValidator implements ConstraintValidator<EnumValid, String> {
    private EnumValid annotation;

    @Override
    public void initialize(EnumValid constraintAnnotation) {
        this.annotation = constraintAnnotation;
    }

    @Override
    public boolean isValid(String value, ConstraintValidatorContext constraintValidatorContext) {
        if (Objects.isNull(value)) {
            return !annotation.required();
        }
        try {
            return BaseEnumValidator.isInEnum(annotation, value);
        } catch (Exception e) {
            log.warn("EnumValidator catch an exception: ", e);
            return false;
        }
    }
}
```





### ListStringEnumValidator.java

```java
import lombok.extern.slf4j.Slf4j;
import org.springframework.util.CollectionUtils;

import javax.validation.ConstraintValidator;
import javax.validation.ConstraintValidatorContext;
import java.util.List;

/**
 * @author WENSONG1
 */
@Slf4j
public class ListStringEnumValidator implements ConstraintValidator<EnumValid, List<String>> {
    private EnumValid annotation;

    @Override
    public void initialize(EnumValid constraintAnnotation) {
        this.annotation = constraintAnnotation;
    }

    @Override
    public boolean isValid(List<String> valueList, ConstraintValidatorContext constraintValidatorContext) {
        if (CollectionUtils.isEmpty(valueList)) {
            return !annotation.required();
        }
        try {
            return BaseEnumValidator.isInEnumByListString(annotation, valueList);
        } catch (Exception e) {
            log.warn("EnumValidator catch an exception: ", e);
            return false;
        }
    }
}
```



