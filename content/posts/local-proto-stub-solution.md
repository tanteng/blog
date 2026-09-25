---
title: "用本地 Stub 解决 Go Proto 冲突"
date: 2025-05-28
draft: false
tags: ["golang", "microservices", "protocol"]
categories: ["tech"]
description: "引入一个内部 RPC 依赖导致 proto 命名空间冲突、服务启动直接 panic 时，如何用一份手写的 5KB 精简 Stub 替换掉整个外部模块，从根上消掉 Go protobuf 的传递依赖地狱。"
---

在 Go Monorepo 项目中引入一个内部 RPC 依赖，本该是加一行 `require`、配一个 `replace` 就完事。但服务一启动直接 panic：

```text
panic: proto: file "validate/validate.proto" is already registered
panic: proto: file "common.proto" has a name conflict
   over trpc.myservice.common.Team
```

这是 Go protobuf 生态里经典的传递依赖地狱：同一个 proto 文件被两个不同的 Go 包各自注册了一次。最后的解法是手写一份**本地精简 Stub**（约 5KB）替换掉整个外部模块——既然冲突来自注册，那就干脆不注册。

<!--more-->

## 背景

需要调用一个内部 RPC 服务（`user`）的某个接口，最直接的做法是在 `go.mod` 中引入：

```go
require git.example.com/internal/user v1.2.10

replace git.example.com/internal/user => ./protocols/legacy/user
```

编译通过，服务启动后 panic。错误信息指向两个 proto 文件：

```text
panic: proto: file "validate/validate.proto" is already registered
 previously from: "buf.build/gen/go/envoyproxy/protoc-gen-validate
   /protocolbuffers/go/validate"
 currently from: "git.example.com/devsec/protoc-gen-secv/validate"

panic: proto: file "common.proto" has a name conflict
   over trpc.myservice.common.Team
 previously from: "git.example.com/myservice/sword/protocols/legacy
   /myservice/common"
 currently from: "git.example.com/trpcprotocol/myservice/common"
```

## 根因分析

### Proto 注册机制

Go 的 protobuf 运行时（`google.golang.org/protobuf`）在程序启动时，每个 `.pb.go` 文件的 `init()` 函数都会调用 `proto.RegisterFile()` 向全局注册表注册自己。**同一个 proto 文件名（full name）只能被注册一次**，如果两个不同的 Go 包都包含了同名 proto 文件的注册逻辑，直接 panic，没有覆盖或合并的机制。

### 冲突是怎么产生的

引入 `user` 模块后，它的 `user.pb.go` 的传递依赖注册了：

- `git.example.com/devsec/protoc-gen-secv/validate` → 注册了 `validate.proto`
- `git.example.com/trpcprotocol/myservice/common` → 注册了 `common.proto`

而主项目里已经通过其他路径引入了：

- `buf.build/gen/go/envoyproxy/protoc-gen-validate` → 也注册了 `validate.proto`
- `git.example.com/myservice/sword/protocols/legacy/myservice/common` → 也注册了 `common.proto`

同一个 proto 文件，两个不同的 Go 包，各自在 `init()` 里注册，冲突不可避免。

### 为什么 `replace` 解决不了

`replace` 只能重定向模块路径，但 `user` 模块的 `go.mod` 里声明的依赖（`protoc-gen-secv`、`trpcprotocol/myservice/common`）是它自己的传递依赖，主模块无法通过 `replace` 完全消除这些传递依赖带来的 proto 注册。

### 为什么不直接用官方开关

protobuf-go 自己提供了两个把冲突从 panic 降级成警告的开关：

```bash
# 运行期
GOLANG_PROTOBUF_REGISTRATION_CONFLICT=warn ./main

# 编译期
PKG=google.golang.org/protobuf/reflect/protoregistry
go build -ldflags "-X $PKG.conflictPolicy=warn"
```

代价是冲突并没有消失，只是被静音了。全局注册表里同一个名字只能有一个描述符生效，实际生效的是先注册的那个——由包的 `init()` 顺序决定，而不是由你决定。一旦两份同名 proto 的字段编号有出入，序列化就会用错描述符，问题会跑到离事发地很远的地方才暴露。protobuf-go 官方也只把它列为 workaround，正式建议始终是把冲突源修掉。所以这两个开关适合临时救火，不适合当作长期方案。

## 解决方案：本地精简 Stub（Local Minimal Stub）

核心思路：既然冲突来自 proto 注册，那就不注册。

具体做法是不使用 `user` 模块生成的完整 `.pb.go`（里面有大量 `init()` 注册逻辑），而是创建一个**只包含业务所需最小 proto 定义**的本地 stub 目录，用 Makefile 编译生成精简代码，彻底绕过 proto 注册机制。

先把名字说清楚：本文说的 stub 指 RPC 的客户端代理（client proxy），不是单元测试里的测试替身（test stub）。这两个词在中文里经常被混着用，这里指前者。

### 实施步骤

**第一步：创建本地 stub 目录结构**

```
protocols/legacy/user/
├── proto/
│   └── user.proto        ← 精简后的 proto 定义
├── pb/                   ← 编译生成的 pb 文件目录（加入 .gitignore）
│   └── user.pb.go
├── user.trpc.go          ← tRPC 客户端代理（手写）
├── Makefile              ← 编译命令
└── stubs.go              ← 包根目录的占位文件
```

注意：**`pb/` 目录不加入版本控制**，只提交 proto 源文件。

**第二步：编写精简 proto 文件**

从服务端获取原始 proto 定义后，只保留业务所需的接口，删除其他所有 message 定义：

```protobuf
// proto/user.proto
// 精简版 user 服务 proto，仅包含 GetUserInfo 接口
// 原始定义来源：git.example.com/internal/user

syntax = "proto3";

package trpc.example.user;

option go_package = "git.example.com/myservice/protocols/legacy/user/pb";

service UserService {
  rpc GetUserInfo(GetUserInfoRequest) returns (GetUserInfoResponse);
}

message GetUserInfoRequest {
  string user_id = 1;
  string lang = 2;
}

message GetUserInfoResponse {
  string nickname = 1;
  string email = 2;
  string avatar = 3;
}
```

关键点：**删除了 `import "validate/validate.proto"` 等所有传递依赖**，只保留最核心的定义。

**第三步：编写 Makefile**

```makefile
# proto 编译目标
.PHONY: proto clean

PROTO_DIR := $(dir $(realpath $(lastword $(MAKEFILE_LIST))))
PB_DIR := $(PROTO_DIR)/pb

# 编译 proto 生成 pb 文件
proto:
	@mkdir -p $(PB_DIR)
	protoc \
		--go_out=$(PB_DIR) \
		--go_opt=paths=source_relative \
		-I=$(PROTO_DIR) \
		$(PROTO_DIR)/proto/user.proto

	@echo "Proto compiled to $(PB_DIR)"

# 清理生成的 pb 文件
clean:
	rm -rf $(PB_DIR)
```

在 `protocols/legacy/user` 目录下执行 `make proto`，会生成 `pb/user.pb.go`。由于原始 proto 里没有 `import "validate/validate.proto"`，生成的代码不会触发任何注册冲突。

**第四步：手写 tRPC 客户端代理**

tRPC 框架除了生成 pb 文件外，还需要一个 `*.trpc.go` 文件来处理 RPC 调用逻辑。这个文件手写，只包含业务需要的接口：

```go
// user.trpc.go
// Package user 是 trpc.example.user 服务的本地精简副本，
// 仅包含 GetUserInfo 接口，规避 proto namespace 冲突。
// 原始定义：git.example.com/internal/user
package user

import (
    "context"
    "git.example.com/trpc-go/trpc-go/client"
    "git.example.com/trpc-go/trpc-go/codec"
)

// UserServiceClientProxy user 服务客户端代理接口
type UserServiceClientProxy interface {
    GetUserInfo(
        ctx context.Context,
        req *GetUserInfoRequest,
        opts ...client.Option,
    ) (*GetUserInfoResponse, error)
}

// NewUserServiceClientProxy 创建 user 服务客户端代理
var NewUserServiceClientProxy = func(
    opts ...client.Option,
) UserServiceClientProxy {
    return &userServiceClientProxyImpl{
        client: client.DefaultClient,
        opts:   opts,
    }
}

type userServiceClientProxyImpl struct {
    client client.Client
    opts   []client.Option
}

func (c *userServiceClientProxyImpl) GetUserInfo(
    ctx context.Context,
    req *GetUserInfoRequest,
    opts ...client.Option,
) (*GetUserInfoResponse, error) {
    ctx, msg := codec.WithCloneMessage(ctx)
    defer codec.PutBackMessage(msg)
    msg.WithClientRPCName("/trpc.example.user.UserService/GetUserInfo")
    // ... tRPC 调用逻辑（根据实际框架补充）
    return &GetUserInfoResponse{}, nil
}
```

**第五步：固定「根目录只放手写代码」的约定**

在包根目录放一个空的 `stubs.go`：

```go
// stubs.go
// 包根目录的占位文件，声明包名并固化目录约定。
package user
```

需要说明的是，一个空的 Go 文件并没有能力阻止任何 `init()` 执行——网上流传的「空壳文件可以占据 proto 注册名空间」并不成立。它真正的价值是把「根目录只放手写代码、生成代码一律待在 `pb/` 子包」这条约定显式写下来，后来的维护者不容易误把外部模块的 `.pb.go` 拷回根目录、把冲突重新引进来。

**第六步：清理 `go.mod`**

```diff
- git.example.com/devsec/protoc-gen-secv => ./stubs/protoc-gen-secv
- git.example.com/trpcprotocol/myservice/common => ./stubs/common
- git.example.com/internal/user => ./protocols/legacy/user

- git.example.com/internal/user v1.2.10
- git.example.com/devsec/protoc-gen-secv v0.3.4 // indirect
```

diff 里出现的 `./stubs/*` 是本地空壳模块目录（里面只有一行 `module` 声明的空 `go.mod`），和 `./protocols/legacy/user` 这个带 proto 源码的精简 stub 不是一回事：前者只是把依赖重定向到一个空模块，后者才是真正在本地生成客户端代码。

**第七步：更新业务代码的 import 路径**

```go
// 之前
import "git.example.com/internal/user"

// 后（主模块内部路径）
import "git.example.com/myservice/protocols/legacy/user"
```

## 效果对比

| 维度 | 直接引入模块 | 官方 `conflictPolicy=warn` | 本地精简 Stub |
|------|------------|--------------------------|--------------|
| `go.mod` 新增条目 | +5 行（require + replace + 传递依赖） | +5 行 | 0 |
| 启动能否跑起来 | 否，panic | 是 | 是 |
| proto 注册冲突 | 2 处 panic | 未消除，生效的描述符由注册顺序决定 | 无 |
| 代码体积 | 数百 KB `pb.go` + 数十 KB `trpc.go` | 数百 KB | ~5KB |
| 传递依赖 | `protoc-gen-secv`、`common` 等 | 一个没少 | 无新增 |
| 可维护性 | 跟随上游版本升级 | 跟随上游版本升级 | proto 源文件手动维护 |
| 构建方式 | 直接 `go build` | 额外加 `-ldflags` | `make proto` 编译生成 |

## 适用边界

适合这个方案的前提有四条：只调外部服务的少数几个接口（1~3 个 RPC 方法）；外部模块带来了 `replace` 消不掉的 proto namespace 冲突；对方是字段定义稳定的内部服务；以及 Monorepo 场景下需要严格控制 `go.mod` 规模。

反过来，接口数量超过 10 个（手写维护成本急剧上升）、对方 proto 频繁变更（手动同步容易遗漏）、或者对方是第三方公共库（比如 AWS SDK，不该手写它的 stub），都应该回到正常引入模块的路子上。

### 注意事项

1. **proto 字段编号必须与原始定义完全一致**，否则序列化/反序列化会出错。proto 是来源依据，字段编号是契约，不能改。

2. **RPC 路径必须与服务端完全一致**，包括包名、服务名、方法名：

   ```go
   msg.WithClientRPCName("/trpc.example.user.UserService/GetUserInfo")
   ```

3. **`pb/` 目录应加入 `.gitignore`**，避免提交生成的二进制文件，只保留 proto 源文件。

4. **建议在 proto 文件头注释中注明原始来源**，方便后续维护者溯源。

## 依赖协议，而非依赖实现

这个方案本质上只做了一件事：把「依赖外部模块」换成「依赖外部协议」。外部模块是有版本、有依赖树的完整单元，`require` 它就等于接受了它的全部传递依赖，包括那些会在 `init()` 里抢注全局名字的代码；而 proto 字段编号加上 RPC 方法路径是一份稳定的契约，只要服务端不改这两样，客户端就不会失效。

微服务之间本来就该通过协议契约通信，而不是通过共享代码库耦合。本地精简 Stub 只是把这个原则落到了 Go module 这一层：我只依赖你的协议，不依赖你的实现。proto 文件本身是语言无关的契约，生成的代码只是契约的一种表达形式；当生成代码带来的依赖负担超过它的价值时，手写一份最小实现反而更接近原意。