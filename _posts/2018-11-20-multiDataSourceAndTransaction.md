---
layout: post
title: 多数据源方案及事务管理
date: 2018-11-20
tags: 多数据源事务
---
### 事件背景及需求
　　项目中使用两个或多个数据源，切在一个`service`中有操作(`insert`,`update`,`delete`)两个数据源的行为,其实这本身并不难，而且网上随便一搜就有一大堆解决方案,但是经我自己验证大部分的方案
在不考虑事务的情况下没问题，但是一旦加了事务(这里单指spring管理的事务)就有各种问题:数据源切换失败或者事务不起作用。其实要解决这个问题，首先要明白正常的情况下应该如何管理事务，下面结合具
体代码分析一下。

###分库方案描述
　　这里采用了网上用的较多的spring的`AbstractRoutingDataSource`这个动态数据源结合自定义注解+`aop`切面进行自动路由。

数据源枚举类型(省略getter and setter)
```
public enum DataSourceType {

    DS_A("ds_a"),

    DS_B("ds_b") ;
    
    private String type;

    DataSourceType(String type) {
        this.type = type;
    }
}
```
动态数据源
```
public class DynamicDataSource extends AbstractRoutingDataSource {

    @Override
    protected Object determineCurrentLookupKey() {
        return DynamicDataSourceHolder.getDataSource();
    }
```
注册动态数据源
```
    @Bean("ds_a")
    public DataSource dataSourceA() {
        return new DruidDataSource();
    }

    @Bean("ds_b")
    public DataSource dataSourceB() {
        return new DruidDataSource();
    }
    
    @Primary
    @Bean("dynamicDataSource")
    public DataSource dynamicDataSource() {
        DynamicDataSource dynamicDataSource = new DynamicDataSource();
        //key为数据源标识，value为具体数据源
        Map<Object, Object> dataSources = Maps.newHashMap();
        dataSources.put(DataSourceType.DS_A.getType(), dataSourceA());
        dataSources.put(DataSourceType.DS_B.getType(), dataSourceB());
        dynamicDataSource.setTargetDataSources(dataSources);
        //默认数据源
        dynamicDataSource.setDefaultTargetDataSource(dataSourceA());
        return dynamicDataSource;
    }
```
数据源上下文holder(记录/切换数据源)
```
public class DynamicDataSourceHolder {

    private static final ThreadLocal<String> contextHolder = new ThreadLocal<String>();
    
    //给当前线程绑定标识为`type`的数据源
    public static void setDataSource(String type) {
        contextHolder.set(type);
    }
    
    //返回当前线程绑定的数据源标识
    public static String getDataSource() {
        if(null == contextHolder.get()){
            setDataSource(DataSourceType.DS_A.getType());
        }
        return contextHolder.get();
    }
    public static void clearDataSource() {
        contextHolder.remove();
    }
}
```
自定义数据源注解
```
@Documented
@Inherited
@Target({ElementType.TYPE,ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
public @interface TargetDataSource {
    DataSourceType value() default DataSourceType.DS_A;
}
```
aop切面自动切换数据源
```
@Aspect
@Component
public class ChangeDataSourceAop {

    @Before("execution(* com.example.demo.dao..*.*(..))")
    public void changeDataSource(JoinPoint joinPoint){
        MethodSignature signature = (MethodSignature) joinPoint.getSignature();
        Method method = signature.getMethod();
        Class<?> declaringClass = method.getDeclaringClass();

        //处理类上的注解
        if (declaringClass.isAnnotationPresent(TargetDataSource.class)) {
            TargetDataSource annotation = declaringClass.getAnnotation(TargetDataSource.class);
            DynamicDataSourceHolder.setDataSource(annotation.value().getType());
        }

        //处理方法上的注解(会覆盖类上的注解)
        if (method.isAnnotationPresent(TargetDataSource.class)) {
            TargetDataSource annotation = method.getAnnotation(TargetDataSource.class);
            DynamicDataSourceHolder.setDataSource(annotation.value().getType());
        }
    }

    @After("execution(* com.example.demo.dao..*.*(..))")
    public void clear(){
        DynamicDataSourceHolder.clearDataSource();
    }
}
```
测试
```
public void insert(){
    
    dataSourceAMapper.insert(A);//这个mapper的声明上有@TargetDataSource(DataSourceType.DS_A)注解
    dataSourceBMapper.insert(B);//这个mapper的声明上有@TargetDataSource(DataSourceType.DS_B)注解
}
```
　　ok,测试通过，能正常切换数据源，也能正常插入数据,但这种方案如果直接加上spring的事务管理器管理事务，是不起作用的，甚至会报错，无法切换数据源，下面我们来慢慢分析一下。

### 传统事务
传统的事务管理是怎样的呢？
```
Connection conn = getConnection();
try{
    Statement stm1 = conn.parpareStatement(sql_a);
    stm1.executeUpdate();
    Statement stm2 = conn.parpareStatement(sql_b);
    stm2.executeUpdate();
    conn.commit();
}catch(Exception e){
    conn.rollback();
}   
```
　　对于同一个连接，无论执行多少条sql语句，任何一条语句出现了错误的时候，整个操作流程都可以回滚，从而达到事务的原子操作。

### spring是如何做的



### 多数据源下为何失效



### 怎么办(不是太完美版)