---
layout:     post
title:      "使用Eclipse阅读Hadoop源码"
subtitle:	"安装以及解决问题的方法"
date:       2017-07-20 23:40:00
author:     "zhouleihao"
header-img: "img/post-eclipse-hadoop-source-code.png"
header-mask: 0.3
catalog:    true
tags:
    - Hadoop
---

下半年规划对Hadoop集群做资源管理的工作，计划阅读部分代码，深入了解yarn和hdfs等相关组件的内部实现细节。对于一个习惯写C++的人来说，Hadoop源码导入Eclipse着实费了不少功夫，本文记录下导入的步骤以及期间遇到的一些问题，沉淀的同时也方便其他有需求的同学。

## 环境

**操作系统**：MacOS Sierra 10.12.5

**Eclipse**：Oxygen Release (4.7.0)

**Hadoop源码**：hadoop-2.6.0-cdh5.11.0，下载地址点击[这里](http://archive.cloudera.com/cdh5/cdh/5/hadoop-2.6.0-cdh5.11.0.tar.gz)

**编译需要的库**：

1. JDK 1.7
2. Maven 3.5
3. Protobuf 2.5
4. cmake
5. gcc

需要注意的是JDK和PB的版本，系统中JDK安装的是1.8，PB首次安装的是3.0，在后面对代码执行mvn命令的时候会报错，可以选择重新安装，或者安装多版本来解决。介绍完需要的环境，下面进度正题。

## 源码导入Eclipse步骤

### 编译代码

* 解压上面下载到的hadoop源码hadoop-2.6.0-cdh5.11.0.tar.gz到自己喜欢的位置，完成执行以下命令进入hadoop mvn目录。

  ```shell
  cd hadoop-2.6.0-cdh5.11.0/src/hadoop-maven-plugins
  ```

* 执行如下命令编译项目，等待完成。

  ```
  mvn install
  cd hadoop-2.6.0-cdh5.11.0/src
  mvn org.apache.maven.plugins:maven-eclipse-plugin:2.6:eclipse -DskipTests
  ```

  官方以及很多网友推荐的做法是执行mvn eclipse:eclipse -DskipTests，但是执行这个命令会报如下的错误：

  ```
  [ERROR] Failed to execute goal org.apache.maven.plugins:maven-eclipse-plugin:2.8  :eclipse (default-cli) on project hadoop-common: Request to merge when 'filtering' is not identical. Original=resource src/main/resources: output=target/classes, include=[], exclude=[common-version-info.properties|**/*.java], test=false, filtering=false, merging with=resource src/main/resources: output=target/classes, include=[common-version-info.properties], exclude=[**/*.java], test=false, filtering=true -> [Help 1] 
  ```

  这一步在安装过程中遇到的错误都是因为上面所需要的库版本问题导致，如果遇到问题请考虑版本问题或者认真看日志，定位问题。

### 导入Eclipse

编译完成后，打开Eclipse导入源码，有两种方式，import->Existing Projects into Workspace和import->Existing Maven Projects，不同之处是前者只导入一个项目，后者会导入几十个小的项目，我选择的是后者，为了更直观的在Package Explorer中找到想要阅读的模块。当然，这个纯看个人的喜好。

一切的前提是已经配置好了Eclipse的mvn环境，包括制定M2_REPO的路径。在import->Existing Maven Projects的对话框中选中hadoop-2.6.0-cdh5.11.0路经后，点击Finish开始导入，等到自动build完成后Project->clean，会发现左侧Package Explorer中所有Hadoop相关的项目都有一个红叉，下面Problems页面有200+的错误。

对于一个强迫症来说，实在是无法接受如此多的错误。下面的部分主要介绍遇到的问题以及解决的办法。

## 各种报错解决办法

* AvroRecord cannot be resolved to a type

  报错文件位置：/hadoop-common/src/test/java/org/apache/hadoop/io/serializer/avro/TestAvroSerialization.java

  ```
  # 下载avro-tools-1.7.7.jar
  curl ***/avro-tools-1.7.7.jar
  cd hadoop-2.6.0-cdh5.11.0/src/hadoop-common-project/hadoop-common/src/test/avro
  java -jar ~/Downloads/avro-tools-1.7.7.jar compile schema avroRecord.avsc ../java
  ```

  hadoop-common项目上右键Refresh。

* EchoRequestProto cannot be resolved

  ```
  cd hadoop-2.6.0-cdh5.11.0/src/hadoop-common-project/hadoop-common/src/test/proto
  protoc --java_out=../java *.proto
  ```

  hadoop-common项目上右键Refresh。

* maven-resources-plugin prior to 2.4 is not supported by m2e. Use maven-resources-plugin version 2.4 or later.

  错误很明显，当前使用的版本2.2过低，只要使用高版本的maven-resources-plugin就可以解决问题。打开项目hadoop-project的pom.xml文件，修改如下：

  ```
  <plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-resources-plugin</artifactId>
    <version>2.5(初始是2.2)</version>
  </plugin>
  ```

* Plugin execution not covered by lifecycle configuration: org.apache.avro:avro-maven-plugin:1.7.6-cdh5.11.0:protocol (execution: default, phase: generate-sources)

  这个错误与eclipse有关，网上很多解决方案，比如在pom.xml文件的plugins标签外层添加pluginmanagment，但是这个方法对于有几十个pom.xml文件来说是灾难性的，这里采用的是另一种方法，修改lifecycle-mapping-metadata.xml文件。

  打开Properties面板，Maven->Lifecycle-Mapping，右侧点击open workspace lifecycle mapping metadata，添加如下内容：

  ```
  <?xml version="1.0" encoding="UTF-8"?>
  <lifecycleMappingMetadata>
      <pluginExecutions>
          <pluginExecution>
              <pluginExecutionFilter>
                  <groupId>org.apache.maven.plugins</groupId>
                  <artifactId>maven-antrun-plugin</artifactId>
                  <goals>
                      <goal>run</goal>
                  </goals>
                  <versionRange>[1.7,)</versionRange>
              </pluginExecutionFilter>
              <action>
                  <ignore />
              </action>
          </pluginExecution>
          <pluginExecution>
              <pluginExecutionFilter>
                  <groupId>org.apache.maven.plugins</groupId>
                  <artifactId>maven-jar-plugin</artifactId>
                  <goals>
                      <goal>test-jar</goal>
                  </goals>
                  <versionRange>[2.3.1,)</versionRange>
              </pluginExecutionFilter>
              <action>
                  <ignore />
              </action>
          </pluginExecution>
      </pluginExecutions>
  </lifecycleMappingMetadata>
  ```

  保存后点击reload workspace lifecycle mapping metadata，然后对所有项目执行maven-update project。其他类似的报错同上解决方法，添加对应的groupId，artifactId，goals，versionRange，例如：

  ```
  # 如下错误提示信息，
  # groupid：org.apache.avro
  # artifactId：avro-maven-plugin
  # goals：schema
  # versionRange：[1.7.6-cdh5.11.0,)
  org.apache.avro:avro-maven-plugin:1.7.6-cdh5.11.0:schema (execution: generate-avro-test-sources, phase: generate-test-sources)	
  ```

* Project 'hadoop-streaming' is missing required source folder

  找到hadoop-streaming项目，右键点击Properties，弹出的面板中选择Java Build Path，在右侧选择Source，选中报错的那一个文件路径删除掉即可。

* The type package-info is already defined

  在hadoop-common的pom.xml文件中加入如下配置：

  ```
  <plugin>
    <!--eclipse do not support duplicated package-info.java, in both src and test.-->
    <artifactId>maven-compiler-plugin</artifactId>
    <executions>
      <execution>
        <id>default-testCompile</id>
        <phase>test-compile</phase>
        <configuration>
          <testExcludes>
            <exclude>**/package-info.java</exclude>
          </testExcludes>
        </configuration> 
        <goals>
        	<goal>testCompile</goal>
        </goals>
        </execution>                  
    </executions>
  </plugin>
  ```

* 还剩下两个错误其实没有真正去处理，因为出现在Test中，人为的把它屏蔽掉了：

  1. The field LOG is defined in an inherited type band an enclosing scope

     报错文件位置：/hadoop-yarn-server-resourcemanager/src/test/java/org/apache/hadoop/yarn/server/resourcemanager/TestRMEmbeddedElector.java

     ```
     try {
         callbackCalled.set(true);
         LOG.info("Callback called. Sleeping now"); // 报错
         Thread.sleep(delayMs);
         LOG.info("Sleep done"); // 报错
     } catch (InterruptedException e) {
     	e.printStackTrace();
     }
     ```

     解决办法：注释掉。。。

  2. The method thenReturn. in the type ...not applicable for the arguments (Enumeration)

     ```
     The method thenReturn(Enumeration<String>) in the type OngoingStubbing<Enumeration<String>> is not applicable for the arguments (Enumeration<Object>)
     ```

     报错文件位置：/hadoop-yarn-server-applicationhistoryservice/src/test/java/org/apache/hadoop/yarn/server/timeline/webapp/TestTimelineWebServices.java

     ```
     Enumeration<Object> names = mock(Enumeration.class);
     when(names.hasMoreElements()).thenReturn(true, true, true, false);
     when(names.nextElement()).thenReturn(
       AuthenticationFilter.AUTH_TYPE,
       PseudoAuthenticationHandler.ANONYMOUS_ALLOWED,
       DelegationTokenAuthenticationHandler.TOKEN_KIND);
     when(filterConfig.getInitParameterNames()).thenReturn(names); // names报错
     ```

     解决办法：names定义Object修改为String

## 最后

希望本文介绍的安装和问题的解决方法能够对各位有帮助。对于开源软件，不管是代码编译还是在Linux安装，总是会遇到各种各样的错误问题，如何解决掉才是根本，在解决问题的过程中合理的使用Google搜索会事半功倍。

