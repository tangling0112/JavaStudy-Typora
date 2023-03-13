# 子项目`pom.xml`中添加`devtools`依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-devtools</artifactId>
    <scope>runtime</scope>
    <optional>true</optional>
</dependency>
```

# 父项目`pom.xml`中导入`Maven`插件

> **当然用子项目导入也可以,只不过父项目导入了,所有的子项目就不用导入了**

```xml
<build>
    <finalName>SpringCloudTest</finalName>
    <plugins>
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
            <configuration>
                <addResources>true</addResources>
            </configuration>
        </plugin>
    </plugins>
</build>
```

# `IDEA`开启自动编译

<img src="https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202303132318753.png" alt="image-20230313231858386" style="zoom: 80%;" />

<img src="https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202303132334657.png" alt="image-20230313233412286" style="zoom:80%;" />
