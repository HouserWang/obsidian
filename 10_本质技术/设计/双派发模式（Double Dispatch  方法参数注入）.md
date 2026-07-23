# 双派发模式（Double Dispatch / 方法参数注入）

### 📌 名词定义

双派发（在 DDD 战术设计中）是指**由于领域实体（Entity）不受 Spring 容器管理，无法使用依赖注入（如 @Autowired），因此在执行某些需要外部能力的领域行为时，由调用方（AppService）将所需的只读接口、服务或 Lambda 表达式，作为方法参数动态“派发”给实体的过程。**

### 🧠 第一性原理

实体通过 new 或工厂方法创建，属于生命周期短暂的领域对象，无法直接拥有单例的 Repository。  
为了防止逻辑泄露到 AppService（造成贫血模型），我们不把数据从实体里拿出来，而是**把“能力（只读接口）”送进实体里**。  

```
对象A 拿着 行为B 去找 对象C→A.execute(C)对象A 拿着 行为B 去找 对象C→A.execute(C)
```

### 🛠️ 落地实现（极简的 Lambda 双派发）

code

```Java
// 1. 聚合根内部方法：接收一个 Java 函数式接口作为“校验能力”
public class User {
    public static User register(String name, String email, Predicate<String> emailExistsChecker) {
        // 实体自己在内存里执行逻辑，但能力是外面送来的
        if (emailExistsChecker.test(email)) { 
            throw new DomainException("邮箱已被注册");
        }
        return new User(name, email);
    }
}

// 2. 应用服务层：动态将 Repository 的方法引用作为参数派发过去
userRepository.save(
    User.register(name, email, userRepository::existsByEmail)
);
```

### ⚖️ 架构权衡

- **优点**：完美解决了“实体不能注入 Spring Bean”的技术阻碍，让业务校验规则百分之百内聚在实体内部。使用 Lambda 表达式实现时，零新增接口成本，测试极度友好。
    
- **缺点**：对于习惯了传统贫血 MVC 的开发人员来说，方法参数传递函数指针的写法需要一定的学习成本。
    

---