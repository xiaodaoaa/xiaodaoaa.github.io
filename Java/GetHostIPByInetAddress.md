在Java中，可以使用`InetAddress`类来获取本机的IP地址。下面是一个简单的例子，展示如何获取本机的IP地址：

```java
import java.net.InetAddress;
import java.net.UnknownHostException;

public class Main {
    public static void main(String[] args) {
        try {
            // 获取本机的InetAddress实例
            InetAddress localHost = InetAddress.getLocalHost();

            // 获取IP地址
            String ipAddress = localHost.getHostAddress();

            // 打印IP地址
            System.out.println("本机IP地址: " + ipAddress);
        } catch (UnknownHostException e) {
            e.printStackTrace();
        }
    }
}
```

以上代码首先通过`InetAddress.getLocalHost()`方法获取一个表示本机地址的`InetAddress`对象，然后调用`getHostAddress()`方法获取本机的IP地址，并将其打印出来。

需要注意的是，如果你的机器有多个网络接口（例如有线网和无线网），`getLocalHost()`方法可能不会返回你期望的那个IP地址。在这种情况下，你可以通过`NetworkInterface`类来获取更多的网络接口信息，从而选择你需要的IP地址。以下是一个示例：

```java
import java.net.InetAddress;
import java.net.NetworkInterface;
import java.net.SocketException;
import java.util.Enumeration;

public class Main {
    public static void main(String[] args) {
        try {
            Enumeration<NetworkInterface> interfaces = NetworkInterface.getNetworkInterfaces();
            while (interfaces.hasMoreElements()) {
                NetworkInterface networkInterface = interfaces.nextElement();
                
                // 跳过回环接口和没有启用的接口
                if (networkInterface.isLoopback() || !networkInterface.isUp()) {
                    continue;
                }
                
                Enumeration<InetAddress> addresses = networkInterface.getInetAddresses();
                while (addresses.hasMoreElements()) {
                    InetAddress inetAddress = addresses.nextElement();
                    
                    // 仅获取IPv4地址
                    if (inetAddress instanceof java.net.Inet4Address) {
                        System.out.println("Interface: " + networkInterface.getDisplayName() + " IP: " + inetAddress.getHostAddress());
                    }
                }
            }
        } catch (SocketException e) {
            e.printStackTrace();
        }
    }
}
```

这个示例将会列出所有网络接口的IPv4地址。