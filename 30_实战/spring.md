
1. **利用@RequiredArgsConstructor和final修饰属性来注入其他bean**

```
@Service
@RequiredArgsConstructor
public class OrderService {

    // 情况 1：static final 常量
    // 【结果】忽略。类加载时初始化，不进构造器。
    private static final String DEFAULT_STATUS = "PENDING";

    // 情况 2：已初始化的 final 实例变量
    // 【结果】忽略。因为已经有初始值了，不需要构造器赋值。
    private final long createTimestamp = System.currentTimeMillis();

    // 情况 3：未初始化的 final 实例变量（Bean 依赖）
    // 【结果】加入构造器。这是唯一的“Required”字段。
    private final OrderRepository orderRepository;

}
```
2. **@ConfigurationProperties(prefix = "aliyun.oss")来把value的字段抽取出来
```
   @ConfigurationProperties(prefix = "aliyun.oss")
   @RequiredArgsConstructor
   // 在 Spring Boot 3.0+ 中，这通常配合.EnableConfigurationProperties 使用
public class OssConfig {

    private final String bucketName;
    private final String endpoint;
    private final int maxConnections;
    
    // Spring 会自动识别这是一个“构造器绑定（Constructor Binding）”的配置类
}


@Service
@RequiredArgsConstructor // 只负责注入 Properties 这个 Bean
public class OssService {

    // 注入的是整个配置对象，它是 final 的
    private final AliyunOssProperties ossProperties;
    private final OssClient ossClient;

    public void upload(byte[] data) {
        // 使用时通过对象获取，代码极其整洁
        String bucket = ossProperties.bucketName();
        // ...
    }
}
   ```

