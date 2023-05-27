# npm-link
> 用了链接链接一个包

## 简介
```
npm link [<package-spec>]
alias: ln
```
## 使用
链接一个包主要有两个步骤

1.在包文件夹下执行`npm link`（不带参数），将会在全局文件夹中`{prefix}/lib/node_modules/<package>`创建一个链接到该包的软链接（`symlink 也叫符号链接`）；如何该包bins文件中有执行命令，也会被链接到`{prefix}/bin/{name}`下。
***注意：npm link 创建的是全局软链接，所有包文件夹下都可以使用***

2.在其包他文件夹中，执行`npm link package-name`（package-name来源于package.json中的name，非文件名），将从全局安装的package-name创建一个到当前文件`node_modules/`下的软链接。

例如：
```
cd ~/projects/node-redis    # go into the package directory
npm link                    # creates global link
cd ~/projects/node-bloggy   # go into some other package directory.
npm link redis              # link-install the package
```
> 任何`~/projects/node-redis`中的变化都会自动同步到`~/projects/node-bloggy/node_modules/node-redis/`

我们也可以将上述两个步骤精简为一个，例如：
```
cd ~/projects/node-bloggy  # go into the dir of your main project
npm link ../node-redis     # link the dir of your dependency
```
***注意：这里使用的是文件夹的名字node-redis,而不是package.json中的name***

这相当于执行以下命令：
```
(cd ../node-redis; npm link)
npm link redis
```

## 注意事项
以这种方式链接的包依赖关系默认不会被保存到package.json中，因为我们的目的是让一个链接代替普通的非链接依赖关系。否则，例如，如果你依赖`redis@^3.0.1`，并运行`npm link redis`，它将用`file:.../path/to/node-redis`替换`redis@^3.0.1`的依赖，这可能并不是我们希望看到的。

如果希望将一个新的依赖当做链接添加，请使用`npm install <dep> --package-lock-only`将其添加到相关元数据中。

如果你想在package.json和package-lock.json文件中保存文件，你可以使用`npm link <dep> --save`来实现。

## 缺陷
`npm link`进行调试的时候有一些缺陷，所有npm提供了更好的调试方案，那就是workspaces。
[npm link的缺陷](https://zhuanlan.zhihu.com/p/507651534?utm_id=0)

# Workspaces

## 描述
**Workspaces**是一个通用术语，指的是npm cli中的一组功能，它支持从你的本地文件系统中的一个单一的顶级根包中管理多个包。

这一系列的功能使得处理来自本地文件系统的链接包的工作流程更加精简。执行`npm install`时自动链接，避免使用`npm link`来手动添加需要链接到当前工作文件夹node_modules目录下的包的引用。

这些在`npm install`过程中被自动链接的包称为一个**workspace**，这意味着它是当前本地文件系统中的一个嵌套包，可以在package.json中的workspaces字段中明确配置。

# 定义workspaces

**Workspace**通常通过package.json中的workspaces属性进行定义，例如
```json
{
  "name": "my-workspaces-powered-project",
  "workspaces": [
    "packages/a"
  ]
}
```
在当前工作目录下（根目录）创建`packages/a`文件夹，在该文件夹下执行`npm init -y`创建package.json文件
```
.
+-- package.json
`-- packages
   +-- a
   |   `-- package.json
```
在根目录下执行`npm install`，`packages/a`文件夹将被符号链接到当前工作目录的`node_modules`文件夹。
```
.
+-- node_modules
|  `-- a -> ../packages/a
+-- package-lock.json
+-- package.json
`-- packages
   +-- a
   |   `-- package.json
```

我们可以通过一个命令完成上面两个步骤，在当前工作目录（根目录，已通过`npm init`创建package.json）执行以下命令创建一个**workspace**:
```
npm init -w ./packages/a -y
```
>**上述命令做了以下几个事情：**
>1.创建对应的文件夹
>2.执行npm init安装package.json
>3.执行npm install将packages/a链接到根目录的node_modules下
>4.配置根目录package.json中的workspaces字段

或者执行以下命令可以同时创建多个**workspace**
```
npm init -w ./packages/a -y -w ./packages/b -y
```

## 在workspace中添加依赖
执行以下命令可以单独名称为a的workspace安装依赖
```
npm install lodash -w a
```
卸载依赖
```
npm uninstall lodash -w a
```

## 使用workspaces
在根目录添加`./lib/index.js`，在a文件夹下添加index.文件
```
// ./packages/a/index.js
module.exports = 'a'

// ./lib/index.js
const moduleA = require('a')
console.log(moduleA) // -> a
```

执行以下命令
```
node lib/index.js
```

## 在workspaces的上下文中执行命令行
我们可以使用workspace选项在某个workspace的的上下文中执行命令行，另外，如果当前工作目录是一个workspace，workspace选项会被自动设置，prefix会设置为root workspace。

在名称为a的workspace上下文中执行命令行
```
npm run test --workspace = a
```
这相当于执行如下命令
```
cd packages/a && npm run test
```
这将会执行在`./packages/a/package.json`中定义的test脚本

我们也可以同时执行多个workspace中你的脚本
```
npm run test --workspace=a --workspace=b
```
这相当于执行
```
npm run test --workspaces
```
这将会执行`./packages/a`和`./packages/b`中的test脚本，命令的执行顺序根据package.json中的workspaces字段的配置，a中脚本执行先于b中脚本
```json
{
  "workspaces": [ "packages/a", "packages/b" ]
}
```

多个workspace中，并不是所有的workspace中都在package.json中实现了scirpt脚本，如果某个workspace中为实现script脚本，可以通过下面的命令跳过执行
```
npm run test --workspaces --if-present
```