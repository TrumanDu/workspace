# 用 github 来作为 maven 仓库发布 jar

好久没有写 java 代码了，持续更新一下自己的脚手架项目[smart-spring-boot-project](https://github.com/TrumanDu/smart-spring-boot-project)，发现有个更佳的实践，将脚手架以 jar 发布，脚手架项目以后升级版本就好了。

没有用中央仓库发布过 jar，听说也是比较麻烦，看到可以用 github 来实现，索性就来摸索一下。

简单看了一下发现有两种方式：

-   GitHub Packages registry
-   Github Repository 作为 Maven 仓库

接下来分别试试看。

## GitHub Packages registry

首先采用 GitHub Packages registry 来实现，免费空间 500MB。简单方便，唯一不足的就是使用的时候需要**权限验证**，这点有点不太好。这里的权限验证不是 Package,而是 github 账户的 token 验证，只要有一个读取包的权限既可下载 Apache Maven registry 的 jar。

### maven 配置文件

首先先在 maven 配置文件中增加 token 验证和仓库地址

```
<settings xmlns="http://maven.apache.org/SETTINGS/1.0.0"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.0.0
                      http://maven.apache.org/xsd/settings-1.0.0.xsd">

  <activeProfiles>
    <activeProfile>github</activeProfile>
  </activeProfiles>

  <profiles>
    <profile>
      <id>github</id>
      <repositories>
        <repository>
          <id>central</id>
          <url>https://repo1.maven.org/maven2</url>
        </repository>
        <repository>
          <id>github</id>
          <url>https://maven.pkg.github.com/trumandu/maven-repository</url>
          <snapshots>
            <enabled>true</enabled>
          </snapshots>
        </repository>
      </repositories>
    </profile>
  </profiles>

  <servers>
    <server>
      <id>github</id>
      <username>trumandu</username>
      <password>TOKEN</password>
    </server>
  </servers>
</settings>
```

### 待发布 jar 项目 pom

然后再待发布 jar 项目 pom 添加项目发布仓库地址

```
<distributionManagement>
        <repository>
            <id>github</id>
            <name>GitHub OWNER Apache Maven Packages</name>
            <url>https://maven.pkg.github.com/trumandu/maven-repository</url>
        </repository>
    </distributionManagement>
```

最后执行`maven deploy`既可。访问地址`https://github.com/TrumanDu/maven-repository/packages/`既可以看到相应的 jar.

### 使用发布的 jar

在需要使用的项目 pom 文件中添加如下内容：

```
<dependencies>
        <dependency>
            <groupId>top.trumandu</groupId>
            <artifactId>smart-rest-spring-boot</artifactId>
            <version>0.0.1-SNAPSHOT</version>
        </dependency>
    </dependencies>
    <repositories>
        <repository>
            <id>github</id>
            <name>GitHub OWNER Apache Maven Packages</name>
            <url>https://maven.pkg.github.com/trumandu/maven-repository</url>
        </repository>
    </repositories>
```

这里要注意的是，对于下载 github packages 需要权限验证，所以在 maven 配置文件中需要添加 token 信息。

```
 <servers>
    <server>
      <id>github</id>
      <username>trumandu</username>
      <password>TOKEN</password>
    </server>
  </servers>
```

通过如上简单的方法既可发布属于自己的 jar 到外网，以供任何人使用，切记需要添加验证信息。

## 使用 Github Repository 作为 Maven 仓库

### 配置 token

在 maven 配置文件 conf.xml 中增加 token 配置，这个 token 切记增加`user:email`权限。

```
<servers>
    <server>
      <id>github</id>
      <username>trumandu</username>
      <password>TOKEN</password>
    </server>
  </servers>
```

### 待发布 jar 项目 pom

```
<properties>
        <github.global.server>github</github.global.server>
</properties>
<!--1.作用：将jar deploy(发布)到本地储存库位置(altDeploymentRepository)-->
            <plugin>
                <artifactId>maven-deploy-plugin</artifactId>
                <configuration>
                    <altDeploymentRepository>internal.repo::default::file://${project.build.directory}/mvn-repo
                    </altDeploymentRepository>
                </configuration>
            </plugin>
            <!--2.作用：将本地存储库位置的jar文件发布到github上-->
            <plugin>
                <groupId>com.github.github</groupId>
                <artifactId>site-maven-plugin</artifactId>
                <version>0.12</version>
                <configuration>
                    <message>Maven artifacts for ${project.version}</message>
                    <noJekyll>true</noJekyll>
                    <!--本地jar相关文件地址，与上方配置储存库位置(altDeploymentRepository)保持一致-->
                    <outputDirectory>${project.build.directory}/mvn-repo</outputDirectory>
                    <!--配置上传到github哪个分支，此处配置格式必须以refs/heads/+分支名称-->
                    <branch>refs/heads/main</branch>
                    <merge>true</merge>
                    <includes>
                        <include>**/*</include>
                    </includes>
                    <!--对应github上创建的仓库名称 name-->
                    <repositoryName>maven-repository</repositoryName>
                    <!--github 仓库所有者即登录用户名-->
                    <repositoryOwner>TrumanDu</repositoryOwner>
                </configuration>
                <executions>
                    <execution>
                        <goals>
                            <goal>site</goal>
                        </goals>
                        <phase>deploy</phase>
                    </execution>
                </executions>
            </plugin>
```

执行命令发布到 github 仓库：`mvn clean deploy`

### 使用 jar

在任何 maven 项目 pom 文件中添加如下内容：

```
<dependencies>
        <dependency>
            <groupId>top.trumandu</groupId>
            <artifactId>smart-rest-spring-boot</artifactId>
            <version>0.0.2-SNAPSHOT</version>
        </dependency>
    </dependencies>
    <repositories>
        <repository>
            <id>github</id>
            <!-- 格式是 https://raw.githubusercontent.com/[github 用户名]/[github 仓库名]/[分支名]/repository -->
            <url>https://raw.githubusercontent.com/trumandu/maven-repository/main/repository</url>
            <snapshots>
                <enabled>true</enabled>
                <updatePolicy>always</updatePolicy>
            </snapshots>
        </repository>
    </repositories>
```

Enjoy!!!