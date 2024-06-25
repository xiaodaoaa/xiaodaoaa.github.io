ExpiringMap 是一个 Java 库，用于创建自动过期的键值对集合。它支持基于时间的条目过期，并且提供了一些有用的特性，比如过期回调和自定义过期策略。以下是一个详细的 ExpiringMap 示例，展示了它的基本用法及其特性。

### 1. 引入 ExpiringMap 库

首先，你需要在项目中引入 ExpiringMap 库。可以通过 Maven 或 Gradle 进行依赖管理。

#### Maven
```xml
<dependency>
    <groupId>net.jodah</groupId>
    <artifactId>expiringmap</artifactId>
    <version>0.5.9</version>
</dependency>
```

#### Gradle
```groovy
implementation 'net.jodah:expiringmap:0.5.9'
```

### 2. 创建一个 ExpiringMap

下面是一个简单的示例，展示了如何创建和使用 ExpiringMap：

```java
import net.jodah.expiringmap.ExpiringMap;
import java.util.concurrent.TimeUnit;

public class ExpiringMapExample {
    public static void main(String[] args) throws InterruptedException {
        // 创建一个 ExpiringMap 实例，设置条目在 5 秒后过期
        ExpiringMap<String, String> map = ExpiringMap.builder()
                                                     .expiration(5, TimeUnit.SECONDS)
                                                     .build();

        // 添加一些条目
        map.put("key1", "value1");
        map.put("key2", "value2");

        // 打印当前 map 内容
        System.out.println("Initial map: " + map);

        // 等待 6 秒，超过条目过期时间
        Thread.sleep(6000);

        // 打印 map 内容，检查条目是否过期
        System.out.println("Map after 6 seconds: " + map);
    }
}
```

### 3. 设置不同的过期时间

你可以为每个条目设置不同的过期时间：

```java
import net.jodah.expiringmap.ExpiringMap;
import java.util.concurrent.TimeUnit;

public class ExpiringMapExample {
    public static void main(String[] args) throws InterruptedException {
        ExpiringMap<String, String> map = ExpiringMap.builder()
                                                     .expiration(10, TimeUnit.SECONDS)
                                                     .build();

        // 为每个条目设置不同的过期时间
        map.put("key1", "value1", 5, TimeUnit.SECONDS);
        map.put("key2", "value2", 10, TimeUnit.SECONDS);

        // 打印当前 map 内容
        System.out.println("Initial map: " + map);

        // 等待 6 秒
        Thread.sleep(6000);

        // 打印 map 内容，检查条目是否过期
        System.out.println("Map after 6 seconds: " + map);

        // 等待再 5 秒
        Thread.sleep(5000);

        // 打印 map 内容，检查条目是否过期
        System.out.println("Map after 11 seconds: " + map);
    }
}
```

### 4. 过期回调

ExpiringMap 允许你在条目过期时执行回调操作：

```java
import net.jodah.expiringmap.ExpiringMap;
import net.jodah.expiringmap.ExpirationListener;
import java.util.concurrent.TimeUnit;

public class ExpiringMapExample {
    public static void main(String[] args) throws InterruptedException {
        // 创建一个带有过期监听器的 ExpiringMap
        ExpiringMap<String, String> map = ExpiringMap.builder()
                                                     .expiration(5, TimeUnit.SECONDS)
                                                     .expirationListener(new ExpirationListener<String, String>() {
                                                         @Override
                                                         public void expired(String key, String value) {
                                                             System.out.println("Expired: " + key + " => " + value);
                                                         }
                                                     })
                                                     .build();

        // 添加一些条目
        map.put("key1", "value1");
        map.put("key2", "value2");

        // 打印当前 map 内容
        System.out.println("Initial map: " + map);

        // 等待 6 秒，超过条目过期时间
        Thread.sleep(6000);

        // 打印 map 内容，检查条目是否过期
        System.out.println("Map after 6 seconds: " + map);
    }
}
```

### 5. 自定义过期策略

你可以使用不同的过期策略，例如访问后过期或创建后过期：

```java
import net.jodah.expiringmap.ExpiringMap;
import net.jodah.expiringmap.ExpirationPolicy;
import java.util.concurrent.TimeUnit;

public class ExpiringMapExample {
    public static void main(String[] args) throws InterruptedException {
        // 创建一个 ExpiringMap 实例，设置条目在 5 秒后访问过期
        ExpiringMap<String, String> map = ExpiringMap.builder()
                                                     .expiration(5, TimeUnit.SECONDS)
                                                     .expirationPolicy(ExpirationPolicy.ACCESSED)
                                                     .build();

        // 添加一些条目
        map.put("key1", "value1");
        map.put("key2", "value2");

        // 打印当前 map 内容
        System.out.println("Initial map: " + map);

        // 访问一个条目，延长其过期时间
        map.get("key1");

        // 等待 4 秒
        Thread.sleep(4000);

        // 打印 map 内容，检查条目是否过期
        System.out.println("Map after 4 seconds: " + map);

        // 再等待 2 秒
        Thread.sleep(2000);

        // 打印 map 内容，检查条目是否过期
        System.out.println("Map after 6 seconds: " + map);
    }
}
```

`ExpirationPolicy.ACCESSED` 是 ExpiringMap 提供的一种过期策略。它的含义是：条目的过期时间会在每次访问（读取）时重新计算。这意味着，如果你访问了某个条目，它的过期时间会被延长，相当于“刷新”了它的存在时间。

具体来说，当一个条目被放入 `ExpiringMap` 时，初始的过期时间会被设置。如果在过期时间到达之前，该条目被访问（通过 `get` 方法读取），那么它的过期时间会被重新计算，从访问的时刻开始重新计时。这种策略在某些场景下非常有用，例如实现带有自动续期功能的缓存。

### 结论

ExpiringMap 是一个强大的工具，可以轻松地管理带有自动过期功能的键值对集合。它提供了灵活的配置选项，可以根据需要设置不同的过期策略和回调函数，适用于各种场景。