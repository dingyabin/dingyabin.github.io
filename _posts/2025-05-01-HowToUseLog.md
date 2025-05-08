---
layout: post
title: 日志使用技巧
date: 2025-05-01
tags: 日志
---
日志是软件开发中不可或缺的一部分，它能帮助我们了解应用运行状态、调试问题和监控性能。

在项目中，使用正确的使用和记录日志不仅能提高代码可维护性，还能在生产环境中更快地排查问题。

#  1\. 使用SLF4J门面模式统一日志API

SLF4J (Simple Logging Facade for Java) 提供了统一的日志API接口，让你可以轻松切换底层日志实现。

    
    
    import org.slf4j.Logger;  
    import org.slf4j.LoggerFactory;  
      
    public class UserService {  
        // 获取Logger实例  
        private static final Logger logger = LoggerFactory.getLogger(UserService.class);  
          
        public void createUser(User user) {  
            logger.info("Creating user: {}", user.getUsername());  
            // 业务逻辑  
        }  
    }

** 最佳实践  ** ：始终使用SLF4J作为日志门面，避免直接依赖具体实现如Log4j或Logback，这样可以在不修改代码的情况下切换底层日志框架。

#  2\. 使用参数化日志替代字符串拼接

字符串拼接在日志中是常见的性能陷阱，正确的做法是使用参数化日志。

    
    
    // 错误示例 - 即使日志级别不满足也会执行字符串拼接  
    logger.debug("Processing order: " + order.getId() + " with amount: " + order.getAmount());  
      
    // 正确示例 - 只有在日志级别满足时才会执行参数替换  
    logger.debug("Processing order: {} with amount: {}", order.getId(), order.getAmount());

** 性能提升  ** ：参数化日志避免了不必要的字符串拼接操作，特别是当日志级别高于DEBUG时，可以节省大量CPU资源。

#  3\. 使用条件日志避免高成本计算

对于需要复杂计算的日志信息，应该先检查日志级别。

    
    
    // 检查日志级别再执行耗时操作  
    if (logger.isDebugEnabled()) {  
        logger.debug("Complex calculation result: {}", calculateExpensiveValue());  
    }

** 应用场景  ** ：当日志内容需要复杂计算或资源密集型操作时，这一方法能显著提高性能。

#  4\. 合理使用不同日志级别

选择正确的日志级别对于控制日志输出量和重要性至关重要。

    
    
    // 跟踪详细信息  
    logger.trace("Entering method with parameters: {}", params);  
      
    // 调试信息  
    logger.debug("Processing item at index: {}", index);  
      
    // 正常业务流程信息  
    logger.info("User {} successfully logged in", username);  
      
    // 警告信息  
    logger.warn("Database connection pool is running low: {} connections left", availableConnections);  
      
    // 错误信息  
    logger.error("Failed to process transaction", exception);

** 最佳实践  ** ：

  * • TRACE：仅用于非常详细的诊断信息 
  * • DEBUG：用于开发和调试信息 
  * • INFO：用于记录正常业务流程 
  * • WARN：潜在问题但不影响正常运行 
  * • ERROR：错误导致功能无法正常工作 

#  5\. MDC (Mapped Diagnostic Context) 上下文信息添加

MDC是一个非常强大的工具，可以在整个调用链上传递上下文信息。

    
    
    import org.slf4j.MDC;  
      
    // 在请求处理开始添加上下文  
    MDC.put("userId", user.getId());  
    MDC.put("requestId", UUID.randomUUID().toString());  
      
    try {  
        // 业务逻辑处理  
        logger.info("Processing user request");  
        // 所有日志都会自动包含MDC中的上下文信息  
    } finally {  
        // 请求结束后清理上下文  
        MDC.clear();  
    }

** 配置Logback输出MDC信息  ** ：

    
    
    <pattern>%d{yyyy-MM-dd HH:mm:ss} [%thread] %-5level %logger{36} [userId:%X{userId}, requestId:%X{requestId}] - %msg%n</pattern>

** 应用场景  ** ：微服务架构中的请求跟踪、用户操作审计、多租户系统中的租户识别。

#  6\. 异常日志记录最佳实践

记录异常时，既要提供上下文信息，也要包含完整的堆栈信息。

    
    
    try {  
        // 业务逻辑  
    } catch (DatabaseException e) {  
        // 提供上下文和完整堆栈  
        logger.error("Failed to save user data for userId: {}", userId, e);  
          
        // 不要这样做 - 丢失堆栈信息  
        // logger.error("Failed to save user data: " + e.getMessage());  
    }

** 最佳实践  ** ：始终将异常对象作为日志方法的最后一个参数，这样可以捕获完整堆栈信息。

#  7\. 使用日志标记分类信息

在复杂系统中，可以使用标记来分类不同类型的日志信息。

    
    
    // 使用Logback的Marker功能  
    import org.slf4j.Marker;  
    import org.slf4j.MarkerFactory;  
      
    public class SecurityService {  
        private static final Logger logger = LoggerFactory.getLogger(SecurityService.class);  
        private static final Marker SECURITY_MARKER = MarkerFactory.getMarker("SECURITY");  
          
        public void login(String username, boolean success) {  
            logger.info(SECURITY_MARKER, "Login attempt: user={}, success={}", username, success);  
        }  
    }

** 过滤特定标记的日志  ** ：

    
    
    <filter class="ch.qos.logback.core.filter.EvaluatorFilter">  
        <evaluator>  
            <expression>marker.contains("SECURITY")</expression>  
        </evaluator>  
        <OnMatch>ACCEPT</OnMatch>  
        <OnMismatch>DENY</OnMismatch>  
    </filter>

#  8\. 结构化日志输出（JSON格式）

在微服务环境中，结构化日志便于集中式日志分析工具处理。

** 添加Logstash编码器依赖  ** ：

    
    
    <dependency>  
        <groupId>net.logstash.logback</groupId>  
        <artifactId>logstash-logback-encoder</artifactId>  
        <version>7.2</version>  
    </dependency>

** Logback配置  ** ：

    
    
    <appender name="JSON" class="ch.qos.logback.core.ConsoleAppender">  
        <encoder class="net.logstash.logback.encoder.LogstashEncoder">  
            <includeMdcKeyName>userId</includeMdcKeyName>  
            <includeMdcKeyName>requestId</includeMdcKeyName>  
        </encoder>  
    </appender>

** 使用效果  ** ：所有日志将输出为JSON格式，便于ELK或类似系统解析。

#  9\. 实现自定义日志格式

针对特定需求，可以创建自定义的日志格式。

    
    
    // 创建自定义日志消息格式化器  
    public class CustomLogMessageFormatter {  
        public static String formatTransaction(String txId, String status, long duration) {  
            return String.format("TX[%s] completed with status %s in %dms", txId, status, duration);  
        }  
    }  
      
    // 在代码中使用  
    logger.info(CustomLogMessageFormatter.formatTransaction("TX12345", "SUCCESS", 134));

** 最佳实践  ** ：对于频繁使用的复杂日志格式，封装成专用方法可以提高代码可读性和一致性。

#  10\. 使用异步日志提升性能

日志I/O操作可能成为性能瓶颈，异步日志可以显著提升应用性能。

性能要求极高的场景推荐使用log4j2异步模式，性能远高于logback的异步模式。

** Logback异步配置  ** ：

    
    
    <appender name="ASYNC" class="ch.qos.logback.classic.AsyncAppender">  
        <appender-ref ref="FILE" />  
        <queueSize>512</queueSize>  
        <discardingThreshold>0</discardingThreshold>  
        <includeCallerData>false</includeCallerData>  
    </appender>  
    <root level="INFO">  
        <appender-ref ref="ASYNC" />  
    </root>

** 性能提升  ** ：异步日志可以减少主线程的阻塞，在高并发系统中尤其有效。

#  11\. 日志输出敏感信息处理

处理用户数据时，需要特别注意敏感信息的日志输出。

    
    
    // 创建敏感数据掩码工具  
    public class LogMaskUtil {  
        public static String maskCardNumber(String cardNumber) {  
            if (cardNumber == null || cardNumber.length() < 8) {  
                return "****";  
            }  
            return "****" + cardNumber.substring(cardNumber.length() - 4);  
        }  
          
        public static String maskEmail(String email) {  
            if (email == null || email.isEmpty() || !email.contains("@")) {  
                return "****@****.com";  
            }  
            String[] parts = email.split("@");  
            return parts[0].substring(0, 1) + "***@" + parts[1];  
        }  
    }  
      
    // 在日志中使用  
    logger.info("Processing payment for card: {}", LogMaskUtil.maskCardNumber(card.getNumber()));  
    logger.info("Sending confirmation to: {}", LogMaskUtil.maskEmail(user.getEmail()));

** 安全提示  ** ：永远不要在日志中记录密码、完整信用卡号、社会安全号码等敏感信息，即使在开发环境中也应如此。

#  12\. 特定业务领域的日志上下文

为不同业务领域创建专用的日志上下文，便于追踪和分析。

    
    
    // 创建业务上下文日志工具  
    public class OrderLogContext {  
        private String orderId;  
        private String customerId;  
        private BigDecimal amount;  
          
        public OrderLogContext(String orderId, String customerId, BigDecimal amount) {  
            this.orderId = orderId;  
            this.customerId = customerId;  
            this.amount = amount;  
        }  
          
        public void setupContext() {  
            MDC.put("orderId", orderId);  
            MDC.put("customerId", customerId);  
            MDC.put("amount", amount.toString());  
        }  
          
        public void clearContext() {  
            MDC.remove("orderId");  
            MDC.remove("customerId");  
            MDC.remove("amount");  
        }  
          
        // 使用try-with-resources模式  
        public static class LogContextResource implements AutoCloseable {  
            private final OrderLogContext context;  
              
            public LogContextResource(OrderLogContext context) {  
                this.context = context;  
                this.context.setupContext();  
            }  
              
            @Override  
            public void close() {  
                this.context.clearContext();  
            }  
        }  
    }  
      
    // 在代码中使用  
    try (OrderLogContext.LogContextResource ignored =   
            new OrderLogContext.LogContextResource(new OrderLogContext(order.getId(),   
                                                                     order.getCustomerId(),   
                                                                     order.getAmount()))) {  
        logger.info("Processing order");  
        orderService.process(order);  
        logger.info("Order completed successfully");  
    }

#  13\. 日志性能监控与计时

使用日志记录操作执行时间，帮助识别性能瓶颈。

    
    
    // 简易性能日志  
    public class PerformanceLogger {  
        private static final Logger logger = LoggerFactory.getLogger(PerformanceLogger.class);  
          
        public static <T> T logExecutionTime(String operationName, Supplier<T> operation) {  
            long startTime = System.currentTimeMillis();  
            try {  
                return operation.get();  
            } finally {  
                long duration = System.currentTimeMillis() - startTime;  
                logger.info("Operation [{}] completed in {}ms", operationName, duration);  
            }  
        }  
          
        // 无返回值版本  
        public static void logExecutionTime(String operationName, Runnable operation) {  
            long startTime = System.currentTimeMillis();  
            try {  
                operation.run();  
            } finally {  
                long duration = System.currentTimeMillis() - startTime;  
                logger.info("Operation [{}] completed in {}ms", operationName, duration);  
            }  
        }  
    }  
      
    // 使用示例  
    User user = PerformanceLogger.logExecutionTime("fetchUserProfile",   
        () -> userService.getUserById(userId));

#  14\. 条件日志收集器

对于需要收集多条日志然后一次性输出的场景，可以实现日志收集器。

    
    
    // 日志收集器  
    public class LogCollector {  
        private final List<String> messages = new ArrayList<>();  
        private final Logger logger;  
        private final Level level;  
          
        public LogCollector(Logger logger, Level level) {  
            this.logger = logger;  
            this.level = level;  
        }  
          
        public void add(String message) {  
            messages.add(message);  
        }  
          
        public void add(String format, Object... args) {  
            messages.add(String.format(format, args));  
        }  
          
        public void flush(String summary) {  
            if (messages.isEmpty()) {  
                return;  
            }  
              
            StringBuilder sb = new StringBuilder(summary);  
            sb.append(":\n");  
            for (int i = 0; i < messages.size(); i++) {  
                sb.append("  ").append(i + 1).append(". ")  
                  .append(messages.get(i)).append("\n");  
            }  
              
            switch (level.toString()) {  
                case "DEBUG":  
                    logger.debug(sb.toString());  
                    break;  
                case "INFO":  
                    logger.info(sb.toString());  
                    break;  
                case "WARN":  
                    logger.warn(sb.toString());  
                    break;  
                case "ERROR":  
                    logger.error(sb.toString());  
                    break;  
            }  
              
            messages.clear();  
        }  
    }  
      
    // 使用示例  
    LogCollector collector = new LogCollector(logger, Level.INFO);  
    for (Item item : items) {  
        try {  
            processItem(item);  
            collector.add("Item %s processed successfully", item.getId());  
        } catch (Exception e) {  
            collector.add("Failed to process item %s: %s", item.getId(), e.getMessage());  
        }  
    }  
    collector.flush("Batch processing results");

#  总结

良好的日志实践不仅能帮助开发者更快地调试问题，还能为生产环境监控和故障排除提供宝贵的信息。

好的日志应该像讲故事一样，清晰地描述应用的运行状态和流程，帮助我们快速理解系统行为。











****



****





__









