
# Hexo + Github Pages 搭建个人博客

### 一、什么是Hexo?
Hexo 是一个快速、简洁且高效的博客框架。Hexo 使用 [Markdown](https://zh.wikipedia.org/wiki/Markdown)（或其他渲染引擎）解析文章，在几秒内，即可利用靓丽的主题生成静态网页。
### 二、准备工作
你需要：
1、`NodeJS`环境，请参考 [NodeJs官网](https://nodejs.org/en/download/)
2、`Git`，请参考 [廖雪峰的Git教程](https://www.liaoxuefeng.com/wiki/0013739516305929606dd18361248578c67b8067c8c017b000)
3、一个[Github](https://github.com)账号

### 三、安装Hexo
检验NodeJs安装：
```bash
$ node -v
```
若安装成功会显示`NodeJs`版本号，例如：
```bash
$ v9.6.1
```
接着，在命令行中输入：
```bash
$ npm install -g hexo-cli
```
由于国外网络延迟的关系，下载速度太慢或者某些依赖安装失败，请安装`淘宝NPM镜像`，设置镜像的方法和使用规则详见：[https://npm.taobao.org](https://npm.taobao.org/)。
至此，若无意外情况，Hexo安装完毕，若有疑问请参考 [Hexo官方手册](https://hexo.io/zh-cn/docs/index.html)

### 四、Github Pages使用
1. 登录`Github`，在右上角选择`New repository`，新建一个`repository`，并且命名为`username.github.io`，注意把`username`替换成你申请账号的用户名，其余不要变。
2. 使用 `git clone https://github.com/MeetMax/meetmax.github.io.git` 命令将项目克隆到本地，这里以我自己的项目为例。
3. 进入文件夹 `cd meetmax.github.io` 
4. 新建 `index.html` 文件，在里面输入

```htmlbars
<!DOCTYPE html>
<html>
<head>
	<title>hello world</title>
</head>
<body>
hello world
</body>
</html>
```
5. 将文件提交到 `Github` 仓库，在`meetmax.github.io`文件夹运行命令：
```bash
$ git add .
$ git commit -m "hello"
$ git push origin master
```
6.在浏览器打开 `https://meetmax.github.io/`，若显示`hello world`则说明配置成功，注意将`meetmax`替换成自己的账号。

### 五、搭建Hexo博客
#### 1. 初始化博客目录
```bash
$ hexo init meetmax.github.io
$ cd meetmax.github.io
$ npm install
```
#### 2. 生成静态页面
```bash
$ hexo clean // 清空代码，初始化
$ hexo g // g表示generate，生成静态页面
```
#### 3.  运行项目
```bash
$ hexo s // s 表示 server
```
然后打开浏览器输入：`http://localhost:4000/`查看
#### 4. 发布文章
```bash
$ hexo new test
```
这时候会在`source/_post`文件夹生成`test.md`文件，或者直接手动在`source/_post`文件夹下新建`test.md`文件。再次运行命令：
```bash
$ hexo clean
$ hexo g 
$ hexo s
```
#### 5.  配置
网站的配置大部分都在**_config.yml**文件中：
- title -> 网站标题
- subtitle -> 网站副标题
- description -> 网站描述
- author -> 作者名字
- language -> 网站使用语言
#### 6. 需要注意的坑
_配置文件名字冒号后面必须加空格，例如 `title: meetmax`_，否则配置不生效。

#### 7. 更换主题
我用的是`Yilia`主题，这里就以该主题为例
1. 克隆主题到 `theme` 文件夹
```bash
$ cd themes
$ git clone https://github.com/litten/hexo-theme-yilia.git
```
2. 配置主题
修改项目文件夹下的`_config.yml`文件，配置如下：  `注意加空格，否则无效`
```
theme: hexo-theme-yilia
```
### 六、部署到Github
#### 1. 安装 hexo-deployer-git
```bash
$ npm install hexo-deployer-git --save
```
若出现下面的错误，请设置`public key`，若不懂请使用搜索引擎
```bash
Permission denied (publickey).
fatal: Could not read from remote repository.
Please make sure you have the correct access rights
and the repository exists.
```
#### 2. 在`_config.yml`中配置 Git，以我自己的为例：
这里要注意 `repo`是 `ssh`地址，非 `https`
```
deploy:
  type: git
  repo: git@github.com:MeetMax/meetmax.github.io.git
  branch: master
```
#### 3. 部署到Github
```bash
$ hexo d // d表示deloy
```
#### 4.访问
打开浏览器访问：https://meetmax.github.io/

### 参考
[手把手教你使用Hexo + Github Pages搭建个人独立博客](https://linghucong.js.org/2016/04/15/2016-04-15-hexo-github-pages-blog/)
[Hexo 博客搭建指南](https://github.com/limedroid/HexoLearning)





