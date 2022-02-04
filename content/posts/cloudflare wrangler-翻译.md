---
title: "Cloudflare Wrangler 翻译"
date: 2022-01-28T16:37:40+08:00
draft: false
categories: 翻译
---
# cloudflare wrangler

[cloudflare github地址](https://github.com/cloudflare/wrangler "cloudflare github地址")

[翻译原文地址](https://goldenrod-ray-a21.notion.site/cloudflare-wrangler-46b76393e22a4f75af26fc4e5d8f8cdf "翻译原文地址")

# 🤠 wrangler

`wrangler` is a CLI tool designed for folks who are interested in using [Cloudflare Workers](https://workers.cloudflare.com/).

`*wrangler` 是一个 CLI 工具，为那些对使用 Cloudflare Workers 感兴趣的人设计。*

Wrangler Demo (*Wrangler 样例*)

## Installation (*安装*)

You have many options to install wrangler!

*你有很多种选择去安装wrangler*

### Install with `npm` (*通过`npm`安装*)

We strongly recommend ， install `npm` with a Node version manager like `[nvm](https://github.com/nvm-sh/nvm#installing-and-updating)`, which puts the global `node_modules` in your home directory to eliminate permissions issues with `npm install -g`. 

*我们强烈建议你用像`nvm`这样的Node版本管理器来安装`npm`，它将全局`node_modules`放在你的主目录下，以消除`npm install -g`的权限问题。*

Distribution-packaged `npm` installs often use `/usr/lib/node_modules` (which is root) for globally installed `npm` packages, and running `npm install -g` as `root` prevents `wrangler` from installing properly.

*分发包的npm安装通常使用/usr/lib/node_modules（也就是root）来安装全局的npm包，以root身份运行npm install -g会阻止wrangler的正常安装。*

Once you’ve installed `nvm` and configured your system to use the `nvm` managed node install, run:

*一旦你已经安装好`nvm` 并配置了系统以使用nvm管理node安装，运行*

```
npm i @cloudflare/wrangler -g
```

If you are running an ARM based system (eg Raspberry Pi, Pinebook) you’ll need to use the `cargo` installation method listed below to build wrangler from source.

*如果你运行的是一个基于ARM的系统(eg Raspberry Pi, pinebook)*

*你将需要用下面列出的`cargo`安装方法从源码构建wrangler。*

### Specify binary location (*指定二进制位置*)

In case you need `wrangler`’s npm installer to place the binary in a non-default location (such as when using `wrangler` in CI), you can use the following configuration options to specify an install location:

*如果你需要`wrangler`的npm安装程序将二进制文件放一个非默认位置(例如当使用`wrangler`在CI)，你可以使用用以下配置选项去指定一个安装位置*

- Environment variable: `WRANGLER_INSTALL_PATH`

       *环境变量：*`WRANGLER_INSTALL_PATH` 

- NPM configuration: `wrangler_install_path`

       *NPM配置：*`wrangler_install_path` 

### Specify binary site URL （*指定二进制站点url*）

In case you need to store/mirror binaries on premise you will need to specify where wrangler should search for them by providing any of the following:

*如果在你需要存储/镜像二进制文件前提下，你将需要通过提供以下任何一项来指定wrangler应该在哪里搜索它们。*

- Environment variable: `WRANGLER_BINARY_HOST`
    
    *环境变量：*`WRANGLER_BINARY_HOST` 
    
- NPM configuration: `wrangler_binary_host`
    
    *NPM配置:*  `wrangler_binary_host` 
    

### Install with `cargo` (*通过`cargo`安装*)

```
cargo install wrangler
```

If you don’t have `cargo` or `npm` installed, you will need to follow these [additional instructions](https://developers.cloudflare.com/workers/tooling/wrangler/install/).

*如果你没有安装`cargo` or `npm`，你需要遵循下面这些额外说明*

### Install on Windows (*在windows上安装*)

[perl is an external dependency of crate openssl-sys](https://github.com/sfackler/rust-openssl/blob/b027f1603189919d5f63c6aff483243aaa188568/openssl/src/lib.rs#L11-L15). If installing wrangler with cargo, you will need to have perl installed. We’ve tested with [Strawberry Perl](https://www.perl.org/get.html). If you instead install perl via scoop, you may need to also run `scoop install openssl` in order to get the necessary openssl dlls. Installing wrangler with `npm` instead of cargo is another option if you don’t want to install perl.

*perl 是crate openssl-sys的一个外部依赖项。如果你通过cargo安装wrangler，你将需要去安装perl。我们已经用Strawberry Perl进行了测试。如果你转而通过`scoop install perl`，你可能需要同时运行scoop install openssl，以便获得必要的openssl dlls。如果你不想安装perl，用`npm`而不是cargo安装wrangler是另一个选择。*

## Updating (*更新*)

For information regarding updating Wrangler, click [here](https://developers.cloudflare.com/workers/cli-wrangler/install-update#update).

*有关更新Wrangler的信息，请点击这里。*

## Getting Started (*入门*)

Once you have installed Wrangler, spinning up and deploying your first Worker is easy!

*一旦你安装了Wrangler，启动和部署你的第一个Worker是很容易的!*

```
$ wrangler generate my-worker
$ cd my-worker
# update your wrangler.toml with your Cloudflare Account ID
$ wrangler config
$ wrangler publish
```

## 🎙️ Top Level Commands （*最高级别的命令*）

### 👯 `generate` （*生成*）

Scaffold a project, including boilerplate code for a Rust library and a Cloudflare Worker.

*构建一个项目的支架，包括Rust库和Cloudflare Worker的模板代码。*

```
wrangler generate <name> <template> --type=["webpack", "javascript", "rust"]
```

All of the arguments and flags to this command are optional:

*这个命令的所有参数和标签是可选的*

- `name`: defaults to `worker`
- `template`: defaults to the `[https://github.com/cloudflare/worker-template](https://github.com/cloudflare/worker-template)`
- `type`: defaults to `javascript` based on the [“worker-template”](https://github.com/cloudflare/worker-template/blob/master/wrangler.toml)

### 📥 `init` (*初始化*)

Creates a skeleton `wrangler.toml` in an existing directory. This can be used as an alternative to `generate` if you prefer to clone a repository yourself.

*在一个现有的目录中创建一个架构`wrangler.toml`。如果你喜欢自己克隆一个版本库，这可以作为`generate`的替代方法。*

```
wrangler init <name> --type=["webpack", "javascript", "rust"]
```

All of the arguments and flags to this command are optional:

*这个命令的所有参数和标签是可选的*

- `name`: defaults to the name of your working directory
    
    *你工作目录的默认名字*
    
- `type`: defaults to [“webpack”](https://developers.cloudflare.com/workers/tooling/wrangler/webpack).
    
    *默认 “webpack”*
    

### 🦀⚙️ `build` （*构建*）

Build your project. This command looks at your `wrangler.toml` file and runs the build steps associated with the `"type"` declared there.

Additionally, you can configure different [environments](https://developers.cloudflare.com/workers/tooling/wrangler/configuration/environments).

*构建你的项目。这个命令查看你的`wrangler.toml`文件并运行声明的，与`type`相关的构建步骤。此外，你可以配置不同的[环境]*

### 🔓 `login`   （*登录*）

Authorize Wrangler with your Cloudflare login. This will prompt you with a Cloudflare account login page and a permissions consent page. This command is the alternative to `wrangler config` and it uses OAuth tokens.

*用你的 Cloudflare 登录来授权 Wrangler。这将提示你一个Cloudflare账户登录页面和一个权限同意页面。这个命令是`wrangler config`的替代方案，它使用OAuth令牌。*

```
wrangler login --scopes-list --scopes <scopes>
```

All of the arguments and flags to this command are optional:

*这个命令的所有参数和标签是可选的*

- `scopes-list`: list all the available OAuth scopes with descriptions.
    
    *列出所有可用的OAuth作用域及描述。*
    
- `scopes`: allows to choose your set of OAuth scopes.
    
    *允许选择你的一组OAuth范围。*
    

Read more about this command in [Wrangler Login Documentation](https://developers.cloudflare.com/workers/cli-wrangler/commands#login).

*在 [Wrangler Login Documentation](https://developers.cloudflare.com/workers/cli-wrangler/commands#login)中阅读更多关于此命令的信息*

### 🔧 `config`  (配置)

Authenticate Wrangler with a Cloudflare API Token. This is an interactive command that will prompt you for your API token:

*用Cloudflare API令牌认证Wrangler。这是一个交互式命令，将提示你提供你的 API 令牌。*

```
wrangler configEnter API token:superlongapitoken
```

You can also provide your email and global API key (this is not recommended for security reasons):

*你也可以提供你的电子邮件和全球API密钥（出于安全原因，不建议这样做）。*

```
wrangler config --api-keyEnter email:testuser@example.comEnter global API key:superlongapikey
```

You can also [use environment variables](https://developers.cloudflare.com/workers/tooling/wrangler/configuration/) to configure these values.

*你也可以[use environment variables](https://developers.cloudflare.com/workers/tooling/wrangler/configuration/) 来配置这些值。*

### ☁️ 🆙 `publish`  (*发布*)

Publish your Worker to Cloudflare. Several keys in your `wrangler.toml` determine whether you are publishing to a workers.dev subdomain or your own registered domain, proxied through Cloudflare.

*将你的 Worker 发布到 Cloudflare。你的 `wrangler.toml` 中的几个键决定了你是发布到 workers.dev 子域还是你自己的通过 Cloudflare 代理注册域。*

Additionally, you can configure different [environments](https://developers.cloudflare.com/workers/tooling/wrangler/configuration/environments).

*此外，你可以配置不同的 [environments](https://developers.cloudflare.com/workers/tooling/wrangler/configuration/environments).*

You can also use environment variables to handle authentication when you publish a Worker.

*当你发布一个Worker时，你也可以使用环境变量来处理认证问题*

```
# e.g.CF_API_TOKEN=superlongtoken wrangler publish# where# $CF_API_TOKEN -> your Cloudflare API tokenCF_API_KEY=superlongapikey CF_EMAIL=testuser@example.com wrangler publish# where# $CF_API_KEY -> your Cloudflare API key# $CF_EMAIL -> your Cloudflare account email
```

### 🗂 `kv`

Interact with your Workers KV store. This is actually a whole suite of subcommands. Read more about in [Wrangler KV Documentation](https://developers.cloudflare.com/workers/cli-wrangler/commands#kv).

*与你的Workers KV商店进行互动。这实际上是一整套的子命令。在[Wrangler KV Documentation](https://developers.cloudflare.com/workers/cli-wrangler/commands#kv)中阅读更多内容。*

### 👂 `dev`

`wrangler dev` works very similarly to `wrangler preview` except that instead of opening your browser to preview your worker, it will start a server on localhost that will execute your worker on incoming HTTP requests. From there you can use cURL, Postman, your browser, or any other HTTP client to test the behavior of your worker before publishing it.

*`wrangler dev` 的工作原理与 `wrangler preview` 非常相似，只是它不会打开浏览器来预览您的 Worker，而是会在 localhost 上启动一个服务器，该服务器会在传入的 HTTP 请求中执行您的 Worker。在那里，您可以使用 cURL、Postman、您的浏览器或任何其他 HTTP 客户端，在发布之前测试您的 Worker 的行为。*

You should run wrangler dev from your worker directory, and if your worker makes any requests to a backend, you should specify the host with `--host example.com`.

*你应该从你的工作者目录中运行 wrangler dev，如果你的工作者向后端发出任何请求，你应该用 `--host example.com` 指定主机。*

From here you should be able to send HTTP requests to `localhost:8787` along with any headers and paths, and your worker should execute as expected. Additionally, you should see console.log messages and exceptions appearing in your terminal.

*在这里，您应该能够向 `localhost:8787`发送 HTTP 请求以及任何头信息和路径，并且您的工作者应该按照预期执行。此外，你应该看到终端中出现 console.log 消息和异常情况。*

```
👂 Listening on http://localhost:8787[2020-02-18 19:37:08] GET example.com/ HTTP/1.1 200 OK
```

All of the arguments and flags to this command are optional:

*这个命令的所有参数和标签是可选的*

- `env`: environment to build
    
    *构建环境*
    
- `host`: domain to test behind your worker. defaults to example.com
    
    *你的worker的test域名，默认是example.com*
    
- `ip`: ip to listen on. defaults to localhost
    
    *监听的IP，默认为localhost*
    
- `port`: port to listen on. defaults to 8787
    
    *监听的端口，默认是8787*
    

## Additional Documentation (*附件文件*)

All information regarding wrangler or Cloudflare Workers is located in the [Cloudflare Workers Developer Docs](https://developers.cloudflare.com/workers/). This includes:

有关 wrangler 或 Cloudflare Workers 的所有信息都位于 Cloudflare Workers 开发者文档中。这包括。

- Using wrangler [commands](https://developers.cloudflare.com/workers/tooling/wrangler/commands)
    
    *wrangler命令行用法*
    
- Wrangler [configuration](https://developers.cloudflare.com/workers/tooling/wrangler/configuration)
    
    *wrangler 配置*
    
- General documentation surrounding Workers development
    
    *围绕workers开发的一般文档*
    
- All wrangler features such as Workers Sites and KV
    
    *所有wrangler功能例如Workers Sites和KV*
    

## ✨Workers Sites

To learn about deploying static assets using `wrangler`, see the [Workers Sites Quickstart](https://developers.cloudflare.com/workers/sites/).

*要了解使用`wrangler`部署静态资产的情况，请参阅Workers Sites Quickstart。*
