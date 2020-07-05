---
title: maven
permalink: maven
date: 2018-03-11
tags: maven
categories: util
---
> 此资料从官网摘取
<!--more-->
## 模板
```
<project xmlns="http://maven.apache.org/POM/4.0.0"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
                      http://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>
 
  <!-- The Basics -->
  <groupId>...</groupId>
  <artifactId>...</artifactId>
  <version>...</version>
  <packaging>...</packaging>
  <dependencies>...</dependencies>
  <parent>...</parent>
  <dependencyManagement>...</dependencyManagement>
  <modules>...</modules>
  <properties>...</properties>
 
  <!-- Build Settings -->
  <build>...</build>
  <reporting>...</reporting>
 
  <!-- More Project Information -->
  <name>...</name>
  <description>...</description>
  <url>...</url>
  <inceptionYear>...</inceptionYear>
  <licenses>...</licenses>
  <organization>...</organization>
  <developers>...</developers>
  <contributors>...</contributors>
 
  <!-- Environment Settings -->
  <issueManagement>...</issueManagement>
  <ciManagement>...</ciManagement>
  <mailingLists>...</mailingLists>
  <scm>...</scm>
  <prerequisites>...</prerequisites>
  <repositories>...</repositories>
  <pluginRepositories>...</pluginRepositories>
  <distributionManagement>...</distributionManagement>
  <profiles>...</profiles>
</project>
```
<span data-type="color" style="color:rgb(0, 0, 0)"> </span>

### 基本配置


<div class="bi-table">
  <table>
    <colgroup>
      <col width="114px" />
      <col width="636px" />
    </colgroup>
    <tbody>
      <tr>
        <td rowspan="1" colSpan="1">
          <div data-type="p">modelVersion</div>
        </td>
        <td rowspan="1" colSpan="1">
          <div data-type="p">Maven模块版本，目前我们一般都取值4.0.0</div>
        </td>
      </tr>
      <tr>
        <td rowspan="1" colSpan="1">
          <div data-type="p">groupId</div>
        </td>
        <td rowspan="1" colSpan="1">
          <div data-type="p">分组名称,一般域名倒叙。</div>
        </td>
      </tr>
      <tr>
        <td rowspan="1" colSpan="1">
          <div data-type="p">artifactId</div>
        </td>
        <td rowspan="1" colSpan="1">
          <div data-type="p">项目名称或子模块名称。</div>
        </td>
      </tr>
      <tr height="34px">
        <td rowspan="1" colSpan="1">
          <div data-type="p">version</div>
        </td>
        <td rowspan="1" colSpan="1">
          <div data-type="p">版本号</div>
          <ul data-type="unordered-list">
            <li data-type="list-item" data-list-type="unordered-list">
              <div data-type="p"><strong>*-SNAPSHOT </strong>不稳定版本</div>
              <ul data-type="unordered-list">
                <li data-type="list-item" data-list-type="unordered-list">
                  <div data-type="p">存放在快照仓库,构建时远程抓取</div>
                </li>
              </ul>
            </li>
            <li data-type="list-item" data-list-type="unordered-list">
              <div data-type="p"><strong>*-RELEASE</strong> 稳定版本</div>
              <ul data-type="unordered-list">
                <li data-type="list-item" data-list-type="unordered-list">
                  <div data-type="p">存放在正式仓库,构建的时会先在本次仓库中查找是否已经有了这个依赖库，如果没有的话才会去远程仓库中去拉取</div>
                </li>
              </ul>
            </li>
          </ul>
        </td>
      </tr>
      <tr height="34px">
        <td rowspan="1" colSpan="1">
          <div data-type="p">name</div>
        </td>
        <td rowspan="1" colSpan="1">
          <div data-type="p">项目名称</div>
        </td>
      </tr>
      <tr height="34px">
        <td rowspan="1" colSpan="1">
          <div data-type="p">description</div>
        </td>
        <td rowspan="1" colSpan="1">
          <div data-type="p">项目描述</div>
        </td>
      </tr>
      <tr>
        <td rowspan="1" colSpan="1">
          <div data-type="p">packaging</div>
        </td>
        <td rowspan="1" colSpan="1">
          <div data-type="p">打包类型，可取值：jar,war,maven-plugin等等，这个配置用于package的phase，具体可以参见package运行的时候启动的plugin，</div>
        </td>
      </tr>
    </tbody>
  </table>
</div>

#### parent
继承父类
```xml
    <parent>
        <groupId>com.yuyu</groupId>
        <artifactId>yuyu-nexus-parent-pom</artifactId>
        <version>0.0.2.RELEASE</version>
```
#### modules
声明子模块
```xml
	<modules>
		<module>sales-mall</module>
		<module>sales-middle</module>
	</modules>

```
#### Dependencies
项目所需要的依赖包

<div class="bi-table">
  <table>
    <colgroup>
      <col width="113px" />
      <col width="637px" />
    </colgroup>
    <tbody>
      <tr>
        <td rowspan="1" colSpan="1">
          <div data-type="p">groupId</div>
        </td>
        <td rowspan="1" colSpan="1">
          <div data-type="p">依赖项的groupId</div>
        </td>
      </tr>
      <tr>
        <td rowspan="1" colSpan="1">
          <div data-type="p">artifactId</div>
        </td>
        <td rowspan="1" colSpan="1">
          <div data-type="p">依赖项的artifactId</div>
        </td>
      </tr>
      <tr>
        <td rowspan="1" colSpan="1">
          <div data-type="p">version</div>
        </td>
        <td rowspan="1" colSpan="1">
          <div data-type="p">依赖项的版本</div>
        </td>
      </tr>
      <tr>
        <td rowspan="1" colSpan="1">
          <div data-type="p">scope</div>
        </td>
        <td rowspan="1" colSpan="1">
          <div data-type="p">依赖项的适用范围：</div>
          <ul data-type="unordered-list">
            <li data-type="list-item" data-list-type="unordered-list">
              <div data-type="p">compile，缺省值，适用于所有阶段，会随着项目一起发布。</div>
            </li>
            <li data-type="list-item" data-list-type="unordered-list">
              <div data-type="p">provided，类似compile，期望JDK、容器或使用者会提供这个依赖。如servlet.jar。</div>
            </li>
            <li data-type="list-item" data-list-type="unordered-list">
              <div data-type="p">runtime，只在运行时使用，如JDBC驱动，适用运行和测试阶段。</div>
            </li>
            <li data-type="list-item" data-list-type="unordered-list">
              <div data-type="p">test，只在测试时使用，用于编译和运行测试代码。不会随项目发布。</div>
            </li>
            <li data-type="list-item" data-list-type="unordered-list">
              <div data-type="p">system，类似provided，需要显式提供包含依赖的jar，Maven不会在Repository中查找它。之前例子里的junit就只用在了test中。</div>
            </li>
          </ul>
        </td>
      </tr>
      <tr>
        <td rowspan="1" colSpan="1">
          <div data-type="p">exclusions</div>
        </td>
        <td rowspan="1" colSpan="1">
          <div data-type="p">排除项目中的依赖冲突时使用。</div>
        </td>
      </tr>
    </tbody>
  </table>
</div>

##### exclusions
1. 排除告诉Maven不要包括指定的项目依赖的这种依赖性(换句话说,它的传递依赖)。例如,maven-embedder需要maven-core,我们不希望使用它或其依赖项,然后我们将它排除。
```xml
 <dependencies>
    <dependency>
      <groupId>org.apache.maven</groupId>
      <artifactId>maven-embedder</artifactId>
      <version>2.0</version>
      <exclusions>
        <exclusion>
          <groupId>org.apache.maven</groupId>
          <artifactId>maven-core</artifactId>
        </exclusion>
      </exclusions>
    </dependency>
    ...
  </dependencies>
```
2. 它有时也有用片段依赖的传递依赖关系。依赖可能错误地指定的范围,与其他依赖或依赖项冲突在您的项目。使用通配符不便于排除所有依赖的传递依赖项。在下面的情况下你可能会使用maven-embedder和你想管理依赖你自己使用,所以你剪辑传递依赖关系:
```xml
    <dependency>
      <groupId>org.apache.maven</groupId>
      <artifactId>maven-embedder</artifactId>
      <version>3.1.0</version>
      <exclusions>
        <exclusion>
          <groupId>*</groupId>
          <artifactId>*</artifactId>
        </exclusion>
      </exclusions>
    </dependency>
```
#### build
构建
```xml
<build>
    <finalName>sales-middle-service</finalName>
      <resources>
        <resource>
            <directory>${project.basedir}/src/main/resources</directory>
             <includes>
                <include>**/*.properties</include>
                <include>**/*.xml</include>
            </includes>
            <filtering>false</filtering>
        </resource>
        <resource>
            <directory>${project.basedir}/src/main/java</directory>
            <includes>
                <include>**/*.properties</include>
                <include>**/*.xml</include>
            </includes>
            <filtering>false</filtering>
        </resource>
    </resources>
</build>
```

| finalName | 项目打包后的名字 |
| :--- | :--- |
| resources | 配置文件 |
| filter | 可以把属性写到文件里,用属性文件里定义的属性做替换 |