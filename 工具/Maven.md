### dependencyManagement

在项目的最顶层的父POM中使用，使所有**子项目**中引用一个依赖而**不用显式的列出版本号**。

Maven会沿着父子层次向上走，直到找到一个拥有dependencyManagement元素的项目，然后就使用此元素中指定的版本号。

好处：

- 在顶层父容器更新，所有子项目一一修改；

- 子项目需要另外一个版本，只需要声明version即可

注意：

- dependencyManagement处仅仅声明依赖，**不实现引入**，子项目需要**显式声明**需要用的依赖



## Maven命令

### compile

compile 是 maven 工程的编译命令，作用是将 src/main/java 下的文件编译为 class 文件输出到 target 目录下

### test

test 是 maven 工程的测试命令 mvn test，会执行 src/test/java 下的单元测试类

### clean

clean 是 maven 工程的清理命令，执行 clean 会删除 target 目录及内容

### package

package 是 maven 工程的打包命令，对于 java 工程执行 package 打成 jar 包，对于 web 工程打成 war包

### install

install 是 maven 工程的安装命令，执行 install 将 maven 打成 jar 包或 war 包发布到本地仓库

### deploy

deploy 同时把打好的可执行jar包或者war包发布到本地仓库和远程私服仓库

## Maven坐标定义

在 pom.xml 中定义坐标，内容包括：groupId、artifactId、version

```pom
<!--项目名称，定义为组织名+项目名，类似包名-->
<groupId>me.sc</groupId>
<!-- 模块名称 -->
<artifactId>online-edu</artifactId>
<!-- 当前项目版本号，snapshot 为快照版本即非正式版本，release 为正式发布版本 -->
<version>1.0-SNAPSHOT</version>
```

