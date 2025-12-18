这是一个非常落地的实操问题。既然你以前写得少，那我们就把**“动作拆解”**到最细，让你打开 IDEA 就能照着做。

我们先解决**“怎么喂给 AI”**的操作问题，再解决**“代码怎么写”**的技术问题，最后理清**“测试分类”**的概念。

---

### 第一部分：IDEA 里怎么把代码“喂”给 DeepSeek？

假设你用的是 **DeepSeek 网页版** 或者 **IDEA 里的 DeepSeek 插件**（比如 Continue / 通义灵码），操作略有不同。

#### 场景 1：Domain 层（纯 Java 逻辑）
*   **你的问题**：怎么把 Entity 和 Service 贴给它？
*   **操作步骤**：
    1.  **多选文件**：在 IDEA 左侧项目视图（Project）里，按住 `Ctrl` (Win) / `Command` (Mac)，同时选中 `Order.java` (Entity) 和 `OrderDomainService.java`。
    2.  **复制**：`Ctrl+C`。
    3.  **投喂**：
        *   **如果是网页版**：直接 `Ctrl+V` 粘贴到对话框。
        *   **如果是插件**：通常选中代码后，右键有 "Add to Context" 或者直接在侧边栏输入框粘贴。
    4.  **Prompt**：*“这是我的领域实体和领域服务。请为 `OrderDomainService` 生成 JUnit 5 单元测试。要覆盖所有业务分支，不需要 Spring 环境。”*

#### 场景 2：Infrastructure 层（MyBatis + 表结构）
*   **你的问题**：AI 能扫描表结构吗？
*   **回答**：AI **不能**直接连你的数据库去扫描。你需要把**“表结构的定义”**喂给它。
*   **操作步骤**：
    1.  **获取表结构**：找到你的 `schema.sql`（建表语句）或者对应的 Java Entity（`OrderPO.java`）。
    2.  **获取 Mapper**：打开 `OrderMapper.java` 接口文件。
    3.  **投喂**：把建表语句（或 PO 对象）+ Mapper 接口一起复制给 AI。
    4.  **Prompt**：*“这是表结构和 MyBatis Mapper。请帮我写一个 `@MybatisTest` 测试。使用 H2 内存数据库，帮我生成初始化数据的 `data.sql` 内容，并验证 `selectByUserId` 方法。”*

---

### 第二部分：各层测试的“具体写法”与“加载机制”

这里是核心。**记住：越往底层（Domain），加载的东西越少；越往外层（Interface），加载的东西越多。**

#### 1. Domain 层：纯单元测试 (Unit Test)
*   **加载什么**：**只加载**你要测的那个类（比如 `new OrderDomainService()`）。**完全不启动 Spring**。
*   **速度**：极快（毫秒级）。
*   **代码样子**：
    ```java
    // 没有 @SpringBootTest，没有 @Runwith
    class OrderDomainServiceTest {
        @Test
        void testCreateOrder() {
            // 手动 new，或者用 Mockito 模拟依赖
            Order order = new Order(); 
            order.calculatePrice(); // 测纯逻辑
            Assertions.assertThat(order.getPrice()).isEqualTo(100);
        }
    }
    ```

#### 2. Application 层：Mock 单元测试
*   **你的问题**：切到 Service 输入 Prompt 就可以？
*   **回答**：是的。但关键是**Application Service 依赖了 Infrastructure 层的 Repository**，你不能真去连库，所以要 **Mock**。
*   **加载什么**：只加载 Service 类本身。依赖的 Repository 用“假人”（Mock）代替。
*   **代码样子**：
    ```java
    @ExtendWith(MockitoExtension.class) // 1. 启用 Mockito，不启动 Spring
    class OrderAppServiceTest {
        
        @Mock // 2. 创建一个假的 Repository
        OrderRepository orderRepo; 
        
        @InjectMocks // 3. 把假的 Repo 注入到 Service 里
        OrderAppService orderService;

        @Test
        void testPlaceOrder() {
            // 4. 教假人做事：当调用 save 时，返回 true
            Mockito.when(orderRepo.save(any())).thenReturn(true);
            
            // 5. 调用方法
            orderService.placeOrder(new Cmd());
            
            // 6. 验证假人有没有被调用过
            Mockito.verify(orderRepo).save(any());
        }
    }
    ```

#### 3. Infrastructure 层：切片测试 (@MybatisTest)
*   **你的问题**：`@MybatisTest` 什么用？
*   **回答**：它是 Spring Boot 提供的一个**“切片”**注解。
*   **加载什么**：它**只加载** MyBatis 相关的 Bean（DataSource, SqlSessionFactory, Mapper）。**它不会加载** Controller、Service、MQ 等无关的 Bean。
*   **关于 H2**：默认情况下，`@MybatisTest` 会自动查找 H2 依赖并启动一个内存数据库。
*   **代码样子**：
    ```java
    @MybatisTest // 1. 只启动 MyBatis 环境
    // @AutoConfigureTestDatabase(replace = Replace.NONE) // 如果想连真实库/Docker，加这行
    class OrderMapperTest {
        
        @Autowired
        OrderMapper mapper; // 真 Mapper，不是 Mock

        @Test
        void testInsert() {
            mapper.insert(new OrderPO("order-1"));
            var result = mapper.selectById("order-1");
            Assertions.assertThat(result).isNotNull();
        }
    }
    ```

#### 4. Interface 层：Web 切片测试 (@WebMvcTest)
*   **你的问题**：是启动整个容器还是指定类？
*   **回答**：**只加载 Web 层**。
*   **加载什么**：它只加载 `Controller`, `ControllerAdvice`, `JsonConverter`, `Filter`。**它不会加载 Service 和 Repository**。
*   **关键点**：因为没加载 Service，你的 Controller 里注入的 Service 会报错，所以必须用 `@MockBean` 给它塞一个假的。
*   **代码样子**：
    ```java
    @WebMvcTest(OrderController.class) // 1. 只加载 OrderController
    class OrderControllerTest {
        
        @Autowired
        MockMvc mvc; // 模拟发 HTTP 请求的工具

        @MockBean // 2. 必须 Mock 掉 Service，因为 Spring 没加载它
        OrderAppService appService;

        @Test
        void testApi() throws Exception {
            // 模拟 Service 行为
            Mockito.when(appService.create(any())).thenReturn("order-id-1");

            // 发送 HTTP 请求
            mvc.perform(post("/orders").content("...json..."))
               .andExpect(status().isOk()) // 验证 200
               .andExpect(jsonPath("$.id").value("order-id-1")); // 验证返回 JSON
        }
    }
    ```

---

### 第三部分：测试分类大白话（Unit vs Slice vs Integration）

这三个的区别主要在于**“范围”**和**“真假”**。

#### 1. Unit Test (单元测试) —— 极速
*   **范围**：**一个类**。
*   **真假**：除了被测的类是真的，其他依赖（数据库、网络、别的类）**全是假的 (Mock)**。
*   **场景**：Domain 层逻辑、Application 层流程编排。
*   **特点**：不启动 Spring，跑一次 100毫秒。

#### 2. Slice Test (切片测试) —— 中速
*   **范围**：**一层**（Web 层 或 Dao 层）。
*   **真假**：
    *   测 Web 层时：Controller 是真的，Service 是假的。
    *   测 Dao 层时：Mapper 和 数据库（H2）是真的，其他是假的。
*   **场景**：`@WebMvcTest` (Interface层), `@MybatisTest` (Infra层)。
*   **特点**：启动部分 Spring 上下文，跑一次 2-5 秒。

#### 3. Integration Test (集成测试) —— 慢
*   **范围**：**整个应用**。
*   **真假**：**全是真的**。从 Controller 到 Service 到 Dao 到 数据库，全链路打通。
*   **注解**：`@SpringBootTest`。
*   **场景**：关键业务链路（比如“下单”），确保各个组件拼在一起能转。
*   **特点**：启动完整 Spring 容器，跑一次 10-20 秒。

#### 4. 其他分类（了解即可）
*   **E2E Test (端到端测试)**：模拟用户操作浏览器（Selenium/Playwright）。这是测试人员干的事。
*   **Contract Test (契约测试)**：微服务之间，A 服务改了接口，B 服务能不能知道？（用 Pact 工具）。

### 总结你的行动路径

1.  **Domain/Application 层**：直接把代码贴给 AI，让它生成 **Mockito** 风格的测试。**不要加 Spring 注解**。
2.  **Infrastructure 层**：把 Mapper 和 SQL 贴给 AI，让它生成 **@MybatisTest**。默认用 H2 跑，最省事。
3.  **Interface 层**：把 Controller 贴给 AI，让它生成 **@WebMvcTest**，并告诉它要 Mock 哪个 Service。

这样你就不用纠结配置，直接享受 AI 带来的效率提升。