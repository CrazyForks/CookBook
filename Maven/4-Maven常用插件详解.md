# Maven 常用插件详解

## 1. 插件概述

```
Maven 插件机制:

┌─────────────────────────────────────────────────────────────┐
│                    Maven 插件体系                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  插件（Plugin）→ 目标（Goal）→ 阶段（Phase）               │
│                                                             │
│  例: maven-compiler-plugin:compile 绑定到 compile 阶段     │
│                                                             │
│  核心插件:                                                  │
│  ├── maven-clean-plugin        清理                        │
│  ├── maven-compiler-plugin     编译                        │
│  ├── maven-surefire-plugin     测试                        │
│  ├── maven-jar-plugin          打包 JAR                    │
│  ├── maven-war-plugin          打包 WAR                    │
│  ├── maven-install-plugin      安装到本地仓库              │
│  └── maven-deploy-plugin       部署到远程仓库              │
│                                                             │
│  常用扩展插件:                                              │
│  ├── spring-boot-maven-plugin  Spring Boot 打包            │
│  ├── maven-shade-plugin        Fat JAR 打包                │
│  ├── maven-assembly-plugin     分发包打包                  │
│  ├── maven-source-plugin       源码打包                    │
│  ├── maven-javadoc-plugin      文档生成                    │
│  ├── maven-checkstyle-plugin   代码规范检查                │
│  ├── maven-pmd-plugin          静态分析                    │
│  ├── maven-spotbugs-plugin     Bug 检测                    │
│  ├── maven-enforcer-plugin     规则强制                    │
│  └── maven-release-plugin      版本发布                    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 2. 编译插件

```xml
<build>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-compiler-plugin</artifactId>
            <version>3.11.0</version>
            <configuration>
                <!-- Java 版本 -->
                <source>17</source>
                <target>17</target>
                <!-- 编码 -->
                <encoding>UTF-8</encoding>
                <!-- 编译器参数 -->
                <compilerArgs>
                    <arg>-parameters</arg>
                    <arg>-Xlint:unchecked</arg>
                </compilerArgs>
                <!-- 注解处理器 -->
                <annotationProcessorPaths>
                    <path>
                        <groupId>org.projectlombok</groupId>
                        <artifactId>lombok</artifactId>
                        <version>1.18.30</version>
                    </path>
                    <path>
                        <groupId>org.mapstruct</groupId>
                        <artifactId>mapstruct-processor</artifactId>
                        <version>1.5.5.Final</version>
                    </path>
                </annotationProcessorPaths>
                <!-- 发布到 Maven Central 需要 -->
                <release>17</release>
            </configuration>
        </plugin>
    </plugins>
</build>
```

## 3. 测试插件

```xml
<build>
    <plugins>
        <!-- Surefire 插件（单元测试） -->
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-surefire-plugin</artifactId>
            <version>3.2.2</version>
            <configuration>
                <!-- 包含的测试类 -->
                <includes>
                    <include>**/*Test.java</include>
                    <include>**/*Tests.java</include>
                    <include>**/*TestCase.java</include>
                </includes>
                <!-- 排除的测试类 -->
                <excludes>
                    <exclude>**/integration/**</exclude>
                </excludes>
                <!-- 系统属性 -->
                <systemPropertyVariables>
                    <java.awt.headless>true</java.awt.headless>
                </systemPropertyVariables>
                <!-- 并行执行 -->
                <parallel>methods</parallel>
                <threadCount>4</threadCount>
                <!-- 失败后继续 -->
                <testFailureIgnore>false</testFailureIgnore>
            </configuration>
        </plugin>
        
        <!-- Failsafe 插件（集成测试） -->
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-failsafe-plugin</artifactId>
            <version>3.2.2</version>
            <executions>
                <execution>
                    <goals>
                        <goal>integration-test</goal>
                        <goal>verify</goal>
                    </goals>
                </execution>
            </executions>
            <configuration>
                <includes>
                    <include>**/*IT.java</include>
                    <include>**/*IntegrationTest.java</include>
                </includes>
            </configuration>
        </plugin>
    </plugins>
</build>
```

## 4. 打包插件

### 4.1 Spring Boot 插件

```xml
<build>
    <plugins>
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
            <version>3.2.0</version>
            <executions>
                <execution>
                    <goals>
                        <goal>repackage</goal>  <!-- 重新打包为可执行 JAR -->
                    </goals>
                    <configuration>
                        <!-- 排除 provided 依赖 -->
                        <excludeDevtools>true</excludeDevtools>
                        <!-- 主类 -->
                        <mainClass>com.example.Application</mainClass>
                        <!-- 分层打包 -->
                        <layers>
                            <enabled>true</enabled>
                        </layers>
                    </configuration>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```

### 4.2 Shade 插件（Fat JAR）

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-shade-plugin</artifactId>
    <version>3.5.1</version>
    <executions>
        <execution>
            <phase>package</phase>
            <goals>
                <goal>shade</goal>
            </goals>
            <configuration>
                <!-- 主类 -->
                <transformers>
                    <transformer implementation="org.apache.maven.plugins.shade.resource.ManifestResourceTransformer">
                        <mainClass>com.example.Application</mainClass>
                    </transformer>
                    <!-- 合并 SPI 文件 -->
                    <transformer implementation="org.apache.maven.plugins.shade.resource.ServicesResourceTransformer"/>
                    <!-- 合并 spring.handlers -->
                    <transformer implementation="org.apache.maven.plugins.shade.resource.AppendingTransformer">
                        <resource>META-INF/spring.handlers</resource>
                    </transformer>
                    <transformer implementation="org.apache.maven.plugins.shade.resource.AppendingTransformer">
                        <resource>META-INF/spring.schemas</resource>
                    </transformer>
                </transformers>
                <!-- 排除依赖 -->
                <filters>
                    <filter>
                        <artifact>*:*</artifact>
                        <excludes>
                            <exclude>META-INF/*.SF</exclude>
                            <exclude>META-INF/*.DSA</exclude>
                            <exclude>META-INF/*.RSA</exclude>
                        </excludes>
                    </filter>
                </filters>
            </configuration>
        </execution>
    </executions>
</plugin>
```

### 4.3 Assembly 插件

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-assembly-plugin</artifactId>
    <version>3.6.0</version>
    <executions>
        <execution>
            <id>make-assembly</id>
            <phase>package</phase>
            <goals>
                <goal>single</goal>
            </goals>
            <configuration>
                <descriptors>
                    <descriptor>src/main/assembly/assembly.xml</descriptor>
                </descriptors>
                <finalName>${project.name}-${project.version}</finalName>
            </configuration>
        </execution>
    </executions>
</plugin>
```

```xml
<!-- assembly.xml -->
<assembly xmlns="http://maven.apache.org/plugins/maven-assembly-plugin/assembly/1.1.3"
          xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xsi:schemaLocation="http://maven.apache.org/plugins/maven-assembly-plugin/assembly/1.1.3 
          http://maven.apache.org/xsd/assembly-1.1.3.xsd">
    
    <id>bin</id>
    
    <formats>
        <format>tar.gz</format>
        <format>zip</format>
    </formats>
    
    <includeBaseDirectory>true</includeBaseDirectory>
    
    <fileSets>
        <!-- 配置文件 -->
        <fileSet>
            <directory>src/main/resources</directory>
            <outputDirectory>conf</outputDirectory>
        </fileSet>
        <!-- 启动脚本 -->
        <fileSet>
            <directory>src/main/bin</directory>
            <outputDirectory>bin</outputDirectory>
            <fileMode>0755</fileMode>
        </fileSet>
    </fileSets>
    
    <dependencySets>
        <dependencySet>
            <outputDirectory>lib</outputDirectory>
            <useProjectArtifact>true</useProjectArtifact>
        </dependencySet>
    </dependencySets>
</assembly>
```

## 5. 代码质量插件

```xml
<build>
    <plugins>
        <!-- Checkstyle 代码规范 -->
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-checkstyle-plugin</artifactId>
            <version>3.3.1</version>
            <configuration>
                <configLocation>checkstyle.xml</configLocation>
                <consoleOutput>true</consoleOutput>
                <failsOnError>true</failsOnError>
            </configuration>
            <executions>
                <execution>
                    <phase>validate</phase>
                    <goals>
                        <goal>check</goal>
                    </goals>
                </execution>
            </executions>
        </plugin>
        
        <!-- SpotBugs 静态分析 -->
        <plugin>
            <groupId>com.github.spotbugs</groupId>
            <artifactId>spotbugs-maven-plugin</artifactId>
            <version>4.8.1.0</version>
            <configuration>
                <effort>Max</effort>
                <threshold>Medium</threshold>
                <xmlOutput>true</xmlOutput>
            </configuration>
            <executions>
                <execution>
                    <phase>verify</phase>
                    <goals>
                        <goal>check</goal>
                    </goals>
                </execution>
            </executions>
        </plugin>
        
        <!-- JaCoCo 代码覆盖率 -->
        <plugin>
            <groupId>org.jacoco</groupId>
            <artifactId>jacoco-maven-plugin</artifactId>
            <version>0.8.11</version>
            <executions>
                <execution>
                    <goals>
                        <goal>prepare-agent</goal>
                    </goals>
                </execution>
                <execution>
                    <id>report</id>
                    <phase>test</phase>
                    <goals>
                        <goal>report</goal>
                    </goals>
                </execution>
                <execution>
                    <id>check</id>
                    <phase>verify</phase>
                    <goals>
                        <goal>check</goal>
                    </goals>
                    <configuration>
                        <rules>
                            <rule>
                                <element>BUNDLE</element>
                                <limits>
                                    <limit>
                                        <counter>LINE</counter>
                                        <value>COVEREDRATIO</value>
                                        <minimum>0.80</minimum>
                                    </limit>
                                </limits>
                            </rule>
                        </rules>
                    </configuration>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```

## 6. 版本发布插件

```xml
<build>
    <plugins>
        <!-- Release 插件 -->
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-release-plugin</artifactId>
            <version>3.0.1</version>
            <configuration>
                <tagNameFormat>v@{project.version}</tagNameFormat>
                <autoVersionSubmodules>true</autoVersionSubmodules>
                <useReleaseProfile>false</useReleaseProfile>
                <releaseProfiles>release</releaseProfiles>
                <goals>deploy</goals>
            </configuration>
        </plugin>
        
        <!-- Versions 插件（版本管理） -->
        <plugin>
            <groupId>org.codehaus.mojo</groupId>
            <artifactId>versions-maven-plugin</artifactId>
            <version>2.16.2</version>
        </plugin>
    </plugins>
</build>
```

```bash
# 版本发布流程
mvn release:prepare          # 准备发布
mvn release:perform          # 执行发布
mvn release:rollback         # 回滚

# 版本管理命令
mvn versions:display-dependency-updates   # 检查依赖更新
mvn versions:display-plugin-updates       # 检查插件更新
mvn versions:use-latest-versions          # 更新到最新版本
mvn versions:set -DnewVersion=2.0.0       # 设置版本号
```

## 7. Enforcer 插件

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-enforcer-plugin</artifactId>
    <version>3.4.1</version>
    <executions>
        <execution>
            <id>enforce</id>
            <goals>
                <goal>enforce</goal>
            </goals>
            <configuration>
                <rules>
                    <!-- Maven 版本要求 -->
                    <requireMavenVersion>
                        <version>3.6.0</version>
                    </requireMavenVersion>
                    
                    <!-- Java 版本要求 -->
                    <requireJavaVersion>
                        <version>17</version>
                    </requireJavaVersion>
                    
                    <!-- 禁止特定依赖 -->
                    <bannedDependencies>
                        <excludes>
                            <exclude>commons-logging:commons-logging</exclude>
                            <exclude>log4j:log4j</exclude>
                        </excludes>
                        <message>请使用 spring-boot-starter-logging 替代</message>
                    </bannedDependencies>
                    
                    <!-- 依赖收敛（版本一致） -->
                    <dependencyConvergence/>
                    
                    <!-- 禁止重复依赖 -->
                    <banDuplicatePomDependencyVersions/>
                </rules>
                <fail>true</fail>
            </configuration>
        </execution>
    </executions>
</plugin>
```

## 8. 插件执行配置

```xml
<build>
    <plugins>
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-antrun-plugin</artifactId>
            <version>3.1.0</version>
            <executions>
                <!-- 执行点 1: 编译后 -->
                <execution>
                    <id>after-compile</id>
                    <phase>compile</phase>
                    <goals>
                        <goal>run</goal>
                    </goals>
                    <configuration>
                        <target>
                            <echo>编译完成！</echo>
                        </target>
                    </configuration>
                </execution>
                
                <!-- 执行点 2: 打包前 -->
                <execution>
                    <id>before-package</id>
                    <phase>prepare-package</phase>
                    <goals>
                        <goal>run</goal>
                    </goals>
                    <configuration>
                        <target>
                            <echo>准备打包...</echo>
                        </target>
                    </configuration>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```
