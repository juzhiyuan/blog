# React form with JSON Schema

## JSON Schema

`JSON Schema` 是一个用于校验 JSON 数据格式的工具，通过约定的 `JSON Schema` 校验被传入的 JSON 内容。

场景：客户端通过 API 传递 JSON，服务端根据约定的 `JSON Schema` 进行校验是否合法。

注意， `JSON Schema` 的表现也是 JSON 格式，如下示例：

``` json
{
	"type": "object",
	"properties": {
		"firstName": {
      "type": "string",
      "minLength": 3
		}
	}
}
```

针对上述 `JSON Schema` ，下方 JSON 是合法的：

``` json
{
  "firstName": "abc123"
}
```

有关更多关于 `JSON Schema` 的内容，请访问：[https://json-schema.org/understanding-json-schema/index.html](https://json-schema.org/understanding-json-schema/index.html)

## 通过 JSON Schema 构建表单

在我们的开源网关项目 [Apache APISIX](https://github.com/apache/incubator-apisix) 中，配套的 Dashboard 需要能够展示不同插件并供用户操作，以插件 `[jwt-auth](https://github.com/apache/incubator-apisix/blob/master/apisix/plugins/jwt-auth.lua#L31)` 为例，其 schema 为：

``` json
{
  "type": "object",
  "properties": {
    "key": {
      "type": "string"
    },
    "secret": {
      "type": "string"
    },
    "algorithm": {
      "type": "string",
      "enum": {
        "HS256",
        "HS384",
        "HS512",
        "RS256",
        "ES256"
      }
    },
    "exp": {
			"type": "integer",
		  "minimum": 1
		},
		"base64_secret": {
			"type": "boolean",
			"default": false
		}
  }
}
```

上述 schema 表达的含义是：存在 key、secret、algorithm、exp、base64_secret 共 5 个可选字段：

1. key、secret 为字符串，表现为**输入框**；
2. algorithm 为字符串枚举，表现为**下拉选择器**；
3. exp 需为整数，且最小值为1，表现为**输入框**；
4. base64_secret 为布尔值，默认为 false，表现为 **Checkbox**；

我们的最终目标是通过上方描述、构建Form 表单，由于 schema 非常灵活，为每一个插件的 schema 构建独立表单组件成本极高，于是在 Dashboard 中我对 API 响应的 schema 进行解析，通过多种条件判断、不断遍历最终构建出 Form [表单组件](https://github.com/apache/incubator-apisix-dashboard/blob/c616a34d059997e78683df8d3da0f6f690fccce5/src/components/PluginForm/PluginForm.tsx)。

随着插件逐渐增多，schema 情况也越来越多，这导致现有的**插件表单组件**所包含的条件（情况）无法完整覆盖插件所包含的格式，从而无法构建正确的表单 UI，对用户而言是不可用的。直到发现了 `[react-jsonschema-form](https://github.com/rjsf-team/react-jsonschema-form/)` 这个开源的、根据 JSON Schema 生成 UI 的库：

![images/Untitled.gif](images/Untitled.gif)

来自 react-jsonschema-form

访问 [https://rjsf-team.github.io/react-jsonschema-form](https://rjsf-team.github.io/react-jsonschema-form) 以查看更多示例。

## react-jsonschema-form

经过翻阅该项目[源码](https://github.com/rjsf-team/react-jsonschema-form/blob/master/packages/core/src/components/Form.js)，其实现思路同样是通过匹配各种条件与遍历，构建 UI。它按照 `JSON Schema` 规范进行实现，因此构建出的效果非常好！但由于它具有[多个版本标准](https://json-schema.org/specification.html)，所以对于旧有 schema，在传入该组件前要进行转换，以便构建正确的 UI。
