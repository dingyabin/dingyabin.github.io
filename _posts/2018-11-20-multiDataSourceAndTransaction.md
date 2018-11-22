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
### 分库方案描述  
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
　　我们知道spring是通过aop切面来做事务管理的，其切面类为`org.springframework.transaction.interceptor.TransactionInterceptor`,追踪其invoke方法里重要的一段代码：
```
	protected Object invokeWithinTransaction(Method method, Class<?> targetClass, final InvocationCallback invocation)throws Throwable {

		// If the transaction attribute is null, the method is non-transactional.
		final TransactionAttribute txAttr = getTransactionAttributeSource().getTransactionAttribute(method, targetClass);
		final PlatformTransactionManager tm = determineTransactionManager(txAttr);
		final String joinpointIdentification = methodIdentification(method, targetClass, txAttr);

		if (txAttr == null || !(tm instanceof CallbackPreferringPlatformTransactionManager)) {
			// Standard transaction demarcation with getTransaction and commit/rollback calls.
			TransactionInfo txInfo = createTransactionIfNecessary(tm, txAttr, joinpointIdentification);
			Object retVal = null;
			try {
				// This is an around advice: Invoke the next interceptor in the chain.
				// This will normally result in a target object being invoked.
				retVal = invocation.proceedWithInvocation();
			}
			catch (Throwable ex) {
				// target invocation exception
				completeTransactionAfterThrowing(txInfo, ex);
				throw ex;
			}
			finally {
				cleanupTransactionInfo(txInfo);
			}
			commitTransactionAfterReturning(txInfo);
			return retVal;
		}
	                            .
	                            .
	                            .
	                            .
	   
```
注意`retVal = invocation.proceedWithInvocation() `这行，这行的意思是继续执行下一个切面增强类的处理，直到最后的被代理类的被代理方法返回结果, 这行代码之前，spring
进行了一些列的初始化操作，其中createTransactionIfNecessary(tm, txAttr, joinpointIdentification)即为开启了一个事务，顺着往下走，看看createTransactionIfNecessary方法做了啥：
```
	protected TransactionInfo createTransactionIfNecessary(PlatformTransactionManager tm, TransactionAttribute txAttr, final String joinpointIdentification) {

		// If no name specified, apply method identification as transaction name.
		if (txAttr != null && txAttr.getName() == null) {
			txAttr = new DelegatingTransactionAttribute(txAttr) {
				@Override
				public String getName() {
					return joinpointIdentification;
				}
			};
		}

		TransactionStatus status = null;
		if (txAttr != null) {
			if (tm != null) {
				status = tm.getTransaction(txAttr);
			}else {
				if (logger.isDebugEnabled()) {
					logger.debug("Skipping transactional joinpoint [" + joinpointIdentification +"] because no transaction manager has been configured");
				}
			}
		}
		return prepareTransactionInfo(tm, txAttr, joinpointIdentification, status);
	}
```
这里调用了`tm.getTransaction(txAttr)`来开启事务,由于这里的`tm`我们一般使用的是`DataSourceTransactionManager`,直接进去看`DataSourceTransactionManager`的`getTransaction()`方法:
```
	public final TransactionStatus getTransaction(TransactionDefinition definition) throws TransactionException {
	
		Object transaction = doGetTransaction();

                            .......
        ///判断是否已经存在的一个事务
		if (isExistingTransaction(transaction)) {
			// Existing transaction found -> check propagation behavior to find out how to behave.
			return handleExistingTransaction(definition, transaction, debugEnabled);
		}

                            .......

						
		doBegin(transaction, definition);
				
                            .......
	}
```
源码太长，只截取了部分核心代码，进入方法，先委托doGetTransaction()拿到一个事务，然后调用isExistingTransaction()看看之前是不是已经存在一个事务了，如果之前没有事务，则doBegin(),开启事务。
先看看`doGetTransaction()`方法:
```
	protected Object doGetTransaction() {
		DataSourceTransactionObject txObject = new DataSourceTransactionObject();
		txObject.setSavepointAllowed(isNestedTransactionAllowed());
		ConnectionHolder conHolder =(ConnectionHolder) TransactionSynchronizationManager.getResource(this.dataSource);
		txObject.setConnectionHolder(conHolder, false);
		return txObject;
	}
```
这里主要是根据dataSource来获取ConnectionHolder，这个ConnectionHolder是放在TransactionSynchronizationManager的ThreadLocal中持有的，此时如果是第一次来获取，肯定得到是null，
这里请记住`TransactionSynchronizationManager`这个类，后面会再遇到的。在看看doBegin()

```
	protected void doBegin(Object transaction, TransactionDefinition definition) {
		DataSourceTransactionObject txObject = (DataSourceTransactionObject) transaction;
		Connection con = null;
		try {
			if (txObject.getConnectionHolder() == null ||txObject.getConnectionHolder().isSynchronizedWithTransaction()) {
				//从dataSource获取一个新的Connection
				Connection newCon = this.dataSource.getConnection();
				//把上面新的连接放入txObject中，并标记为true(表示是一个新的ConnectionHolder)
				txObject.setConnectionHolder(new ConnectionHolder(newCon), true);
			}

			txObject.getConnectionHolder().setSynchronizedWithTransaction(true);
			con = txObject.getConnectionHolder().getConnection();

                ...............

			// 如果是新的ConnectionHolder，绑定到线程里
			if (txObject.isNewConnectionHolder()) {
				TransactionSynchronizationManager.bindResource(getDataSource(), txObject.getConnectionHolder());
			}
		}
		catch (Throwable ex) {
			if (txObject.isNewConnectionHolder()) {
				DataSourceUtils.releaseConnection(con, this.dataSource);
				txObject.setConnectionHolder(null, false);
			}
			throw new CannotCreateTransactionException("Could not open JDBC Connection for transaction", ex);
		}
	}
```
由于第一次开启事务时，上一步获得的ConnectionHolder为null，所以从dataSource获取了一个新的Connection包装成ConnectionHolder放入txObject中，
并标记为true(一个新的ConnectionHolder); 紧接着判断是一个新的ConnectionHolder的话，用`TransactionSynchronizationManager.bindResource()`方法绑定到线程里。
还记得上面的doGetTransaction()方法里用TransactionSynchronizationManager.getResource()获取连接么,如果是同一个线程，再次进入执行的话就会获取到同一个ConnectionHolder，
在后面的isExistingTransaction方法也可以判定为是已有的transaction，于是后面的操作就都复用同一个connect了，发生异常时只要connect.rollback()一下，就会使前面的操作都回滚。

### 多数据源下为何失效  



### 怎么办(不是太完美版)