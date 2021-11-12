# 端云联调操作：Proxied

- [简介与原理](#简介与原理)

- [快速使用](#快速使用)

  - [简单使用](#简单使用)

    - [准备工作](#准备工作)
    - [本地调用](#本地调用)
    - [清理工作](#清理工作)

  - [断点调试](#断点调试)

---

阿里云函数计算（FC）组件为使用者提供了 FC 相关资源本地与云端联合调试的能力。可以通过`proxied`指令，快速进行端云联调操作。

您可以通过`proxied -h`/`proxied --help`参数，唤起帮助信息。例如执行`s proxied -h`后，可以看到：

```
Proxied

  Local invoke via proxied service.

Detail

  Local invoke with real net traffic via proxied service.

SubCommand List

  setup    Setup the preconditions.
  invoke   Invoke local function.
  clean    Clean the related resource and environment..

Usage

  s proxied <SubCommand> <options>


Global Options

  -d, --debug string   Output debug informations.
  -h, --help string    Display help for command.

Example1

  Help for setup.    $ s proxied setup -h
  Help for invoke.   $ s proxied invoke -h
  Help for clean.    $ s proxied cleanup -h

```

Proxied 命令为我们提供了三个子命令：

- setup: 准备端云联调的辅助资源和相关环境，可以通过`s proxied setup -h`获取帮助文档

  ```
  Setup

    Setup Operation.

  Detail

    Setup for local invoke via proxied service.

  Usage

    s proxied setup <options>


  Options

    -c, --config string       Select which IDE to use when debugging and output related debug config tips
    for the IDE. Options：'vscode'.
    --debug-args string       Additional parameters that will be passed to the debugger.
    -d, --debug-port string   Specify the sandboxed container starting in debug mode, and exposing this
    port on localhost.
    --debugger-path string    The path of the debugger on the host.
    --tmp-dir string          The temp directory mounted to /tmp , default to
    './.s/tmp/invoke/serviceName/functionName/'

    Global Options

    -d, --debug string   Output debug informations.
    -h, --help string    Display help for command.

  Example

    Just setup.         $ s proxied setup
    Setup with debug.   $ s proxied setup --config vscode --debug-port 3000

  ```

- invoke: 调用本地 FC 函数，可以通过`s proxied invoke -h`获取帮助文档

  ```
  Invoke

    Invoke local function.

  Detail

    Invoke local function in the container.Need setup first

  Usage

    s proxied invoke <options>


  Options

    -e, --event string         Event data (strings) passed to the function during invocation (default:
                               "").Http function format refers to [https://github.com/devsapp/fc-remote-
                               invoke#特别说明]
    -f, --event-file string    Event funtion: A file containing event data passed to the function during
                               invoke. Http function: A file containing http request options sent to https
                               strigger. Format refers to [https://github.com/devsapp/fc/blob/main/docs/Usage/invoke.md#invoke-http-parameter]
    -s, --event-stdin string   Read from standard input, to support script pipeline.Http function format
                               refers to [https://github.com/devsapp/fc/blob/main/docs/Usage/invoke.md#invoke-http-parameter]

  Global Options

    -d, --debug string   Output debug informations.
    -h, --help string    Display help for command.

  Example

    Just invoke.         $ s proxied invoke
    Invoke with event.   $ s proxied invoke --event string

  ```

- clean: 清理本次端云联调的辅助资源和相关环境，可以通过`s proxied cleanup -h`获取帮助文档

  ```
  Clean

    Clean the related resource and environment.

  Detail

    Clean the helper resource and the local container.

  Usage

    s proxied cleanup <options>


  Global Options

    -d, --debug string   Output debug informations.
    -h, --help string    Display help for command.

  Example

    Just cleanup.   $ s proxied cleanup

  ```

# 简介与原理

## 背景

首先我们从两份报告开始：

![image.png](https://img.alicdn.com/imgextra/i4/O1CN017typlI1k8i3m4l7rQ_!!6000000004639-2-tps-3558-1360.png)
从报告中我们开始看出， 在 Serverless/FaaS 领域中， 调试成为了一个最突出的痛点， 虽然云厂商也提供了一个工具， 比如阿里云函数计算的上一代工具 Funcraft、AWS lambda 的 SAM Local， 但是都是集中在本地执行环境模拟调试 + mock 参数阶段， 会存在如下痛点问题：


- 开发的函数需要访问只有 vpc 内网地址的数据库或者 kafka 等

- 函数计算挂载 NAS 功能， 目录只能 mock

- 开发的函数需要从 oss 下载上传， 线上函数可以直接使用 internal endpoint, 本地调试只能临时改成 public endpoint, 比如大点音视频， 除了函数执行时间长， 还有 OSS 公网流量费用

- 表格存储触发器， 触发函数的 event 是二进制 cbor 格式的 json, 基本很难 mock

- Context 中的 creds 不真实， 不是真实基于 service role 构建， 等函数部署以后， 可能不符合预期

- ...

简单总结下来就是：执行环境出的流量不能触达指定 vpc 或者内网 endpoint 服务， 入的流量（函数的入参 context 和 event） 不够真实。
​

## 原理


# 快速使用

当我们下载好[Serverless Devs 开发者工具](../Getting-started/Install-tutorial.md), 并完成[阿里云密钥配置](../Getting-started/Setting-up-credentials.md)之后，我们可以根据自身的需求进行函数的端云联调。如果你使用的是非 admin 全息的子账号的 AK, 需要给这个子账号增加如下 policy:

```json
{
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "tns:*",
            "Resource": "*"
        }
    ],
    "Version": "1"
}
```

## 简单使用

### 准备工作

首先，执行 `s proxied setup` 来准备端云联调所需的辅助资源以及本地环境。执行完成后会阻塞住：

```
$ s proxied setup
[2021-07-13T08:51:51.670] [INFO ] [S-CLI] - Start ...
✔ Session created, session id: S-d1564a76-9e6b-4da1-8ecc-8c1378c6a330.
[2021-07-13T08:51:53.399] [INFO ] [FC-PROXIED-INVOKE] - Deploying helper function...
...
[2021-07-13T08:51:56.868] [INFO ] [FC-DEPLOY] - Creating service: SESSION-S-d1564
[2021-07-13T08:51:56.868] [INFO ] [FC-DEPLOY] - Creating function: python-event
✔ Make service SESSION-S-d1564 success.
✔ Make function SESSION-S-d1564/python-event success.
[2021-07-13T08:51:57.340] [INFO ] [FC-DEPLOY] - Checking Service SESSION-S-d1564 exists
[2021-07-13T08:51:57.541] [INFO ] [FC-DEPLOY] - Checking Function python-event exists

There is auto config in the service: SESSION-S-d1564
✔ Helper function is set to 1 provison and 0 elasticity.
✔ Proxy container is running.
✔ Session established!
[2021-07-13T08:52:22.322] [INFO ] [FC-PROXIED-INVOKE] - Skip pulling image aliyunfc/runtime-python3.6:1.9.18...
End of method: proxied
FunctionCompute python3 runtime inited.
```

如果在当前的 yaml 中有多个项目，需要指定某个项目进行测试，例如`s <projectName> proxied setup`

### 本地调用

对于无触发器的普通事件函数或者 http 触发器函数，准备工作完成后，启动另一个新的终端，切换到该项目路径下，执行 `s proxied invoke` 来调用本地函数。

```
# 如果是 event 函数， 直接使用 s proxied invoke -e '{}'
# 如果是 http trigger 则可以使用
# s invoke -e '{"body":123,"method":"GET","headers":{"key":"value"},"queries":{"key":"value"},"path":"string"}'

$ s proxied invoke -e '{}'
[2021-07-13T08:55:05.260] [INFO ] [S-CLI] - Start ...
========= FC invoke Logs begin =========
Not all function logs are available, please retry
FC Invoke End RequestId: bb720e13-e57a-4040-a920-82621e275ff1
Duration: 42.66 ms, Billed Duration: 43 ms, Memory Size: 512 MB, Max Memory Used: 40.85 MB
========= FC invoke Logs end =========

FC Invoke Result:
hello world
```

对于具有触发器的事件函数，首先需要确定事件类型(例如 oss 事件， cdn 事件等), 把触发器临时指向生成的辅助 service/function(比如本文 setup 命令中输出的 SESSION-S-d1564/python-event)，然后去线上实际触发相应的事件，即可调用本地函数。

### 清理工作

上述调试完成后，执行 `s proxied cleanup` 清理端云联调所需的辅助资源以及本地环境。

```
$ s proxied cleanup
[2021-07-13T08:59:46.635] [INFO ] [S-CLI] - Start ...
✔ Stop container succeed.
✔ Unset helper function provision and on-demand config done.
[2021-07-13T08:59:49.599] [INFO ] [FC-DEPLOY] - Using region: cn-hangzhou
[2021-07-13T08:59:49.599] [INFO ] [FC-DEPLOY] - Using access alias: default
[2021-07-13T08:59:49.599] [INFO ] [FC-DEPLOY] - Using accessKeyID: ***********3743
[2021-07-13T08:59:49.599] [INFO ] [FC-DEPLOY] - Using accessKeySecret: ***********PeUX
[2021-07-13T08:59:49.942] [INFO ] [FC-DEPLOY] - Checking Service SESSION-S-d1564 exists
[2021-07-13T08:59:50.025] [INFO ] [FC-DEPLOY] - Service: SESSION-S-d1564 already exists online.
[2021-07-13T08:59:50.126] [INFO ] [FC-DEPLOY] - Checking Function python-event exists
[2021-07-13T08:59:50.165] [INFO ] [FC-DEPLOY] - Function: python-event already exists online.
📎 Using fc deploy type: sdk, If you want to deploy with pulumi, you can [s cli fc-default set deploy-type pulumi] to switch.
✔ Delete function SESSION-S-d1564/python-event success.
✔ Delete service SESSION-S-d1564 success.
✔ Delete done.
✔ Stop container succeed.
```

清理完成后 `s proxied setup` 阻塞住的进程也会随之退出。

## 断点调试

目前仅支持 vscode 以及 intellij 编辑器的断点调试。操作案例可以参考：

- [vscode 断点调试案例](../../examples/fc-proxied-invoke/python-event)
- [intelli 断点调试案例](../../examples/fc-proxied-invoke/java8-event)
