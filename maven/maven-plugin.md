## maven插件

### 编译

#### maven-compiler-plugin

maven是个项目管理的工具，maven默认会使用比较低版本的jdk进行打包，这样容易造成编译问题。
使用maven-compiler-plugin可以指定一下属性：

* source及target的jdk版本
* utf-8编码
* 是否展示警告等等

### 打包

#### mvn package(maven-jar-plugin)

maven自带的打包工具，不能打包成可直接运行的jar

#### 强大的打包工具：maven-assembly-plugin

The Assembly Plugin for Maven is primarily intended to allow users to aggregate the project output along with its dependencies, modules, site documentation, and other files into a single distributable archive.

简单来说，assembly就是将项目的依赖，模块，文档和其他资源打包到一个归档中的工具

示例：

```xml
<build>
    <plugins>
        <plugin>       
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-assembly-plugin</artifactId>
            <executions>
              
              <execution><!-- 配置执行器 -->
              <id>make-assembly</id>
                <phase>package</phase><!-- 绑定到package生命周期阶段上 -->
                (1)
                <goals>
                  <goal>single</goal><!-- 只运行一次 -->   
                </goals>
                
                (2)
                <configuration>
                  <finalName>${project.name}</finalName>
                  <!--主类入口等-->
                  ...
                  
                  <descriptor>src/main/assembly/assembly.xml</descriptor><!--配置描述文件路径--> 
                </configuration>
              
              </execution>
           </executions>
        </plugin>
     </plugins>
</build>
```

可以直接使用mvn package运行

使用assembly，你可以

* 可以打包成多种形式：zip,tar,tar.gz,jar,dir,war  and any other format that the ArchiveManager has been configured for 
* 指定打包范围：bin, jar-with-dependencies, src, project
* 通过自定义Assembly Descriptor灵活定义打包方法 

#### 终极神器：maven-shade-plugin

maven-assembly-plugin虽然很强大，但是不能很好的解决包冲突的问题，比如打好的包要运行在容器中，包中的某个jar和容器中的某个jar发生了冲突，这时就需要maven-shade-plugin出场了。

maven-shade-plugin是一个maven打包工具，可以重命名包中一些依赖的包名，从而比较好的解决包冲突的问题

```xml
<plugin>
   <groupId>org.apache.maven.plugins</groupId>
   <artifactId>maven-shade-plugin</artifactId>
   <version>3.2.1</version>
   <executions>
       <execution>
           <phase>package</phase>
           <goals>
               <goal>
                   shade
               </goal>
           </goals>
           <configuration>
               <transformers>
                   <transformer implementation="org.apache.maven.plugins.shade.resource.ManifestResourceTransformer">
                       <mainClass>com.tigerbrokers.zeus.commander.core.flink.FlinkExecutor</mainClass>
                   </transformer>
               </transformers>
           </configuration>
       </execution>
   </executions>
</plugin>
```

使用mvn shade:shade进行打包

### spring-boot的maven插件

#### spring-boot-maven-plugin

spring-boot的maven插件为spring-boot应用提供了mvn操作的可能
spring-boot的maven插件可以：
* spring-boot:repackage: 将spring-boot打成jar包，jar包可以直接以spring-boot的方式运行，mvn package的包添加后缀.origin
* spring-boot:run: 直接本地运行spring-boot服务
* spring-boot:start: 在mvn integration-test阶段，进行Spring Boot应用生命周期的管理
* spring-boot:stop: 在mvn integration-test阶段，进行Spring Boot应用生命周期的管理
* spring-boot:build-info: 生成Actuator使用的构建信息文件build-info.properties

```xml
<build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
                <version>2.0.1.RELEASE</version>
                <executions>
                    <execution>
                        <goals>
                            <goal>repackage</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
```
