
&emsp;&emsp;APP开发过程中，往往需要在多个网络环境或配置中进行切换，以获取不同配置的APP，甚至有时需要用一套代码经过简单的配置生成不同的APP。而手动配置费时费力，且容易出错。这里介绍用脚本工具，去生成不同配置的APP工程。

### 1. 需求分析  

在配置工程中我们需要事先知道有哪些配置项：
  
 1. APP 启动图、图标等资源文件。  
 2. APP 名称、版本号、bundleID。  
 3. APP 相关的微信 APPKey、scheme。  
 4. 网络环境、部分代码替换。 
 5. provisioning profile 修改

分析以上需要配置的项，我们可以发现这些配置项可以分为**三类**，分别是：

1. 资源文件替换
2. plist 字段修改
3. 部分代码替换

接下来介绍如何实现满足以上需求的Shell脚本。

### 2. 脚本设计&实现

#### 2.1 创建脚本

&emsp;&emsp;首先创建一个shell脚本文件，在命令行工具中输入`touch xxx.sh`就可以创建shell脚本文件,在这里将shell脚本命名为：`projectDeploy.sh`。  
&emsp;&emsp;运行脚本时在命令行工具中切换到脚本文件所在的路径，输入:`sh projectDeploy.sh`命令就可以运行脚本了。
#### 2.2 脚本传参
我们需要传入参数告诉脚需要的具体配置,shell 脚本传入参数的方式有多种，这里只介绍一种：

```
while getopts ":e:s:" opt

do
    case $opt in
        e)
            environment=$OPTARG;;

        s)
            supplier=$OPTARG;;

        ?)
            echo "请输入正确的参数"
            exit 1;;
    esac
done

```

这里定义了两个全局变量 `environment` 和 `supplier`，当调用脚本时，输入命令 `sh projectDeploy.sh -e prd -s xxx`  
这种传参的好处是，指定了具体变量对应的值，不必保持参数的顺序。

#### 2.3 资源替换

&emsp;&emsp;资源替换这里使用了shell脚本里的 `copy` 命令：
`cp source_file_directory destination_directory`。  
将要替换的资源文件复制到指定位置，有时需要事先将路径下的文件清理一下再进行copy 操作：
`rm -rf destination_directory`

&emsp;&emsp;如果有同名文件存在，copy操作会覆盖原文件。  
&emsp;&emsp;这一操作可以用来替换启动图、桌面图标、图片资源、SDK、代码文件等文件类替换需求。

#### 2.4 plist 字段修改

`APP桌面名称`、`版本号`、`bundleID`、`scheme`等等plist 内的值需要替换，  
这里我们使用 MAC 自带的plist 文件编辑工具 `PlistBuddy` 来实现plist文件的维护，
这个工具的访问路径为：`/usr/libexec/PlistBuddy`

具体命令为：

```
/usr/libexec/PlistBuddy -c "set CFBundleIdentifier com.xxx.xxx" ${project_infoplist_path}  
/usr/libexec/PlistBuddy -c "set CFBundleShortVersionString ${appVersion}" ${project_infoplist_path}
/usr/libexec/PlistBuddy -c "set CFBundleDisplayName xxx" ${project_infoplist_path}
```

*scheme 值修改*

&emsp;&emsp;因为 scheme 的数据结构为 数组元素可能是字典，字典key对应的value可能是数组，因此需要事先知道要修改的值的数据结构和位置，在使用 plist 工具时指定字典对应的key，数组对应的下标：

```
/usr/libexec/PlistBuddy -c "set CFBundleURLTypes:0:CFBundleURLSchemes:0 xxx.xxx.xxx" ${project_infoplist_path}
```

&emsp;&emsp;以上命令的意思是取 `CFBundleURLTypes` 数组下的第一个元素 X，再取 X 下 `CFBundleURLSchemes` 数组下的第一个元素 Y，并更新Y的值为`xxx.xxx.xxx`。  

&emsp;&emsp;至于 provisioning profile 的替换，只需要在打包脚本里去指定就可以了，至于`project.pbxproj`文件的一些配置，也可以用 2.5 的方法去配置，见下一部分。

#### 2.5 部分代码替换

&emsp;&emsp;假如需要修改代码里的部分字符，或批量修改某个变量名称，这里就用到了 `sed` 命令.
这里讲一个 `sed `命令的用法，如果有其他特殊需求，可查询sed 命令的其他用法：

```
sed -i '' 's/^\#define XXX_XX_URL.*$/\#define XXX_XX_URL  @\"www.junziboxue.com\"/g' ${configureFile}
```

&emsp;&emsp;这个命令的意思是，匹配所有 `#define XXX_XX_URL` 宏，然后替换为 `#define XXX_XX_URL  @"www.junziboxue.com"`，这里是整行替换。shell 命令里需要对一些特殊字符进行转义，在这里需要注意。

*`^\#define XXX_XX_URL.*$`* 是一个正则表达式，表示所有 以 `#define XXX_XX_URL` 开头的字符，`/g`表示全局替换。
具体 sed 的其他用法可自行查询。

### 3 总结

&emsp;&emsp;在掌握以上方法后，我们基本可以任意配置一份代码工程，修改其一些APP的基本配置。
假如要配置的项很多，可能导致脚本非常臃肿，面对这样的情况，我们需要设计一下脚本的结构：  
&emsp;&emsp;我们可以把要配置的项剥离出来，然后设计一个函数执行具体配置，然后在函数内传入参数去指定具体的配置。
或者将配置项事先写好，然后让函数取读就可以了。
