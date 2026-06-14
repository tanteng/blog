---
title: "用本地 Stub 解决 Go Proto 冲突"
date: 2026-05-28
draft: false
tags: ["go", "golang", "protobuf", "microservices", "protocol"]
categories: ["tech"]
description: "当引入一个内部 RPC 依赖导致两个 proto 文件冲突服务直接 panic 时，如何通过手写本地精简 Stub 绕过注册机制，以最小代价解决这一经典 Go protobuf 依赖地狱问题。"
---

在 Go Monorepo 项目中，引入一个内部 RPC 依赖本该是一件简单的事——加一行 `require`，配一个 `replace`，代码编译通过，似乎万事大吉。但服务一启动，直接 panic：

```text
panic: proto: file "validate/validate.proto" is already registered
panic: proto: file "common.proto" has a name conflict over trpc.myservice.common.Team
```

两个 panic，两个不同的 proto namespace 冲突。这是 Go protobuf 生态中一个经典的「传递依赖地狱」问题。

本文记录一次真实问题的根因分析和解决方案：如何通过手写一个**本地精简 Stub**，以最小代码代价绕过 proto 注册机制，彻底消除冲突。

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
 previously from: "buf.build/gen/go/envoyproxy/protoc-gen-validate/protocolbuffers/go/validate"
 currently from: "git.example.com/devsec/protoc-gen-secv/validate"

panic: proto: file "common.proto" has a name conflict over trpc.myservice.common.Team
 previously from: "git.example.com/myservice/sword/protocols/legacy/myservice/common"
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

## 解决方案：本地精简 Stub（Local Minimal Stub）

核心思路：**既然冲突来自 proto 注册，那就不注册**。

不使用 user 模块生成的完整 `.pb.go`（里面有大量 `init()` 注册逻辑），而是创建一个**只包含业务所需最小 proto 定义**的本地 stub 目录，通过 Makefile 编译生成精简的 stub 代码，彻底绕过 proto 注册机制。

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
└── stubs.go              ← 确保不注册 proto 的空壳文件
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

编译后会生成 `pb/user.pb.go`，但由于原始 proto 里没有 `import "validate/validate.proto"`，生成的代码不会触发冲突。

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
var NewUserServiceClientProxy = func(opts ...client.Option) UserServiceClientProxy {
    return &userServiceClientProxyImpl{client: client.DefaultClient, opts: opts}
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

**第五步：创建 stubs.go 防止意外注册**

在包根目录添加一个空壳文件，确保即使有人误在 `pb/` 目录外引用了 proto 注册逻辑，也不会生效：

```go
// stubs.go
// 空壳文件，用于占据 proto 注册名空间，防止意外注册冲突。
// 所有 proto 定义均在 pb/ 目录下，不在包级别引入任何注册逻辑。
package user

// 本文件不做任何事，仅用于阻断错误的 import 行为。
```

**第六步：清理 `go.mod`**

```diff
- git.example.com/devsec/protoc-gen-secv => ./stubs/protoc-gen-secv
- git.example.com/trpcprotocol/myservice/common => ./stubs/trpcprotocol-myservice-common
- git.example.com/internal/user => ./protocols/legacy/user

- git.example.com/internal/user v1.2.10
- git.example.com/devsec/protoc-gen-secv v0.3.4 // indirect
```

**第七步：更新业务代码的 import 路径**

```go
// 之前
import "git.example.com/internal/user"

// 之后（主模块内部路径）
import "git.example.com/myservice/protocols/legacy/user"
```

**第八步：编译**

```bash
cd protocols/legacy/user && make proto
```

生成精简后的 `pb/user.pb.go`，不再包含任何 `validate.proto` 的注册逻辑。

## 效果对比

| 维度 | 直接引入模块 | 本地精简 Stub |
|------|------------|--------------|
| `go.mod` 新增条目 | +5 行（require + replace + 传递依赖） | 0 |
| proto 注册冲突 | 2 处 panic | 无 |
| 代码体积 | 数百 KB pb.go + 数十 KB trpc.go | ~5KB |
| 传递依赖 | `protoc-gen-secv`、`common` 等 | 无新增 |
| 可维护性 | 跟随上游版本升级 | proto 源文件手动维护 |
| 构建方式 | 直接 go build | `make proto` 编译生成 |

## 适用场景与边界

### 什么时候适合用这个方案

- **只需要外部服务的少数几个接口**（1~3 个 RPC 方法）
- **外部模块带来了 proto namespace 冲突**，且无法通过 `replace` 解决
- **外部模块是内部服务**，proto 字段定义稳定，不会频繁变更
- **Monorepo 场景**，需要严格控制 `go.mod` 的依赖规模

### 什么时候不适合

- 需要调用外部服务的大量接口（>10 个），手写维护成本过高
- 外部服务的 proto 定义频繁变更，手动同步容易遗漏
- 外部模块是第三方公共库（如 AWS SDK），不应手写 stub

### 注意事项

1. **proto 字段编号必须与原始定义完全一致**，否则序列化/反序列化会出错。proto 是来源依据，字段编号是契约，不能改。

2. **RPC 路径必须与服务端完全一致**，包括包名、服务名、方法名：

   ```go
   msg.WithClientRPCName("/trpc.example.user.UserService/GetUserInfo")
   ```

3. **`pb/` 目录应加入 `.gitignore`**，避免提交生成的二进制文件，只保留 proto 源文件。

4. **建议在 proto 文件头注释中注明原始来源**，方便后续维护者溯源。

## 更深层的思考

这个方案本质上是在做一件事：**把「依赖外部模块」变成「依赖外部协议」**。

外部模块（Go module）是一个有版本、有依赖树的完整单元。当你 `require` 它，你接受了它的全部传递依赖。

外部协议（proto 字段定义 + RPC 路径）是一个稳定的契约。只要服务端不改字段编号和方法路径，这个契约就不会失效。

在微服务架构中，服务间通信本来就应该通过「协议契约」解耦，而不是通过「共享代码库」耦合。本地精简 Stub 只是把这个理念落地到了 Go 模块层面：**我只依赖你的协议，不依赖你的实现**。

这也是为什么 gRPC 生态里有 `protoc-gen-go` 这类工具——proto 文件本身就是语言无关的契约，生成的代码只是契约的一种表达形式。当生成的代码带来了不必要的依赖负担，手写最小化实现反而是更纯粹的做法。