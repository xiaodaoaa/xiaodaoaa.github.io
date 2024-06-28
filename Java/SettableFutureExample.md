`SettableFuture` 是 Guava 库中的一个类，用于创建可设置其结果的 `Future`。与标准的 `Future` 不同，`SettableFuture` 允许你手动设置其结果或异常，这使得它在某些异步编程场景中非常有用。以下是 `SettableFuture` 的详细示例和解释：

首先，确保你已经添加了 Guava 依赖项。Maven 项目的 `pom.xml` 文件中需要包含以下依赖：

```xml
<dependency>
    <groupId>com.google.guava</groupId>
    <artifactId>guava</artifactId>
    <version>31.1-jre</version>
</dependency>
```

或者，如果你使用 Gradle，可以添加：

```gradle
implementation 'com.google.guava:guava:31.1-jre'
```

接下来是一个使用 `SettableFuture` 的完整示例：

```java
import com.google.common.util.concurrent.ListenableFuture;
import com.google.common.util.concurrent.MoreExecutors;
import com.google.common.util.concurrent.SettableFuture;

import java.util.concurrent.ExecutionException;

public class SettableFutureExample {

    public static void main(String[] args) {
        // 创建一个 SettableFuture 实例
        SettableFuture<String> settableFuture = SettableFuture.create();

        // 添加一个监听器，监听 future 的完成情况
        settableFuture.addListener(() -> {
            try {
                // 获取并打印 future 的结果
                String result = settableFuture.get();
                System.out.println("Future completed with result: " + result);
            } catch (InterruptedException | ExecutionException e) {
                e.printStackTrace();
            }
        }, MoreExecutors.directExecutor());

        // 模拟异步操作并设置结果
        new Thread(() -> {
            try {
                // 模拟一些异步操作
                Thread.sleep(2000);
                // 设置 future 的结果
                settableFuture.set("Hello, SettableFuture!");
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }).start();

        // 主线程继续执行其他任务
        System.out.println("Main thread is doing other tasks...");

        // 在主线程中等待 future 完成（仅用于示例目的，不建议在生产代码中使用）
        try {
            String result = settableFuture.get();
            System.out.println("Main thread received future result: " + result);
        } catch (InterruptedException | ExecutionException e) {
            e.printStackTrace();
        }
    }
}
```

### 解释

1. **创建 `SettableFuture` 实例**：使用 `SettableFuture.create()` 创建一个新的 `SettableFuture` 对象。
2. **添加监听器**：使用 `addListener` 方法为 `SettableFuture` 添加一个监听器，该监听器将在 `SettableFuture` 完成时执行。这里我们使用 `MoreExecutors.directExecutor()` 直接在调用线程中执行监听器。
3. **模拟异步操作**：在一个新的线程中模拟异步操作。使用 `Thread.sleep` 模拟延迟，然后使用 `set` 方法设置 `SettableFuture` 的结果。
4. **主线程继续执行**：主线程继续执行其他任务，同时可以选择等待 `SettableFuture` 完成并获取结果。

### 注意事项

- `SettableFuture` 的 `set` 方法只能调用一次，后续的调用将会失败。
- 可以使用 `setException` 方法来设置异常，这样监听器中的 `get` 方法会抛出该异常。
- 监听器在 `SettableFuture` 完成时执行，可以在主线程或指定的执行器中运行。

通过以上示例，可以看到 `SettableFuture` 在处理异步任务时的灵活性和便利性，尤其适合需要手动控制任务结果的场景。





以下是一个更完整的Java示例，演示了如何使用Google Guava库中的`SettableFuture`来实现一个简单的异步任务处理流程，包括启动一个后台线程来执行任务、设置结果、以及在结果可用时通过监听器进行处理。

```java
import com.google.common.util.concurrent.SettableFuture;
import com.google.common.util.concurrent.MoreExecutors;
import java.util.concurrent.ExecutionException;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class SettableFutureExample {

    public static void main(String[] args) {
        // 创建一个SettableFuture实例，用于存储异步操作的结果
        final SettableFuture<Integer> futureResult = SettableFuture.create();

        // 创建一个固定大小的线程池来执行后台任务
        ExecutorService executor = Executors.newFixedThreadPool(1);

        // 提交一个Runnable任务到线程池
        executor.submit(new Runnable() {
            @Override
            public void run() {
                try {
                    // 模拟耗时操作
                    Thread.sleep(2000);
                    // 设置异步操作的结果
                    futureResult.set(42); // 假设这是任务完成后的结果
                } catch (InterruptedException e) {
                    // 如果在执行过程中遇到异常，设置异常结果
                    futureResult.setException(e);
                }
            }
        });

        // 添加监听器，当结果可用时（无论是成功还是失败），将自动调用该监听器
        futureResult.addListener(new Runnable() {
            @Override
            public void run() {
                try {
                    Integer result = futureResult.get(); // 阻塞等待结果
                    System.out.println("异步任务完成，结果是: " + result);
                } catch (InterruptedException | ExecutionException e) {
                    System.err.println("异步任务执行失败: " + e.getCause());
                }
            }
        }, MoreExecutors.directExecutor());

        // 关闭线程池
        executor.shutdown();
        System.out.println("主线程继续执行其他任务...");
    }
}
```

在这个例子中：

1. **创建`SettableFuture`**: 用来保存异步操作的结果。
2. **启动后台任务**: 使用`ExecutorService`提交一个`Runnable`到线程池执行，模拟一个耗时操作并最终设置`SettableFuture`的结果。
3. **添加监听器**: 通过`addListener`方法注册一个`Runnable`作为监听器，当`SettableFuture`的结果被设置时（无论成功还是失败），这个监听器会被执行。
4. **关闭线程池**: 在任务提交后，记得关闭线程池以释放资源。

请注意，此代码示例需要引入Guava库才能编译和运行。