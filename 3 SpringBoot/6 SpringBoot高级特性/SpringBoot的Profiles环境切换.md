# 0 `SpringBoot`的默认配置文件与特定配置文件

## 默认配置文件

> 在任何情况下我们的`SpringBoot`都会去加载两个配置文件
>
> - 一个是**`application.properties`配置文件**
> - 一个是`application.yaml`配置文件
>
> 这两个配置文件也就**是我们所说的`SpringBoot`默认配置文件**

## 特定配置文件

> - 我们的`SpringBoot`还可以根据我们当前使用的工作环境去加载我们的配置文件,这些配置文件的命名格式为`application-工作环境名.properties`或`application-工作环境名.yaml`
>
> - 而这些只有当我们的某个工作环境被加载时,才会被我们的`SpringBoot`读取加载的配置文件就被我们称之为特定配置文件.

## 默认配置与特定配置文件之间的优先级关系

> - 若我们的特定配置文件被激活了,并且其与我们的默认配置文件都修改了同一个配置属性(如`server.port`属性),那么我们的`SpringBoot`最终该属性的值,是会遵循我们的该特定配置文件所指定的值.
> - **也就是说,已激活的特定配置文件的配置优先级要高于我们的默认配置文件**

# 1 `SpringBoot`配置文件的`Profile`环境切换

> **应用场景**
>
> - 在我们的开发过程中,我们的项目大多数时候要经过开发,测试,投入生产环境,维护等等众多的状态.有时每一个状态下我们对于我们的`SpringBoot`的配置文件的配置都不同,比如生产环境下连接的数据库不同于测试环境下,我们需要在配置文件中对数据库连接进行修改等等.
> - 因此我们的`SpringBoot`开发了`Profile`环境切换功能,我们可以针对不同的环境配置不同的`.yaml`配置文件,然后借助于我们的`spring.profiles.active`配置属性来选择性激活不同的环境.

## 1.1 方法

- **构建我们不同工作环境下的`.yaml`配置文件**
    - 基本格式为`application-工作环境名称.yaml`
- **在我们的`SpringBoot`的默认`application.properties`或`application.yaml`配置文件中使用`spring.profiles.active=工作环境1名[,工作环境2名]`来选择激活我们的工作环境**
    - 当然我们也可以通过命令行来激活`java -jar xxx.jar --spring.profiles.active=工作环境1名[,工作环境2名]`
- **最后,我们的默认配置文件与我们在默认配置文件(或命令行)中设置激活的配置环境下的配置文件会被激活使用**

> **注意**:若同一个属性我们既在默认配置文件中设置了值又在我们的`Profile`工作环境配置文件中设置了值,那么最终这个值还是会以我们的`Profiles`工作环境配置文件中设置的值为准

## 1.2 示例

- ```properties
    spring.profiles.active=dev,test
    ```

- ```yaml
    spring:
    	profiles:
    		active: dev,test
    ```

- ```shell
    java -jar xxx.jar --spring.profiles.active=dev,test
    ```

****

# 2 `Profiles Group`的使用

> **应用场景**
>
> - 

## 2.1 方法

- **在我们的`SpringBoot`的默认配置文件中使用`spring.profiles.group.组名[组序号]=环境名`的方式定义我们的`Profiles`组**

<img src="https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202302252147411.png" alt="image-20230225214739332" style="zoom:67%;" />

- **在我们的`SpringBoot`的默认`application.properties`或`application.yaml`配置文件中使用`spring.profiles.active=组名`的方式激活某一个组的配置文件.**

<img src="https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202302261514618.png" alt="image-20230226151457382"  />

## 2.2 示例

- ```properties
    spring.profiles.group.mygroup[0]=dev
    spring.profiles.group.mygroup[1]=test
    ```

- ```yaml
    spring:
    	profiles:
    		group:
    			mygroup:
    				- "dev"
    				- "test"
    ```

****

# 3 `@Profile`注解的使用

> **应用场景**
>
> - 前面我们的`SpringBoot`的配置文件借助于`SpringBoot`的`Profile`切换机制实现了根据不同的工作环境配置我们的配置文件,并借助于`spring.profiles.active`进行配置文件的选择性激活.我们实现了`SpringBoot`配置文件的基于工作环境配置
> - 同样的,我们的项目中的很多组件,配置类等等我们同样也希望根据不同的工作环境来进行选择性加载.例如与测试相关的组件以及配置类在我们的生产环境中就可以不注册加载到我们的`Spring BeanFactory`中
> - 而我们的`@Profile`注解就为我们实现了这一想法,通过使用`@Profile`注解去修饰我们的`Spring`组件,配置类等等,就可以指定好这些组件在哪些工作环境下才被我们的`SpringBoot`加载到我们的`Spring BeanFactory`中

## `@Profile`注解介绍

- `value`属性
    - `String[]`
    - 只有在这个属性存储的**工作环境之一**被激活的情况下,我们的当前被`@Profile`注解修饰的组件才会被加载到我们的`Spring BeanFactory`中
    - 由于这个属性是`String[]`类型的,因此我们可以存储多个不同的工作环境

![image-20230226161005655](https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202302261610705.png)

## `@Profile`可以使用的地方

- **`@Component`+`@Profile`**
    - `@Service`,`@Controller`,`@Repository`等这类`@Component`的派生注解也可以和`@Prifile`组合使用,效果一致
    - 作用为控制我们的当前组件只有在`@Profile`注解标识的工作环境下才被激活,注册到我们的`Spring BeanFactory`中
- **`@Configuration`+`@Profile`**
    - 所有`@Configuration`的派生注解同样也是可以与`@Profile`组合使用的,并且效果一致(其实`@Configure`也是`@Component`的派生注解之一)
    - 作用为控制我们的当前配置类,只有在`@Profile`注解标识的工作环境下才生效
- **`@ConfigurationProperties`+`@Profile`**
    - 作用为控制我们的当前类,只有在`@Profile`注解标识的工作环境下才会根据我们的`@ConfigurationProperties`注解标识的配置前缀进行加载
- **`@Bean`+`@Profile`**
    - 作用为控制我们当前基于`@Bean`加载的`Bean`对象,只有在`@Profile`注解标识的工作环境下才会真正地去注册加载,如果不在,则不注册加载

> **注意**:如果 `@ConfigurationProperties` `Bean`是通过`@EnableConfigurationProperties`注册的，而**不是通过自动扫描注册的**，则需要在具有`@EnableConfigurationProperties` 注解的 `@Configuration` 类上**指定 `@Profile` 注解**。 只有在 `@ConfigurationProperties`是被自动扫描所注册的情况下，`@Profile` 可以在 `@ConfigurationProperties`类本身指定。

<img src="https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202302252144345.png" alt="image-20230225214403277" style="zoom: 67%;" />

<img src="https://raw.githubusercontent.com/tangling0112/MyPictures/master/img/202302252143753.png" alt="image-20230225214353671" style="zoom:67%;" />

# 4 重要属性介绍

## 4.1 `spring.config.activated.on-profile`属性

## 4.2 `spring.profiles.active`属性的使用

- **作用**

    - `spring.profiles.active`属性是用于设置我们的`SpringBoot`项目需要激活的工作环境的
    - 若我们**不指定这个属性的值**,那么代表着我们**没有任何工作环境需要被激活**.

- **注意事项**

    - **`spring.profiles.active`属性** **不能**在`SpringBoot`的**特定配置文件中使用
    - **`spring.profiles.active`属性**的值**可以**为`Profiles Group`名

- **示例**

    - ```properties
        spring.profiles.active=dev,test
        ```

    - ```yaml
        spring: 
        	profiles:
        		active: dev,test
        ```

## 4.3 `spring.profiles.include`属性的使用

- **作用**

    - `spring.profiles.include`属性是用于设置我们的`SpringBoot`项目的**固定激活环境**的.
    - 如果一个**环境`A`**被我们使用该属性设置为**固定激活环境**,那么即便我们**没有使用**`spring.profiles.active`属性**激活任何一个环境**,或者**激活了**一些环境**但是没有激活我们这个环境`A`**,那么我们的`SpringBoot`也**照样会激活我们的这个固定激活环境**

- **注意事项**

    - **`spring.profiles.include`属性**与`spring.profiles.active`属性一样,**不能**在`SpringBoot`的**特定配置文件**中使用
    - **`spring.profiles.include`属性**的值**可以**为`Profiles Group`名

- **示例**

    - ```properties
        spring.profiles.include[0]=dev
        spring.profiles.include[1]=test
        ```

    - ```yaml
        spring:
        	profiles:
        		include:
        			- "test"
        			- "dev"
        ```

