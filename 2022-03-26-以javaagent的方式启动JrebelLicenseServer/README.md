受 [ja-netfilter](https://github.com/ja-netfilter/ja-netfilter) 使用的 javaagent 方式的启发，那是不是可以可以将 [JrebelLicenseServerforJava](https://gitee.com/gsls200808/JrebelLicenseServerforJava) 做成 javaagent 的方式呢？这样在无网的情况下也能使用，而且不用单独去启一个应用了，让 License Server 随着 IDEA 的启动而启动。下面尝试一下。

第一步：在 `com.vvvtimes.server.MainServer` 中加入 `premain` 方法

```java
public static void premain(String args, Instrumentation inst) {
    try {
        Map<String, String> arguments = parseArguments(new String[]{});
        String port = arguments.get("p");

        if (port == null || !port.matches("\\d+")) {
            port = "8081";
        }

        Server server = new Server(Integer.parseInt(port));
        server.setHandler(new MainServer());
        server.start();
    } catch (Exception e) {
        e.printStackTrace();
    }
}
```

第二步：修改 `pom.xml` 文件中 `maven-shade-plugin` 插件 `transformer` 配置

```xml
<transformer implementation="org.apache.maven.plugins.shade.resource.ManifestResourceTransformer">
  <manifestEntries>
    <Main-Class>com.vvvtimes.server.MainServer</Main-Class>
    <Premain-Class>com.vvvtimes.server.MainServer</Premain-Class>
  </manifestEntries>
</transformer>
```

第三步：运行 `mvn clean package` 并将 `JrebelBrainsLicenseServerforJava-1.0-SNAPSHOT.jar` 复制到一个目录，比如 `E:\jetbrains`

第四步：找到 IntelliJ IDEA 的 `idea64.exe.vmoptions` 文件并在最后加入 javaagent 配置

```
-javaagent:E:/jetbrains/JrebelBrainsLicenseServerforJava-1.0-SNAPSHOT.jar
```

除了端口问题其他看起来还不错。

完~
