# `SpringBoot`外部化配置

## 什么是`SpringBoot`外部化配置?

> - 我们知道，我们可以通过在配置文件中配置一些特定的属性的方式来修改我们的`SpringBoot`的作用方式.例如我们借助`server.port`属性就可以修改我们的`SpringBoot`应用程序监听的端口
> - 这种不使用我们的`Java`代码来对我们的`SpringBoot`的工作方式进行修改的方法就是我们所说的`SpringBoot`外部化配置.外部化也就是在我们的`Java`代码之外.

## 常用的`SpringBoot`外部化配置途径有哪些?

> **以下的途径按照优先级由低到高排序**

- :one:**`SpringBoot`默认属性值**
    - 既我们通过`SpringApplication.setDefualtProperties()`方法所设置的属性值
- :two:**`@Configuration`类上的`@PropertySource`注解所设置的属性值**
    -   注意，这样的属性源直到`Application Context`被刷新时才会被添加到环境中。这对于配置某些属性来说已经太晚了，比如`logging.*`和`spring.main.*`，它们在刷新开始前就已经被读取了。  
- :three:**默认配置文件以及被激活的特定配置文件中所设置的属性的值**
- **:four:`RandomValuePropertySource`,它只有`random.*`属性**  
- :five:**根据操作系统环境变量所配置的属性值**
- :six:**`Java System properties` (`System.getProperties()`)**
- :seven:**`java:comp/env` 中的 `JNDI` 属性**  
- **:eight:`ServletContext init parameters`**
- :nine:**`ServletConfig init parameters`**
- ![image-20230226182817635](https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202302261828705.png)

## `SpringBoot`的基于默认查找路径下的配置文件的属性配置

### `SpringBoot`默认情况下的配置文件的查找加载位置

> 下面的目录是按照其内部配置文件的优先级由低到高排序的.
>
> - 既如果我们的`classpath`根路径下的`properties`配置文件中设置了`server.port=8080`,而当前`SpringBoot`项目根目录下的`properties`配置文件中设置了`server.port=7777`,那么最终我们的`server.port`的值会被设置为`7777`
> - **注意**:`.properties`只能覆盖`.properties`路线下的配置文件,`.yaml`路线下的配置文件只能覆盖`.yaml`路线下的配置文件

- :one:`classpath`根路径

- :two:`classpath:/config`目录

- :three:当前`SpringBoot`项目所在目录

- :four:当前`SpringBoot`项目所在目录下的`/config`目录

- :five:当前`SpringBoot`项目所在目录下的`/config`目录下的所有直接子目录

- ```java
    optional:classpath:/
    optional:classpath:/config/
    optional:file:./
    optional:file:./config/
    optional:file:./config/*/
    ```

> ![image-20230226184240606](https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202302261842637.png)

### 常规情况下`SpringBoot`各个配置文件的优先级排序(**不使用`location`,`extra-location`,`import`的情况下**)

> **优先级从低到高**

- **`Jar`包内部**的`SpringBoot`的**默认配置文件**,也就是`application.properties`和`application.yaml`两个配置文件
- **`Jar`包内部**的`SpringBoot`的**被激活的特定配置文件**,也就是被激活的`application-工作环境名.properties`以及`application-工作环境名.yaml`配置文件
- **`Jar`包外部**的`SpringBoot`的默认配置文件,也就是`application.properties`和`application.yaml`两个配置文件
- **`Jar`包外部**的`SpringBoot`的**被激活的特定配置文件**,也就是被激活的`application-工作环境名.properties`以及`application-工作环境名.yaml`配置文件

> **注意:细分的优先级从低到高为**
>
> - **`Jar`内部**
>
>     - `application.yaml`
>
>     - `application.properties`
>     
>- `application-工作环境名.yaml`
>     - `application-工作环境名.properties`
>
> - **`Jar`外部**
>
>     - `application.yaml`
>
>     - `application.properties`
>
>     - `application-工作环境名.yaml`
>
>     - `application-工作环境名.properties`

### `Jar`包内部和`Jar`包外部是指的什么?

> - `Jar`包内部就是指的我们的`SpringBoot`项目目录,我们的`SpringBoot`项目目录中的内容在经过`Jar`打包之后,就会变为一个`Jar`包,这也就是为什么我们称`SpringBoot`的项目目录为`Jar`包内部的原因
> - `Jar`包外部就是指我们存储在我们的`Jar`包所在目录下.

<img src="https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202302261839878.png" alt="image-20230226183912831" style="zoom: 80%;" />

## `SpringBoot`配置文件的查找加载规则的自定义化配置

### 相关`SpringBoot`配置属性详细介绍

#### **`spring.config.name`**

- **作用**

    - 我们知道我们的`SpringBoot`无论是加载默认配置文件还是加载特定配置文件,都有一个指定的名称,也就是`application`
    - 实际上这个是可以修改的,借助于`spring.config.name`属性我们就可以对这个指定名称进行修改,如`spring.config.name=myproperties`,那么我们的`SpringBoot`在加载配置文件时就会使用`myproperties`去查找加载.比如`myproperties.properties`,`myproperties.yaml`,`myproperties-工作环境名.properties`,`myproperties-工作环境名.yaml`

- ```shell
    java -jar myproject.jar --spring.config.name=myproject
    ```

****

#### **`spring.config.location`**

- **作用**

    - 我们知道我们的`SpringBoot`加载配置文件是有默认的查找加载路径的
    - 同样的这个查找加载的路径我们也是可以修改的,借助于`spring.config.location`我们就可以去进行修改
    - **注意**
        - 使用`spring.config.location`属性是**覆盖配置的**,我们使用了这个属性进行配置,那么我们的`SpringBoot`的**默认查找加载路径中的配置文件就不再会被我们的`SpringBoot`加载**了,**而是只有我们`spring.config.location`属性所指出的路径下的配置文件才会被加载**

- **示例**

    - ```java
        java -jar myproject.jar --spring.config.location=\
        optional:classpath:custom-config/,\
        optional:file:./custom-config/,\
        optional:classpath:/override.properties
        ```

- **此时的搜索的路径为**

    - ```java
        //这些就是我们通过extra-location额外添加的搜寻路径
        optional:classpath:custom-config/
        optional:file:./custom-config/
        optional:classpath:/override.properties
        ```

****

#### **`spring.config.extra-location`**

- **作用**

    - 作用类似于我们的`spring.config.location`,只不过它不是覆盖的,也就是说**不会覆盖掉**我们的`SpringBoot`对于默认查找加载路径下的配置文件的加载.

- **示例**

    - ```java
        java -jar myproject.jar --spring.config.extra-location=\
        optional:classpath:custom-config/,\
        optional:file:./custom-config/,\
        optional:classpath:/override.properties
        ```

- **此时的搜集的路径为**

    - ```java
        //这些就是我们的SpringBoot默认会去查找的目录
        optional:classpath:/
        optional:classpath:/config/
        optional:file:./
        optional:file:./config/
        optional:file:./config/*/
        
        //这些就是我们通过extra-location额外添加的搜寻路径
        optional:classpath:custom-config/
        optional:file:./custom-config/
        optional:classpath:/override.properties
        ```

****

#### **`spring.config.import  `**

- **作用**

    - 前面我们的`spring.config.location`以及`spring.config.extra-location`是用来指定一些查找路径,来告诉我们的`SpringBoot`去哪些路径下去查找我们的`SpringBoot`配置文件.并且具体的查找加载时,还是会按照`application.properties`这样的固定文件名去查找,假如我们的查找路径下有一个`myOnlyProperties.yaml`配置文件,那么我们的`SpringBoot`是不会加载它的,因为其名称不符合规范
    - 此时就轮到我们的`spring.config.import`登场了,借助于这个`SpringBoot`配置属性,我们可以显式地指定一个配置文件,然后我们的`SpringBoot`就会去加载这个配置文件,即便这个文件的命名不符合规范

- **示例**

    - ```properties
        spring.config.import=file:./myOnlyProperties.yaml,file:./myOnlyProperties.properties
        ```

    - ```yaml
        spring:
        	config:
        		import:
        			- "optional:file:./myOnlyProperties.yaml"
        			- "optional:file:./myOnlyProperties.properties"
        ```

- **注意**

    - 不同于前面的`location`与`extra-location`两大属性指定的可以是路径也可以是一个具体的文件,我们的`spring.config.import`只能被赋值为一个具体的`.yaml`或`.properties`文件路径,而不能是一个目录路径
    - 假如我们指定了要导入`optional:file:./myOnlyProperties.yaml`,并且此时我们的`SpringBoot`激活了`test`工作环境,那么我们的`SpringBoot`就会额外地自动去导入`optional:file:./myOnlyProperties-test.yaml`配置文件

- **`spring.config.on-not-found`**

    - **作用**

### 属性使用的注意事项

- :one:**`spring.config.location`与`spring.config.extra-location`相关**

    - 目录路径末尾必须添加`/`

    - **`optional`的使用**

        - 默认情况下,如果我们指定的查找加载路径下没有我们的`application.properties`这样的配置文件可以加载,那么`SpringBoot`会抛出异常,我们的`SpringBoot`应用也就无法正确开启.

        - 我们可以在使用**`spring.config.location`与`spring.config.extra-location`**去配置我们的的查找加载路径时可以给我们的路径加上`optional:`前缀,这样即便在这个目录下找不到配置文件也不会爆异常.

        - ```java
            optional:classpath:custom-config/
            optional:file:./custom-config/
            ```
    
- :two:**`spring.config.import`导入的配置文件的优先级高于`spring.config.location`以及`spring.config.extra-location`导入的配置文件**

- :three:**无论什么情况下,对于`import`,`location`,`extra-location`而言,我们的`SpringBoot`默认查找路径下的配置文件的优先级总是低于它们.**

## 基于命令行的`SpringBoot`属性配置

> 我们可以在使用命令行运行我们的`SpringBoot`项目打包的`Jar`文件时通过`--属性名=属性值`的方式来对我们的`SpringBoot`属性进行配置.只要是我们的`SpringBoot`配置文件可以配置的我们的命令行都可以实现配置

- `java -jar xxx.jar --server.port=6666 --person.name=tangling --person.age=18`

### **注意**

> 我们可以在我们的`SpringBoot`应用中通过使用`SpringApplication.setAddCommandLineProperties(false)  `来禁用我们的基于命令行的`SpringBoot`属性配置功能.在禁用掉之后,我们上面所说的基于命令行的配置就不再能够生效

### 在命令行中使用`JSON Application Properties`

> **应用场景**
>
> - 我们在使用命令行来设置属性时,我们只是通过`--`的方式进行设置,这种方式往往有多种限制.因此我们的的`SpringBoot`引入了`JSON Application Properties`来解决这一问题

```shell
SPRING_APPLICATION_JSON='{"my":{"name":"test"}}' java -jar myapp.jar
```

```shell
java -jar myapp.jar --spring.application.json='{"my":{"name":"test"}}'
```

**效果与`java -jar mysqpp.jar --my.name=test`相同**

## 通配符的使用

> **应用场景**
>
> - 

<img src="https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202302261850330.png" alt="image-20230226185059292" style="zoom:67%;" />

## 指定`SpringBoot`配置文件查找路径时**`;`与`,`的使用**

> 我们可以形象地理解为
>
> - 

<img src="https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202302270951233.png" alt="image-20230227095130163" style="zoom:67%;" />

## `SpringBoot`各个配置文件优先级排序

> - **不讨论`Jar`包外部**
>
> - **优先级从低到高排序**
> - **激活环境为`E,H`**

- :one:**默认查找路径**
    - `application.yaml`
    - `application.properties`

    - `application-E.yaml`

    - `application-E.properties`
    - `application-H.yaml`
    - `application-H.properties`
- :two:**`spring.config.extra-location`指定的路径或配置文件**(`路径A,路径B;路径C,路径D)`
    - **路径A**
        - `application.yaml`
        - `application.properties`

        - `application-E.yaml`
        - `application-E.properties`
        - `application-H.yaml`
        - `application-H.properties`
    - **路径B与路径C**
        - 路径B`application.yaml`
        - 路径C`application.yaml`
        - 路径B`application.properties`
        - 路径C`application.properties`
        - 路径B`application-E.yaml`
        - 路径C`application-E.yaml`
        - 路径B`application-E.properties`
        - 路径C`application-E.properties`
        - 路径B`application-H.yaml`
        - 路径C`application-H.yaml`
        - 路径B`application-H.properties`
        - 路径C`application-H.properties`
    - **路径D**
        - `application.yaml`
        - `application.properties`

        - `application-E.yaml`
        - `application-E.properties`
        - `application-H.yaml`
        - `application-H.properties`
- :three:**`spring.config.import`指定的配置文件**
    - 第一个配置文件
    - 第二个配置文件
    - …
    - **注意**:如果第一个配置文件为`.properties`,第二个为`.yaml`,那么这个`.yaml`配置文件的优先级就会高于`.properties`配置文件

## `SpringBoot`导入无扩展名的文件

> **什么是无扩展名?**
>
> - <img src="https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202302270115657.png" alt="image-20230227011527604" style="zoom:67%;" />

## `SpringBoot`使用`configtree`配置树

<img src="https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202302270117903.png" alt="image-20230227011714832" style="zoom:67%;" />

<img src="https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202302270117935.png" alt="image-20230227011731886" style="zoom:67%;" />

## **`SpringBoot`使用属性占位符**

<img src="https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202302270120243.png" alt="image-20230227012009190" style="zoom:67%;" />

## `Spring Boot`多文档配置文件搭配`spring.config.activation.on-profile`与`spring.config.activation.on-cloudplatform`配置属性

### 什么是多文档文件?

> 多文档文件即通过一定的方式将物理上的一个`.properties`或`.yaml`配置文件划分为逻辑上的多个配置文件.

#### `Properties`

```yaml
person.name="tangling"
person.age=18
#---
person.name="zhangsan"
book.title="The World!""
```

#### `YAML`

```Yaml
person:
	name: "tangling"
	age: 18
---
person:
	name: "zhangsan"
book:
	title: "The World!"
```

### 示例

> **以下示例的效果为**
>
> - 只有当我们的`cloud-platform=kubernetes`,`on-profile=prod或staging`时我们的`myotherprop=sometimes-set`这个属性配置才会生效.
> - 而我们的`myprop=always-set`这个属性配置则是不受`cloud-platform=kubernetes`,`on-profile=prod或staging`所约束的.

```properties
myprop=always-set
#---
spring.config.activate.on-cloud-platform=kubernetes
spring.config.activate.on-profile=prod | staging
myotherprop=sometimes-set
```

```yaml
myprop:
"always-set"
---
spring:
	config:
		activate:
			on-cloud-platform: "kubernetes"
			on-profile: "prod | staging"
myotherprop: "sometimes-set
```

## `SpringBoot`配置随机值
