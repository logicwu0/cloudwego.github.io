---
title: "扩展 Service 代码生成模板"
date: 2022-11-03
weight: 5
description: >
---

## 介绍

从 v0.4.3 版本开始，Kitex 代码生成工具新增一个名为 `-template-extension` 的参数，支持了对 Service 生成模板进行一定程度的扩展。

- **用法**：`kitex -template-extension extensions.yaml YOUR_IDL`。

其中 **extensions.yaml** 是一个 YAML 文件，其内容必须是一个 [TemplateExtension](https://pkg.go.dev/github.com/cloudwego/kitex/tool/internal_pkg/generator#TemplateExtension) 对象的序列化结果。这个对象的各个字段将被注入到 Kitex 的代码生成器后端，以在特定位置插入指定的代码。

Kitex 会在 kitex_gen 目录下面为 IDL 里出现的每一个 service 定义生成一个对应的 package，在其中包含了 `NewClient`、`NewServer` 等 API。

extensions.yaml 文件提供的内容将会被应用到所有 service 定义对应的 package，其中 `extend_client`、`extend_server`、`extend_invoker` 字段分别对应 client.go 、 server.go 、invoker.go 的扩展。

### 应用场景

适用于统一定制 suite 场景。

企业用户对框架有统一的定制，即有一套固定的 option 配置，我们建议这些 option 封装在 suite 中，这样初始化时只需要配置一个 option。但对于内部的业务开发者来说依然需要配置这个 suite option，实际业务开发者并不需要关注这个配置，因为业务开发者不需要关注基础设施的能力。

字节内部会在生成代码中注入 bytedSuite，为了方便外部框架定制者也能这么使用，所以提供了这个配置化定制能力。当然，如果要进一步对业务开发者屏蔽这个细节，也可以对 Kitex 代码生成工具做一个封装，内置这个统一的配置。

## 示例

有这么一个 extensions.yaml 文件：

```yaml
---
dependencies:
  example.com/my/pkg: pkg
extend_client:
    import_paths:
      - example.com/my/pkg
    extend_option: 
      options = append(options, client.WithSuite(pkg.MyClientSuite()))
    extend_file: |-
        func Hello() {
          println("hello world")
        }
extend_server:
    import_paths:
      - example.com/my/pkg
    extend_option: 
      options = append(options, server.WithSuite(pkg.MyServerSuite()))
    extend_file: |-
        func Hello() {
          println("hello world")
        }
```

- **dependencies**：字段定义了在模板里可能会被使用到的包的列表，以及它们被引用的名字。

- **extend_client**：对 client.go 的扩展定制

  - **import_paths**：声明了对 client.go 文件注入的代码需要导入的包的列表，其内容必须是 **dependencies** 字段里声明过的。

  - **extend_option**：提供的代码片段会被注入到 `NewClient` 函数里，构造默认的选项的位置，可以在这个地方注入你自己定义的 suite 之类的选项。

  - **extend_file**：提供的代码片段将会被直接添加到 client.go 文件的末尾，因此可以在这里提供额外的函数、常量定义等。

- **extend_server** 的作用和 **extend_client** 是类似的，这里不再赘述。

一个应用了 extensions.yaml 的 IDL，生成的 client.go 可能是这样的（高亮的就是插入的部分）：

```go {linenos=table,hl_lines=[8,22,"53-55"]}
// Code generated by Kitex v0.4.2. DO NOT EDIT.

package demoservice

import (
	"context"
	demo "example.com/demo/kitex_gen/demo"
	pkg "example.com/my/pkg"
	client "github.com/cloudwego/kitex/client"
	callopt "github.com/cloudwego/kitex/client/callopt"
)

// Client is designed to provide IDL-compatible methods with call-option parameter for kitex framework.
type Client interface {
	Test(ctx context.Context, request *demo.DemoTestRequest, callOptions ...callopt.Option) (r *demo.DemoTestResponse, err error)
}

// NewClient creates a client for the service defined in IDL.
func NewClient(destService string, opts ...client.Option) (Client, error) {
	var options []client.Option
	options = append(options, client.WithDestService(destService))
	options = append(options, client.WithSuite(pkg.MyClientSuite()))

	options = append(options, opts...)

	kc, err := client.NewClient(serviceInfo(), options...)
	if err != nil {
		return nil, err
	}
	return &kDemoServiceClient{
		kClient: newServiceClient(kc),
	}, nil
}

// MustNewClient creates a client for the service defined in IDL. It panics if any error occurs.
func MustNewClient(destService string, opts ...client.Option) Client {
	kc, err := NewClient(destService, opts...)
	if err != nil {
		panic(err)
	}
	return kc
}

type kDemoServiceClient struct {
	*kClient
}

func (p *kDemoServiceClient) Test(ctx context.Context, request *demo.DemoTestRequest, callOptions ...callopt.Option) (r *demo.DemoTestResponse, err error) {
	ctx = client.NewCtxWithCallOptions(ctx, callOptions)
	return p.kClient.Test(ctx, request)
}

func Hello() {
	println("hello world")
}
```