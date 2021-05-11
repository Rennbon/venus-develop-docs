# Venus wallet

## 概念

- Venus wallet 的出现是为了支持 Filecoin 中远程钱包的概念，它已经被 Venus 采用，它使用`github.com/filecoin-project/go-jsonrpc`作为网络连接库，所以 Lotus 也可以很简单的使用 Venus wallet 作为远程钱包使用。
- 支持 BLST 和 SECP 私钥管理，能够生成随机私钥，或者导入私钥，使用 aes128 对私钥进行对称加密后存储，同时支持私钥对数据的签名。
- 提供签名数据策略化验证，目前支持数 10 种数据结构以及 message 数据结构中的 60 余种概念分类进行统一管理，按需配置各种组合方式绑定私钥签名规则，而后可以将数种不同的私钥签名规则组成一个整体授权于外部组件使用，同时该授权也附带了多个策略组之间的环境隔离。

## 模块

### 1. 私钥管理模块

私钥管理模块主要功能为新增/删除私钥，导入/导出私钥，查看私钥列表，数据签名，设置密码，锁定/解锁私钥仓库运用等功能。调用方式提供 CLI 和 RPC2.0 两种，程序启动后，第一步需要设置保护密码，该密码与配置文件及程序内部自定义的变量结合，通过 aes 对重要数据进行对称加密，同时设置密码后，也会通过 aes，uuid 等算法定向生成一个 root token，供程序本地 CLI 调用，设置密码成功后，程序默认为解锁状态，程序锁的设计是为了安全考虑，只有解锁状态下，重要的接口逻辑才会被真正放行，这也是最直接的程序安全保护关卡，需要终止所有程序调用的时候，可以通过调用加锁来禁用程序的重要对外暴露功能而非直接 kill 掉程序进程。解锁状态下，拥有权限的用户可以通过命令创建或导入私钥，并通过该私钥对数据进行签名并返回签名数据。

### 2. 策略管理模块

策略管理模块主要是配置私钥待签名的数据类型，聚合多个私钥生成一个授权码，通过这个授权码，调用方可以在指定的私钥允许的范围内对一些数据进行签名，同时能查看授权码允许的私钥地址范围。策略的配置可以分 2 步走，第一步就是定义程序使用策略的等级，通过在 config.toml 文件中 Strategy.Level 来设定，0 为禁用策略，1 为初级策略，2 位高级策略。禁用策略下，程序解锁后将放行所有操作，包括私钥管理模块中的创建私钥，签名，导出私钥等。初级策略和高级策略下，只能使用授权码影响下的功能，初级高级的区别在于，初级只对代码签名类型的数据结构做限制，而高级策略则会在数据结构 chainMsg 做深入验证，目前该类型下包含 60 余种概念细分，相对的高级策略的细分需要链上状态数据的支持，在配置高级策略的同时，也需要配置一个带有状态数据的节点作为数据源。

## 配置文件

配置文件为 toml 格式，默认位置为`~/.venus_wallet/config.toml`

第一次启动时会自动生成

```toml
# Default config:
[API]
# 程序启动监听IPV4地址
ListenAddress = "/ip4/0.0.0.0/tcp/5678/http"

[DB]
# sqlit文件路径，目前仅支持sqlite
Conn = "~/.venus_wallet/keystore.sqlit"
Type = "sqlite"
DebugMode = true

[JWT]
# 程序admin JWT Token hex格式，用于本地CLI访问调用
Token = "65794a68624763694f694a49557a49314e694973496e523563434936496b705856434a392e65794a42624778766479493657794a795a57466b4969776964334a70644755694c434a7a615764754969776959575274615734695858302e7133787a356f75634f6f543378774d5463743870574d42727668695f67697a4f7a365142674b2d6e4f7763"
# 程序JWT对称加密secret hex格式
Secret = "7c40ce66a492e35ac828e8333a5703e38b23add87f29bd8fc7343989e08b3458"

[Factor]
# 私钥aes加密变量
ScryptN = 262144
ScryptP = 1

[Strategy]
# 策略等级
# 0：不使用策略  1：只验证数据类型  2：验证数据类型及message的method
Level = 2
# 当策略为2时，需要指定带状态数据的lotus或者venus节点
# 用于获取message.to的actor来验证message对应的method
NodeURL = "/ip4/127.0.0.1/tcp/2345/http"

```

- 开启策略之后，远程 token RPC 调用将限制钱包的创建，删除，导入功能

## package 介绍

### api

> 基于 JsonRPC 2.0 协议的全局接口定义

- permission
  - 通过 json tag 的方式对 JSON RPC2.0 的接口进行权限验证
  - PermissionedAny() 通过反射完成 API 和内部业务的调度
- remotecli
  - wallet 对外提供的便捷接入客户端

### build

> 程序启动及生命周期管理，使用`go.uber.org/fx`

### cli

> 维护所有 CLI 操作方法，通过 config 中的 JWT token，API endpoint 及用户的输入的 password 构建 RPC 连接方式将用户输入的指令发送到服务端

- cli_strategy：钱包策略化配置模块命令
- helper：Json RPC 连接构建帮助
- cmds：command 统一管理入口

### cmd

> 程序启动相关

- mock：程序断点调试入口，方便 CLI 及 RPC 调用代码调试，做到更快的排错
- wallet：程序 Venus-wallet 可执行文件编译 main 入口
- daemon：程序进程启动命令，提供给 mock 及 wallet 使用
- rpc：程序网络统一管理

### common

> 通用弱业务相关，目前包含版本号，统一 JWT token 接口权限管理相关业务逻辑

### config

> 配置文件管理

### core

> 顶层业务及类型管理，包含上层库结构体映射，策略依赖管理等

- types：一些上层库的数据结构映射及一些简单全局数据结构的定义
- msgtypes：可签名数据类型定义，抽象覆盖所有 Filecoin 签名类型
- msg_router：签名类型转换，结合 msgtypes 将远端 RPC 调用的参数转换为可签名实体
- msg_enum：策略化中数据结构类型判断相关
- method：配置策略 Level 为 2 时会用到，判断 msg method 的依赖库，引用了 specs-actors 库，需要跟进 specs-actors 版本的迭代

### crypto

> 密码学相关 package

- aes：对称加密算法，运用于私钥加密存储等
- key：私钥接口抽象及私钥工厂方法
- bls：BLST PrivateKey 实现
- secp：secp PrivateKey 实现

### errcode

> 一些公用 error 定义

### example

> venus-wallet 调用案例

### filemgr

> 文件系统，用于管理根目录文件

- config_manager： 配置文件管理
- fs：文件系统
- jwt：程序 JWT secret 生成

### log

> 设置 log 等级

### middleware

> 一些数据监控中间件

### node

> 连接状态数据节点的客户端（支持 Lotus，Venus），目前用于获取指定地址的 Actor，推算 msg 的 method

### storage

> 存储及业务相关

- sqlite
  - conn: sqlite 单例模式，防并发死锁
  - key_store: 私钥 CRUD sqlite 实现
  - strategy_store: 策略 CRUD sqlite 实现
- strategy 策略管理模块
  - cache: 策略缓存，原子性控制
  - strategy: 策略业务封装，分为 2 个概念，IStrategy 为策略逻辑构建，IStrategyVerify 为构建后的策略提供给 wallet 逻辑使用
- wallet 钱包管理模块
- keymix 钱包管理中间层，密码，root token，私钥加解密，钱包加解锁等
- keystore 私钥存储抽象
- strategy_store 策略存储抽象

### version

> wallet 版本控制，发布 release 时需要修改对应 BuildVersion

## Venus-wallet API

模拟请求：

```
curl --request POST \
  --url http://localhost:5678/rpc/v0 \
  --header 'Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJBbGxvdyI6WyJyZWFkIiwid3JpdGUiLCJzaWduIiwiYWRtaW4iXX0.ctjMc_n-W3nBuh4uPau6eVXu1mdLZ0qlJuF7gRp6SCs' \
  --header 'Content-Type: application/json' \
  --header 'StrategyToken: 62d3c94c-86d1-11eb-b252-acde48001122' \
  --header 'cache-control: no-cache' \
  --data '{
    "jsonrpc": "2.0",
    "method": "Filecoin.WalletSign",
    "params": [
        "f3rgwrxcz42gsueww2krnjtrfjcao6i6apq2tcthi3rhnybdimw4lvydyxnmt27bru3gqxe4pfcmhe5a6zro3a",
        "Ynl0ZSBhcnJheQ==",
        {
            "Type": "unknown",
            "Extra": ""
        }
    ],
    "id": 1
}'
```

- Authorization: `Bearer ` + JWT Token
- StrategyToken: 策略 token,

### Wallet

#### 1. WalletNew

> 创建钱包

- perms: admin
- method: `Filecoin.WalletNew`
- params:

```
[
    "bls"
]
```

- response:

```
{
    "jsonrpc": "2.0",
    "result": "f3rgwrxcz42gsueww2krnjtrfjcao6i6apq2tcthi3rhnybdimw4lvydyxnmt27bru3gqxe4pfcmhe5a6zro3a",
    "id": 1
}
```

#### 2. WalletHas

> 检查钱包是否存在

- perms: write
- method: `Filecoin.WalletHas`
- params:

```
[
    "f3rgwrxcz42gsueww2krnjtrfjcao6i6apq2tcthi3rhnybdimw4lvydyxnmt27bru3gqxe4pfcmhe5a6zro3a"
]
```

- response:

```
{
    "jsonrpc": "2.0",
    "result": true,
    "id": 1
}
```

#### 3. WalletList

> 获取钱包列表

- perms: write
- method: `Filecoin.WalletList`
- params: null
- response:

```
{
    "jsonrpc": "2.0",
    "result": [
        "f3rgq44gyg56fv3776cgerzvile52ciax5yvs5i25rhzhkswihqbltmjgfqtppfmythpsiypd72gcdoftp7oba",
        "f3xbwgfrqaigqmlcp3qqggu3go6664cgiajg4qyqthkhypo7zqe2lsjodxrxyhps7vtifdush2vk7pfzwmxyqq",
        "f3wopfvztkr4ggr3k72cduddoh7nstphky5f4yc2bdkubi3ysdu2rp5uvnqynrmesk27dxkai4gqt54e3klg4q"
    ],
    "id": 1
}

```

#### 4. WalletExport

> 导出钱包

- perms: admin
- method: `Filecoin.WalletExport`
- params:

```
[
    "f3rgwrxcz42gsueww2krnjtrfjcao6i6apq2tcthi3rhnybdimw4lvydyxnmt27bru3gqxe4pfcmhe5a6zro3a"
]
```

- response:

```
{
    "jsonrpc": "2.0",
    "result": {
        "Type": "bls",
        "PrivateKey": "LqTCZdSufXP/IqnEIu3G9e0kNIS5r+EwaoG6sPPUDz4="
    },
    "id": 1
}
```

#### 5. WalletDelete

> 删除钱包

- perms: admin
- method: `Filecoin.WalletDelete`
- params:

```
[
    "f3rgwrxcz42gsueww2krnjtrfjcao6i6apq2tcthi3rhnybdimw4lvydyxnmt27bru3gqxe4pfcmhe5a6zro3a"
]
```

- response:

```
{
    "jsonrpc": "2.0",
    "id": 1
}
```

#### 6. WalletImport

> 导入钱包

- perms: admin
- method: `Filecoin.WalletImport`
- params:

```
[
    {
        "Type": "bls",
        "PrivateKey": "LqTCZdSufXP/IqnEIu3G9e0kNIS5r+EwaoG6sPPUDz4="
    }
]
```

- response:

```
{
    "jsonrpc": "2.0",
    "result": "f3rgwrxcz42gsueww2krnjtrfjcao6i6apq2tcthi3rhnybdimw4lvydyxnmt27bru3gqxe4pfcmhe5a6zro3a",
    "id": 1
}
```

#### 7. WalletSign

> 导入钱包

- perms: sign
- method: `Filecoin.WalletSign`
- params:

```
[
    "f3rgwrxcz42gsueww2krnjtrfjcao6i6apq2tcthi3rhnybdimw4lvydyxnmt27bru3gqxe4pfcmhe5a6zro3a",
    "Ynl0ZSBhcnJheQ==",
    {
        "Type": "unknown",
        "Extra": ""
    }
]
```

- response:

```
{
    "jsonrpc": "2.0",
    "result": {
        "Type": 2,
        "Data": "mX8dp4JpCAit+xmQKotUF2v7fTdSkh9U83EB809kZvGMkCRrUE8wPOuHKpNMYOo/Bt1deUnaJ4pRAq8H3fH71YiQFjxWiQxHxXd/KAF1g10cHisGtnIMi+O3rqnDoaiS"
    },
    "id": 1
}
```

## CLI
