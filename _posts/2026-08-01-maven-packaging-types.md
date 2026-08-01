---
layout: post
title: "Maven Packaging 类型详解"
date: 2026-08-01 23:30:00 +0800
description: "介绍 Maven 中不同的 packaging 类型及其用途"
tags: java maven 构建工具
categories: 笔记
---

Maven 的 `<packaging>` 元素决定了项目的构建产物类型。它直接影响 Maven 的生命周期绑定--不同的 packaging 类型会触发不同的构建步骤。本文逐一介绍各类型。

## packaging 类型一览

| 类型 | 产物 | 说明 |
|------|------|------|
| `jar` | `.jar` | 默认类型，打包 Java 类文件 |
| `pom` | 无 | 不产出构件，声明聚合或继承关系 |
| `war` | `.war` | Web 应用，部署到 Servlet 容器 |
| `ear` | `.ear` | Java EE 企业应用，包含多个模块 |
| `maven-plugin` | `.jar` | Maven 插件，包含 Mojo 实现 |
| `ejb` | `.jar` | EJB 打包（现代项目已少用） |
| `rar` | `.rar` | 资源适配器，连接外部 EIS |
| `bundle` | `.jar` | OSGi bundle（需 maven-bundle-plugin） |

---

## 1. jar（默认）

最常用的类型。如果不显式指定 `<packaging>`，Maven 默认就是 `jar`。

```xml
<project>
    <groupId>com.tangerine</groupId>
    <artifactId>my-lib</artifactId>
    <version>1.0.0</version>
    <packaging>jar</packaging> <!-- 可省略，默认就是 jar -->
</project>
```

**绑定的生命周期阶段：**
- `process-classes` -> maven-compiler-plugin 编译
- `package` -> maven-jar-plugin 打包成 `.jar`
- `install` -> 安装到本地仓库

---

## 2. pom

`pom` 类型不产出任何构件。它用在两个场景：

### 2.1 聚合（多模块项目）

```xml
<project>
    <groupId>com.tangerine</groupId>
    <artifactId>my-project</artifactId>
    <version>1.0.0</version>
    <packaging>pom</packaging>

    <modules>
        <module>my-domain</module>
        <module>my-web</module>
        <module>my-service</module>
    </modules>
</project>
```

父 POM 声明 `pom` 类型后，`mvn install` 会依次构建所有子模块。

### 2.2 继承（父 POM 共享配置）

```xml
<!-- 父 POM -->
<project>
    <groupId>com.tangerine</groupId>
    <artifactId>my-parent</artifactId>
    <version>1.0.0</version>
    <packaging>pom</packaging>

    <properties>
        <java.version>17</java.version>
        <maven.compiler.source>17</maven.compiler.source>
        <maven.compiler.target>17</maven.compiler.target>
    </properties>

    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-dependencies</artifactId>
                <version>3.2.0</version>
                <type>pom</type>
                <scope>import</scope>
            </dependency>
        </dependencies>
    </dependencyManagement>
</project>
```

子模块继承父 POM：

```xml
<project>
    <parent>
        <groupId>com.tangerine</groupId>
        <artifactId>my-parent</artifactId>
        <version>1.0.0</version>
    </parent>
    <artifactId>my-domain</artifactId>
    <!-- packaging 默认 jar -->
</project>
```

> **关键点**：`pom` 类型的项目只运行 `install`/`deploy` 来安装或发布 POM 文件本身，不执行编译和打包。

---

## 3. war

Web 应用归档，部署到 Tomcat、Jetty 等 Servlet 容器。

```xml
<project>
    <groupId>com.tangerine</groupId>
    <artifactId>my-web</artifactId>
    <version>1.0.0</version>
    <packaging>war</packaging>

    <dependencies>
        <dependency>
            <groupId>jakarta.servlet</groupId>
            <artifactId>jakarta.servlet-api</artifactId>
            <version>6.0.0</version>
            <scope>provided</scope>
        </dependency>
    </dependencies>
</project>
```

**与 jar 的区别：**
- 额外绑定 `maven-war-plugin`，将编译结果连同 `src/main/webapp/` 下的内容一起打包
- 产物结构符合 WAR 规范：`WEB-INF/classes/`、`WEB-INF/lib/`、`WEB-INF/web.xml`

**目录结构：**
```
my-web/
├── src/
│   ├── main/
│   │   ├── java/          # Java 源码
│   │   ├── resources/     # 配置文件
│   │   └── webapp/        # Web 资源
│   │       ├── WEB-INF/
│   │       │   └── web.xml
│   │       ├── index.html
│   │       └── static/
│   └── test/
└── pom.xml
```

> **Spring Boot 提示**：Spring Boot 项目通常用 `jar`（内嵌容器），只有需要部署到外部 Tomcat 时才用 `war`。切换时还需要将启动类继承 `SpringBootServletInitializer`。

---

## 4. ear

企业级应用归档，把多个 `war`、`jar`、`ejb` 等模块组装成一个 `.ear` 文件，部署到 Java EE 应用服务器（如 WildFly、WebLogic）。

```xml
<project>
    <groupId>com.tangerine</groupId>
    <artifactId>my-ear</artifactId>
    <version>1.0.0</version>
    <packaging>ear</packaging>

    <dependencies>
        <dependency>
            <groupId>com.tangerine</groupId>
            <artifactId>my-web</artifactId>
            <version>1.0.0</version>
            <type>war</type>
        </dependency>
        <dependency>
            <groupId>com.tangerine</groupId>
            <artifactId>my-service</artifactId>
            <version>1.0.0</version>
            <type>jar</type>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-ear-plugin</artifactId>
                <version>3.3.0</version>
                <configuration>
                    <generateApplicationXml>true</generateApplicationXml>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>
```

> **现代项目少用**：微服务架构下，EAR 已不常见。了解即可。

---

## 5. maven-plugin

开发自定义 Maven 插件时的类型。产物是 `.jar`，但内部包含 Mojo 类（带 `@Mojo` 注解的插件目标实现）。

```xml
<project>
    <groupId>com.tangerine</groupId>
    <artifactId>my-maven-plugin</artifactId>
    <version>1.0.0</version>
    <packaging>maven-plugin</packaging>

    <dependencies>
        <dependency>
            <groupId>org.apache.maven</groupId>
            <artifactId>maven-plugin-api</artifactId>
            <version>3.9.6</version>
        </dependency>
        <dependency>
            <groupId>org.apache.maven.plugin-tools</groupId>
            <artifactId>maven-plugin-annotations</artifactId>
            <version>3.11.0</version>
            <scope>provided</scope>
        </dependency>
    </dependencies>
</project>
```

一个简单的 Mojo：

```java
@Mojo(name = "greet", defaultPhase = LifecyclePhase.PACKAGE)
public class GreetMojo extends AbstractMojo {
    @Parameter(property = "greet.name", defaultValue = "Tangerine")
    private String name;

    @Override
    public void execute() throws MojoExecutionException {
        getLog().info("Hello, " + name + "!");
    }
}
```

使用方式：

```bash
mvn com.tangerine:my-maven-plugin:1.0.0:greet
# 或
mvn com.tangerine:my-maven-plugin:1.0.0:greet -Dgreet.name=World
```

> **注意**：`maven-plugin` 类型会额外绑定 `maven-plugin-plugin`，生成插件描述符和文档。

---

## 6. ejb

EJB 打包，产物是 `.jar`，但会额外生成 EJB 部署描述符。现代项目（Jakarta EE 9+）已基本不需要这个类型，直接用 `jar` 配合注解即可。

---

## 7. rar

资源适配器归档（Resource Adapter Archive），用于连接外部企业信息系统（EIS），如 SAP、CICS 等。部署到 Java EE 应用服务器。

```xml
<project>
    <groupId>com.tangerine</groupId>
    <artifactId>my-rar</artifactId>
    <version>1.0.0</version>
    <packaging>rar</packaging>
</project>
```

> 使用场景非常窄，除非你在做 JCA 连接器开发，否则不会遇到。

---

## 8. bundle（OSGi）

不是 Maven 内置类型，需要 `maven-bundle-plugin` 支持。打包出符合 OSGi 规范的 JAR（包含特殊的 `MANIFEST.MF`）。

```xml
<project>
    <groupId>com.tangerine</groupId>
    <artifactId>my-bundle</artifactId>
    <version>1.0.0</version>
    <packaging>bundle</packaging>

    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.felix</groupId>
                <artifactId>maven-bundle-plugin</artifactId>
                <version>5.1.9</version>
                <extensions>true</extensions>
                <configuration>
                    <instructions>
                        <Bundle-SymbolicName>${project.groupId}.${project.artifactId}</Bundle-SymbolicName>
                        <Export-Package>com.tangerine.api</Export-Package>
                    </instructions>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>
```

---

## 生命周期绑定总结

不同 packaging 类型绑定的插件不同：

| 阶段 | jar | war | pom | maven-plugin |
|------|-----|-----|-----|-------------|
| `compile` | compiler | compiler | - | compiler |
| `test` | surefire | surefire | - | surefire |
| `package` | jar | war | - | plugin (descriptor) + jar |
| `install` | install | install | install | install |
| `deploy` | deploy | deploy | deploy | deploy |

> `pom` 类型跳过编译和打包阶段，只执行 `install`/`deploy`。
> `war` 类型在 `package` 阶段用 `maven-war-plugin` 替代 `maven-jar-plugin`。

---

## 实际选择指南

| 场景 | 选择 |
|------|------|
| 普通库/工具类 | `jar` |
| 多模块父项目 | `pom` |
| 传统 Web 应用（外部 Tomcat） | `war` |
| Spring Boot 应用（内嵌容器） | `jar` |
| 自定义 Maven 插件 | `maven-plugin` |
| Java EE 企业应用 | `ear`（几乎不用了） |

---

## 自定义 packaging

通过配置 `<extensions>true</extensions>` 的插件，可以注册新的 packaging 类型。例如 `maven-bundle-plugin` 注册了 `bundle` 类型。核心机制是 `PluginDescriptor` 中的 `lifecycleMapping`。

---

## 总结

Maven 的 packaging 类型本质上是**生命周期绑定的预设方案**。选择正确的类型，Maven 就会自动绑定对应的插件和阶段。日常开发中，`jar` 和 `pom` 覆盖了 90% 的场景，`war` 在传统 Web 项目中常见，其余类型按需使用。

---

> 参考：[Maven POM Reference - Packaging](https://maven.apache.org/pom.html#Packaging)
