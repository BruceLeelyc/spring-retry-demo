# spring-retry-demo
# 添加依赖
# <dependency>
#     <groupId>org.springframework.retry</groupId>
#     <artifactId>spring-retry</artifactId>
#     <version>1.3.3</version>
# </dependency>
#  <dependency>
#      <groupId>org.springframework.boot</groupId>
#      <aifactId>spring-boot-starter-aop</artifactId>
# </dependency>
# 
# @Configuration
# @EnableRetry
# public class RetryConfig {
# }
# 
# @Retryable
# 在需要重试的方法上加上@Retryable注解部分参数如下：
#     label: 名字系统唯一默认:“”
#     maxAttempts:异常时重试次数,默认=3
#     maxAttemptsExpression: SpEL表达式,从配置文件获取maxAttempts的值,可以在application.yml设置,与maxAttempts二选一
#     exceptionExpression: SpEL表达式,匹配异常.例如：exceptionExpression = "#{message.contains('test')}"
#     backoff:重试中的退避策略 ,@Backoff注解，部分参数如下：
#         value:重试间隔ms,默认=1000
#         delay: 在指数情况下用作初始值，在均匀情况下用作最小值, 它与value属性不能共存，当delay不设置的时候会去读value属性设置的值，如果delay设置的话则会忽略value属性, 默认 0
#         delayExpression: SpEL表达式 ，从配置文件获取delay的值，可以在application.yml设置，与delay二选一
#         multiplier: 则用作产生下一个退避延迟的乘数, 默认=0 delay = 2000, multiplier = 2 表示第一次重试间隔为2s，第二次为4秒，第三次为8s
#         maxDelay:最大的重试间隔,当超过这个最大的重试间隔的时候,重试的间隔就等于maxDelay的值默认0
# 
# @Service
# @Slf4j
# public class RetryService {
# 
#     @Retryable(value = RuntimeException.class)
#     public void test(String param){
#         log.info(param);
#         throw new RuntimeException("laker Error");
#     }
# }
# 
# 当抛出RuntimeException时会尝试重试。
# 根据@Retryable的默认行为，重试最多可能发生 3 次，重试之间有 1 秒的延迟。
# @Recover
# 当@Retryable方法重试失败之后，最后就会调用@Recover方法。用于@Retryable失败时的兜底处理方法。
# @Recover的方法必须要与@Retryable注解的方法保持一致，第一入参为要重试的异常，其他参数与@Retryable保持一致，返回值也要一样，否则无法执行！,方法可以是public、private.
# @Service
# @Slf4j
# public class RetryService {
#     @Retryable(value = RuntimeException.class)
#     public void test(String param) {
#         log.info(param);
#         throw new RuntimeException("laker Error");
#     }
#     @Recover
#     void recover(RuntimeException e, String param) {
#         log.info("recover e:{},param:{}", e, param);
#     }
# }
# 
# 在这里，当抛出RuntimeException时会尝试重试
# test方法在 3 次尝试后不断抛出 RuntimeException，则会调用recover()方法
# @CircuitBreaker
# 熔断模式：指在具体的重试机制下失败后打开断路器，过了一段时间，断路器进入半开状态，允许一个进入重试，若失败再次进入断路器，成功则关闭断路器，注解为@CircuitBreaker,具体包括熔断打开时间、重置过期时间。
#     同一个方法上与@Retryable注解只能二选一，否则注解失效
#     相关代码参见CircuitBreakerRetryPolicy.java
# 主要参数如下：
#     maxAttempts： 最大尝试次数（包括第一次失败），默认为 3
#     maxAttemptsExpression: SpEL表达式 ，从配置文件获取maxAttempts的值，可以在application.yml设置，与maxAttempts二选一
#     openTimeout：当在此超时时间内达到maxAttempts失败时，电路会自动打开，防止访问下游组件。默认为 5000
#     openTimeoutExpression: SpEL表达式
#     resetTimeout： 如果电路打开的时间超过此超时时间，则它会在下一次调用时重置，以使下游组件有机会再次响应。默认为 20000
#     resetTimeoutExpression: SpEL表达式
#     label：短路器的名字，系统唯一
#     include：需要短路的异常
#     exclude：不需要短路的异常
#     @CircuitBreaker(maxAttempts = 2, openTimeout = 1000, resetTimeout = 2000, value = RuntimeException.class)
#     public void testCircuitBreaker(String param) {
#         log.info(param);
#         throw new RuntimeException("laker Error");
#     }
#     @Recover
#     void recover(RuntimeException e, String param) {
#         log.info("recover e:{},param:{}", e, param);
# }
# 
# 当抛出RuntimeException时会尝试熔断。
# 在openTimeout 1s时间内，触发异常超过2次，断路器打开，testCircuitBreaker业务方法不允许执行，直接执行恢复方法recover。
# 经过resetTimeout 2s后，熔断器关闭，继续执行testCircuitBreaker业务方法。
# 注意：这里没有上面@Retryable的能力了哦，但是这个实际项目还是很需要的。
# 高级实战
# 上面说到了，断路器@CircuitBreaker 并么有携带重试功能，所有我们实际项目要结合2者使用。
# 方式一 @CircuitBreaker + RetryTemplate
# 1.自定义RetryTemplate
# @Configuration
# @EnableRetry
# public class RetryConfig {
#     @Bean
#     public RetryTemplate retryTemplate() {
#         RetryTemplate retryTemplate = new RetryTemplate();
#         FixedBackOffPolicy fixedBackOffPolicy = new FixedBackOffPolicy();
#         // 退避策略 因为是瞬时异常 所以不宜过大，100ms即可
#         fixedBackOffPolicy.setBackOffPeriod(100L);
#         retryTemplate.setBackOffPolicy(fixedBackOffPolicy);
#         SimpleRetryPolicy retryPolicy = new SimpleRetryPolicy();
#         // 重试3次
#         retryPolicy.setMaxAttempts(3);
#         retryTemplate.setRetryPolicy(retryPolicy);
#         return retryTemplate;
#     }
# }
# 
# 2.在断路器中用retryTemplate包裹一层
# @CircuitBreaker(maxAttempts = 2, openTimeout = 1000, resetTimeout = 2000, value = RuntimeException.class)
#     public String testCircuitBreaker(String param) {
#         return retryTemplate.execute(context -> {
#             log.info(String.format("Retry count %d", context.getRetryCount()) + param);
#             throw new RuntimeException("laker Error");
#         });
#     }
# 
#     @Recover
#     String recover(RuntimeException e, String param) {
#         log.info("recover e:{},param:{}", e, param);
#         return "";
#     }
# 方式二 @CircuitBreaker + @Retryable
# 定义2个springBean，一个用于重试，一个用于熔断，且是熔断包含着重试，否则会失效。
# @Service
# @Slf4j
# public class RetryService {
#     @Autowired
#     RetryTemplate retryTemplate;
#     @Retryable(value = RuntimeException.class,backoff = @Backoff(delay = 100))
#     public void test(String param) {
#         log.info(param);
#         throw new RuntimeException("laker Error");
#     }
# }
# 
# @Service
# @Slf4j
# public class CircuitBreakerService {
#     @Autowired
#     RetryService retryService;
#     @CircuitBreaker(maxAttempts = 2, openTimeout = 1000, resetTimeout = 2000, value = RuntimeException.class)
#     public void testCircuitBreaker(String param) {
#         // 这里是添加了重试注解的方法
#         retryService.test(param);
#     }
#     @Recover
#     void recover(RuntimeException e, String param) {
#         log.info("recover e:{},param:{}", e, param);
#     }
# }
