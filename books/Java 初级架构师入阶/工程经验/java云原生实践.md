---
group:
  title: 工程经验
  order: 3
order: 3
---

# java 云原生实践

借助于GraalVM和springboot 3 ，我们可以将java应用编译成native image，脱离jre单独运行。

## 环境搭建

需要安装GraalVM和C++编译环境

### 安装GraalVM

下载[GraalVM](https://github.com/graalvm/graalvm-ce-builds/releases/tag/vm-22.3.0)，然后解压到相应目录，配置环境变量：

JAVA_HOME

```
D:\app\java\graalvm-ce-java17-22.3.0
```

Path

```
%JAVA_HOME%\bin
```

接着验证一下

```
> java -version
openjdk version "17.0.5" 2022-10-18
OpenJDK Runtime Environment GraalVM CE 22.3.0 (build 17.0.5+8-jvmci-22.3-b08)
OpenJDK 64-Bit Server VM GraalVM CE 22.3.0 (build 17.0.5+8-jvmci-22.3-b08, mixed mode, sharing)
```

最后安装`native-image`,由于功能网络环境限制，先[下载](https://github.com/graalvm/graalvm-ce-builds/releases/tag/vm-22.3.0)，然后使用离线安装：

```
gu install -L native-image-installable-svm-java17-windows-amd64-22.3.0.jar
```

### 安装 MSVC 

在windows中使用native-image需要安装msvc2017-15.9或以上版本，可使用vs安装工具安装所需组件,[vs下载地址](https://visualstudio.microsoft.com/zh-hans/downloads/)，选择社区版即可。

经实践，安装以下组件即可：

![image-20221102090223085](http://static.trumandu.top/image-20221102090223085.png)

环境变量配置

```
WK10_INCLUDE
C:\Program Files (x86)\Windows Kits\10\Include\10.0.20348.0

WK10_LIB
C:\Program Files (x86)\Windows Kits\10\Lib\10.0.20348.0

WK10_BIN
C:\Program Files (x86)\Windows Kits\10\bin\10.0.20348.0

INCLUDE
%WK10_INCLUDE%\ucrt;%WK10_INCLUDE%\um;%WK10_INCLUDE%\shared;%MSVC%\include;

LIB
%WK10_LIB%\um\x64;%WK10_LIB%\ucrt\x64;%MSVC%\lib\x64;

Path下新增
D:\Program Files\Microsoft Visual Studio\2022\Community\VC\Tools\MSVC\14.33.31629\bin\Hostx64\x64
%WK10_BIN%\x64
```



## 最小Demo

在springboot 3 版本已经支持通过aot技术生成GraalVM所需要的配置信息：

```
Resource hints (resource-config.json)

Reflection hints (reflect-config.json)

Serialization hints (serialization-config.json)

Java Proxy Hints (proxy-config.json)

JNI Hints (jni-config.json)
```

最小可运行demo,和之前版本保持一样，使用最新版就可以实现，这里放一下pom文件。

```
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>
	<parent>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-parent</artifactId>
		<version>3.0.0-RC1</version>
		<relativePath/> <!-- lookup parent from repository -->
	</parent>
	<groupId>com.example</groupId>
	<artifactId>demo</artifactId>
	<version>0.0.1-SNAPSHOT</version>
	<name>demo</name>
	<description>Demo project for Spring Boot</description>
	<properties>
		<java.version>17</java.version>
	</properties>
	<dependencies>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-web</artifactId>
		</dependency>

		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-test</artifactId>
			<scope>test</scope>
		</dependency>
	</dependencies>

	<build>
		<plugins>
			<plugin>
				<groupId>org.springframework.boot</groupId>
				<artifactId>spring-boot-maven-plugin</artifactId>
			</plugin>
		</plugins>
	</build>
	<repositories>
		<repository>
			<id>spring-milestones</id>
			<name>Spring Milestones</name>
			<url>https://repo.spring.io/milestone</url>
			<snapshots>
				<enabled>false</enabled>
			</snapshots>
		</repository>
	</repositories>
	<pluginRepositories>
		<pluginRepository>
			<id>spring-milestones</id>
			<name>Spring Milestones</name>
			<url>https://repo.spring.io/milestone</url>
			<snapshots>
				<enabled>false</enabled>
			</snapshots>
		</pluginRepository>
	</pluginRepositories>

</project>
```

代码：

```
@SpringBootApplication
@RestController
public class DemoApplication {

    @RequestMapping("/")
    public String hello() {
        return "Hello Word";
    }
    public static void main(String[] args) {
        SpringApplication.run(DemoApplication.class, args);
    }

}
```

生成native image命令：

```
mvn -Pnative native:compile -DskipTests
```

最后就可以直接运行native应用

```
PS E:\demo\native-demo> .\target\native-demo

  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::            (v3.0.0-RC1)

2022-11-02T13:58:16.011+08:00  INFO 11432 --- [  restartedMain] t.t.nativedemo.NativeDemoApplication     : Starting AOT-processed NativeDemoApplication using Java 17.0.5 on WXMIS121 with PID 11432 (E:\demo\native-demo\target\native-demo.exe started by td20 in E:\demo\native-demo)
2022-11-02T13:58:16.011+08:00  INFO 11432 --- [  restartedMain] t.t.nativedemo.NativeDemoApplication     : No active profile set, falling back to 1 default profile: "default"
2022-11-02T13:58:16.011+08:00  INFO 11432 --- [  restartedMain] .e.DevToolsPropertyDefaultsPostProcessor : Devtools property defaults active! Set 'spring.devtools.add-properties' to 'false' to disable
2022-11-02T13:58:16.011+08:00  INFO 11432 --- [  restartedMain] .e.DevToolsPropertyDefaultsPostProcessor : For additional web related logging consider setting the 'logging.level.web' property to 'DEBUG'
2022-11-02T13:58:16.133+08:00  INFO 11432 --- [  restartedMain] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat initialized with port(s): 8080 (http)
2022-11-02T13:58:16.134+08:00  INFO 11432 --- [  restartedMain] o.apache.catalina.core.StandardService   : Starting service [Tomcat]
2022-11-02T13:58:16.134+08:00  INFO 11432 --- [  restartedMain] o.apache.catalina.core.StandardEngine    : Starting Servlet engine: [Apache Tomcat/10.0.27]
2022-11-02T13:58:16.146+08:00  INFO 11432 --- [  restartedMain] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring embedded WebApplicationContext
2022-11-02T13:58:16.146+08:00  INFO 11432 --- [  restartedMain] w.s.c.ServletWebServerApplicationContext : Root WebApplicationContext: initialization completed in 134 ms
2022-11-02T13:58:16.150+08:00  WARN 11432 --- [  restartedMain] i.m.c.i.binder.jvm.JvmGcMetrics          : GC notifications will not be available because MemoryPoolMXBeans are not provided by the JVM
2022-11-02T13:58:16.182+08:00  INFO 11432 --- [  restartedMain] o.s.b.a.e.web.EndpointLinksResolver      : Exposing 1 endpoint(s) beneath base path '/actuator'
2022-11-02T13:58:16.192+08:00  INFO 11432 --- [  restartedMain] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat started on port(s): 8080 (http) with context path ''
2022-11-02T13:58:16.208+08:00  INFO 11432 --- [  restartedMain] t.t.nativedemo.NativeDemoApplication     : Started NativeDemoApplication in 0.229 seconds (process running for 0.238)
```



## 反射及资源文件，动态代理处理

### 反射

```
@RegisterReflectionForBinding(ReflectionRunner.Customer.class)
```

Demo：

```
@Component
@RegisterReflectionForBinding(ReflectionRunner.Customer.class)
class ReflectionRunner implements ApplicationRunner {
    private final ObjectMapper objectMapper;

    ReflectionRunner(ObjectMapper objectMapper) {
        this.objectMapper = objectMapper;
    }
    record Customer(Integer id, String name) {
    }

    @Override
    public void run(ApplicationArguments args) throws Exception {
        var json = """
                [
                 { "id" : 2, "name": "Dr. Syer"} ,
                 { "id" : 1, "name": "Jürgen"} ,
                 { "id" : 4, "name": "Olga"} ,
                 { "id" : 3, "name": "Violetta"}  
                ]
                """;
        var result = this.objectMapper.readValue(json, new TypeReference<List<Customer>>() {
        });
        System.out.println("there are " + result.size() + " customers.");
        result.forEach(System.out::println);
    }
}
```

还有一种RuntimeHintsRegistrar方式

```
@ImportRuntimeHints(DemoControllerRuntimeHints.class)

	static class DemoControllerRuntimeHints implements RuntimeHintsRegistrar {
		@Override
		public void registerHints(RuntimeHints hints, ClassLoader classLoader) {
			hints.reflection()
					.registerConstructor(
							SimpleHelloService.class.getConstructors()[0], ExecutableMode.INVOKE)
					.registerMethod(ReflectionUtils.findMethod(
							SimpleHelloService.class, "sayHello", String.class), ExecutableMode.INVOKE);
		}
	}
```



### 资源文件

```
@ImportRuntimeHints(DemoControllerRuntimeHints.class)

	static class DemoControllerRuntimeHints implements RuntimeHintsRegistrar {

		@Override
		public void registerHints(RuntimeHints hints, ClassLoader classLoader) {
			hints.resources().registerPattern("hello.txt");
		}

	}
```

### 动态代理

```
  @ImportRuntimeHints(Edge2Application.Hints.class)  
    static class Hints implements RuntimeHintsRegistrar {

        @Override
        public void registerHints(RuntimeHints hints, ClassLoader classLoader) {
            hints
                    .proxies()
                    .registerJdkProxy(
                            com.example.edge2.CrmClient.class, org.springframework.aop.SpringProxy.class,
                            org.springframework.aop.framework.Advised.class, DecoratingProxy.class
                    );
        }
    }
```



## 参考

1. [使用Graalvm简单编译native-image](https://blog.csdn.net/qq_26212181/article/details/121418452?spm=1001.2101.3001.6650.1)
2. [Visual Studio 2019 配置 MSVC 环境变量，使用命令行编译](https://www.jianshu.com/p/7fab25165f4b)
3. [demo-aot-native](https://github.com/snicoll/demo-aot-native)