title: 搭建 maven 私服--by nexus
date: 2015-09-02 09:42:02
categories: maven
tags:

- maven
- nexus

description: nexus 可以代替 maven 的中央仓库，在其中发布内部模块、代理中央仓库，并可以分组管理模块。

---

# 下载安装

## 版本介绍与下载

官网： http://www.sonatype.org/nexus/
OSS、Pro、Pro+、Lifecycle、Auditor 版本，各版本区别可参考：[版本比较](http://www.sonatype.com/nexus/try-compare-buy/compare)
基本的仓库管理（不带权限分配）已经足够使用，所以使用 OSS 版。
[下载地址](https://sonatype-download.global.ssl.fastly.net/nexus/oss/nexus-latest-bundle.zip)
注：浏览器里无法下载，应该是大陆 IP 被屏蔽了，使用离线下载工具即可。

## 安装

下载后的压缩包上传至 linux 服务器

```bash
cd /usr/local
unzip nexus-latest-bundle.zip #解压
rm nexus-latest-bundle.zip #删除
y
cd nexus-2.11.2-06/conf #准备修改端口配置信息
vi nexus.properties #这里端口改为8000；
cd ../bin/
sh nexus start
```

此时提示：

```bash
****************************************
WARNING - NOT RECOMMENDED TO RUN AS ROOT
****************************************
If you insist running as root, then set the environment variable RUN_AS_USER=root before running this script.
```

输入：

```bash
export RUN_AS_USER=root
```

此时执行就 OK 了，想永久赋予权限，可以：
vi /etc/profile 加入 export RUN_AS_USER=root
执行结果：

```bash
****************************************
WARNING - NOT RECOMMENDED TO RUN AS ROOT
****************************************
Starting Nexus OSS...
Started Nexus OSS.

```

打开地址：http://ip:8000/nexus;
点击 login，u/p：admin/admin123

# 使用

nexus 包含代理仓库、宿主仓库、仓库组等。

仓库分为 4 种类型，分别是 group（分组）、hosted（宿主）、proxy（代理）、virtual（虚拟），仓库的类型是 maven1 或者 maven2，此外仓库还有个 policy 属性（策略），表示该仓库为发布（release）版本或者快照（snapshot），最后两列是仓库的状态和路径，图标列表示仓库的健康状态。
maven1 已经很少使用，同时 virtual 也是为了将 maven2 格式化为 maven1，因此也不在关注之列。

默认分组： Public Repositories；
包含以下仓库：

- 3rd party：策略为 realease 的宿主类型仓库，用来部署无法从公共仓库获得的第三方发布版本构件。
- apache snapshots：策略为 shapshot 的代理仓库，目标是 apache maven 仓库的快照版本构件。
- central:该仓库代理 maven 中央库，策略为 release，只会下载和缓存中央仓库中的发布版本构件。
- Codehaus Snapshots：策略为 shapshot 的代理仓库，目标是 Codehaus maven 仓库的快照版本构件。
- releases:策略为 release 的宿主类型仓库，用来部署内部发布版本构件。
- snapshots：策略为 shapshot 的宿主类型仓库，用来部署内部发布的快照版本构件。

一般情况下，public Repositories 已经足够使用。

## 配置 maven

修改~/.m2/settings.xml，添加：

```xml
  <!-- 把所有的maven请求全部映射到私服 -->
  <mirrors>
    <mirror>
      <id>nexus</id>
      <mirrorOf>*</mirrorOf>
      <name>Nexus</name>
      <url>http://ip:8000/nexus/content/groups/public/</url>
    </mirror>
  </mirrors>

  <!-- 添加构件私有仓库、 插件私有仓库-->
  <profiles>
    <profile>
      <id>nexus</id>
      <repositories>
		<repository>
			<id>nexus</id>
			<name>Nexus</name>
			<url>http://ip:8000/nexus/content/groups/public/</url>
			<releases><enabled>true</enabled></releases>
			<snapshots><enabled>true</enabled></snapshots>
		</repository>
      </repositories>

      <pluginRepositories>
		<pluginRepository>
			<id>nexus</id>
			<name>Nexus</name>
			<url>http://ip:8000/nexus/content/groups/public/</url>
			<releases><enabled>true</enabled></releases>
			<snapshots><enabled>true</enabled></snapshots>
		</pluginRepository>
      </pluginRepositories>
    </profile>
  </profiles>

  <!-- 启用profile -->
  <activeProfiles>
    <activeProfile>nexus</activeProfile>
  </activeProfiles>
```

此时 maven 可以让本机所有项目都从 nexus 下载.

## 部署构件到 nexus

除了代理仓库，宿主仓库的作用是提供内部的构件或者第三方的构件以供使用。

### 使用 maven 部署构件

pom 配置：

```xml
   <distributionManagement>
		<repository>
			<id>nexus-releases</id>
			<name>nexus-releases</name>
			<url>http://0.0.0.0:8000/nexus/content/repositories/thirdparty/</url>
		</repository>
		<snapshotRepository>
			<id>nexus-snapshots</id>
			<name>nexus-snapshots</name>
			<url>http://0.0.0.0:8000/nexus/content/repositories/thirdparty/</url>
		</snapshotRepository>
   </distributionManagement>
```

nexus 仓库对于匿名用户是只读的，为了能部署构件，还需要在 settings.xml 中配置认证信息：

```xml
<servers>
	<server>
		<id>nexus-releases</id>
		<username>admin</username>
		<password>1</password>
	</server>
	<server>
		<id>nexus-snapshots</id>
		<username>admin</username>
		<password>1</password>
	</server>
</servers>
```

项目中 mvn clean deploy 后就会发布到 nexus;

### 上传第三方构件

在 3rd party 中的 artifact upload 中操作：
上传时需要确认构件坐标，如果该构件是通过 maven 创建的，在 GAV defintion 中选择 From POM，否则就选择 GAV Parameters。
定义好坐标后点击 select artifact（s） to upload 选择要上传的构件，单击 Add artifact 按钮将其加入到上传列表。一次可以上传一个主构件和多个附属构件(classifier)，最后单击 upload artifacts 按钮上传.

## 权限

nexus 基于用户、角色、权限、仓库过滤来实现权限控制。
用户可以有多个角色，每个角色拥有多个权限，角色还有上下级包容关系。
nexus 内置了许多权限，例如 UI-basic ui privileges（访问界面）、UI-Repository browser（浏览仓库界面）、UI-Search（访问快速搜索栏以及搜索页面）、repo-all repositors（read）（读取所有仓库的权限）、repo-all repositors（full control）（完全控制所有仓库的权限）等。
还可以自定权限（增加仓库过滤，即 repositor target，利用正则表达式控制运行访问的特定路径；并指定运行访问的仓库）；

在 security 中可以设置 ldap（目录服务）、权限（privileges）、角色（roles）、用户（users）。

另外，在每个仓库的 configuration 中可以设置是否公开 url、是否允许上传之类的设置。

## 调度任务

在 administration-->scheduled task 中可以设置调度任务，调度任务可以完成以下任务：

1. downliad indexes：为代理仓库下载索引
2. empty trash：清空 nexus 回收站
3. backup npm metadata database : 备份 npm 数据库
4. download nuget feed：更新 nuget feed
5. evict unused proxied items from repository caches :删除代理仓库中长期没用的构件缓存
6. expire repository caches：清空代理仓库信息缓存
7. optimize repository index：优化仓库索引
8. publish index：发布仓库索引为 m2eclipse 等其他格式
9. Purge Broken Proxied Rubygems Metadata：清空 Rubygems 中失效的数据
10. Purge Nexus Timeline：删除时间线文件（该文件用于生成 rss 源）
11. Purge Orphaned API Keys：清除不再引用的 api keys
12. Rebuild Hosted Rubygems Metadata Files：重建仓库元数据文件 maven-meta-data.xml，同时创建每个文件的 md5 和 sha1.
13. Rebuild NuGet Feed：重建 nuget feed
14. Rebuild hosted npm metadata ：重建 npm 元数据
