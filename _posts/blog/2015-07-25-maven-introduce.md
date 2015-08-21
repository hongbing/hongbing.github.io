---
layout: post
title: maven实践 
description: maven相关概念与实践
category: blog
---

### 一. 相关概念

#### 1.版本号
版本号记录一个项目在某个阶段的某些功能的实现，实现项目中不同功能的隔离与区分。

版本号格式一般分为：

>      主版本号.分支版本号.小版本号-里程碑版本号

**主版本号**：一般指框架具有重大变化的版本号，比如struts1，struts2框架的变化

**分支版本号**： 一般指功能的变化

**小版本号**：bug的修复

**里程碑版本号**：`SNAPSHOT`-->`RELEASAE`-->`FINAL`

#### 2.仓库分类

狭义上讲，仓库分为两种，一种是本地仓库，一种是远程仓库。

广义上讲，远程仓库也可分为私有仓库和中心仓库，也即是分为三种：本地仓库，私有仓库和中心仓库

##### 2.1. 本地仓库

即为本地的./.m2/repository文件夹，项目中需要的jar包都会被保存在本地仓库中，项目的更新操作（commit操作）都会首先同步到本地仓库中。

##### 2.2. 私有仓库

局域网内部的仓库，当本地仓库没有项目中所需要的jar包时，从私有仓库中找，如果存在，就将其下载到本地仓库中。同时，针对多人协作开发，可以将不同的开发版本deploy到私有仓库中。
私有仓库的作用为：提升查找jar包的速度，进行项目版本管理。

（注：使用commit and push命令只会将本地的代码提交到仓库中，使用deploy才会在仓库中为此项目打一个jar或者war包。）

私有仓库中的仓库类型也分为几类：

>**RELEASE**：项目发布版本的存放地

> **SNAPSHOT**：项目开发版本的存放地

> **3rd party**：第三方项目，后者是第三方jar包的存放地，这个里面的内容一般由内部人员添加.

> **group**：将仓库中的多个内容添加到一起组成一个group，这样在项目的pom.xml文件中指定所需要的jar包时就可以通过group的方式，而不必每一个jar包都下一个dependency。

![私有仓库示例](/images/maven/repository.png)


##### 2.3. 中心仓库
maven远程服务器管理的仓库，地址是[maven中心仓库](http://repo1.maven.org.maven2)

#### 3. 项目的发布

在项目中添加  

    <distributeManagement>
        <snapshotRepository>
        <id>firehose.0.0.1-snapshot</id>
        <name>fisehose SNAPSHOT</name>
        <url>10.30.XXX.XXX/nexus/content/Repositories/snapshots/</url>
        </snapshotRepository>
    </distributeManagement>

使用`# mvn deploy`命令即可将项目发布到私有仓库中，也即是上面的url中

#### 4. maven的生命周期

maven的3种独立的生命周期

+ **clean**
 
pre-clean 准备清理工作

clean 清理     

post-clean 清理后需要做的工作

+ **compile**

validate
 
 generate-sources

 process-sources

generate-resources

process-resources     复制并处理资源文件，至目标目录，准备打包。

compile     编译项目的源代码。

process-classes

generate-test-sources 

process-test-sources

generate-test-resources

process-test-resources     复制并处理资源文件，至目标测试目录。

test-compile     编译测试源代码。

process-test-classes

test     使用合适的单元测试框架运行测试。这些测试代码不会被打包或部署。

prepare-package

package     接受编译好的代码，打包成可发布的格式，如 JAR 。

pre-integration-test

integration-test

post-integration-test

verify

install     将包安装至本地仓库，以让其它项目依赖。

deploy     将最终的包复制到远程的仓库，以让其它开发人员与项目共享。

+ **site**

pre-site     执行一些需要在生成站点文档之前完成的工作

site    生成项目的站点文档

post-site     执行一些需要在生成站点文档之后完成的工作，并且为部署做准备

site-deploy     将生成的站点文档部署到特定的服务器上

**clean**、**compile**和**site**这三个生命周期是不相关的。执行每一个生命周期，其前面的所有阶段都会被执行。生命周期中的每一个操作都是以插件机制完成的。


### 二. maven实践

#### 1. 手动添加jar包到本地仓库
执行命令：

>	# mvn install:install-file -Dfile=jar包的位置 -DgroupId=jar的groupId -DartifactId=jar的artifactId -Dversion=jar的version -Dpackaging=jar`

添加到maven库中后，使用该jar包时需要修改maven工程的pom.xml文件，将添加的jar包加入到pom.xml文件中即可使用。
 
#### 2. maven创建web-app
使用maven创建web-app，需要自行建文件夹src/main/java包。

支持web-app发布到tomcat的插件：cargo

支持web-app发布到jetty的插件：jetty-maven-plugin

#### 3. maven查找包依赖

##### 3.1 传递依赖的版本冲突

+ 使用maven project report插件来显示所有的项目依赖关系

在项目pom.xml的<project></project>里添加 maven插件maven-project-info-reports-plugin：

    <reporting>
      <plugins>
       <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>
         maven-project-info-reports-plugin
        </artifactId>
         <version>2.4</version>
       </plugin>
     </plugins>
     </reporting>

使用这个插件，然后执行：

`mvn  project-info-reports:dependencies`

就可以在target/site/dependencies.html里查看依赖报表，通过这个报表就能够找到冲突的版本，然后再使用exclusions来排除相关的包。

    <dependency>
        <groupId>com.x.y</groupId>
        <artifactId>xyz</artifactId>
        <version>1.1.1</version>
        <exclusions>
                <exclusion>
                        <groupId>com.u.v</groupId>
                        <artifactId>uvw</artifactId>
                        <version>0.9.1</version>
                </exclusion>
        </exclusions>
    </dependency>
    
另外使用命令

`mvn dependency:analyze`

可以分析出显示声明但没有依赖的包，可以将其去除。

+ 使用脚本显示循环依赖包

        #!/bin/bash
        ### find cycle in maven depnedency tree
        
        if [ $# -gt 0 ];then
            sourcepath=$1
        else
            sourcepath=`pwd`
        fi
        if [ ! -f "$sourcepath/pom.xml" ]; then
            echo "$sourcepath is not a vaild maven project!"
            echo 'Usage : ./findcycle [path]'
            exit 1;
        fi
        mvn=`which mvn`
        if [ "$mvn" = "" ];then
            echo "counld not found mvn in PATH,exit!"
            exit 1;
        fi
        
        cd $sourcepath
        echo "scan cycle dependency in $sourcepath ..."
        mvn dependency:tree -Dverbose | awk -F'- ' '{if(index($2,"maven-dependency-plugin")>0){indent=0;}else{indent=length($1);}stack[indent]=$2;if(index($0,"for cycle")>0){print "****found cycle****";for(i=0;i<=indent;i++){if(stack[i]!=null){print "->"stack[i]}}}}'
        echo "scan finished!"


使用上面的脚本（将其放置于工程目录下，或者指定工程路径）检测是否存在循环依赖，如果存在，会输出循环依赖的链。（此脚本取自于@秦迪 Axb的自我修养）

![jewel](/images/maven/jewel.jpg)
