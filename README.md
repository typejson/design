# typejson

>  为达到统一接口的目的,为经过允许不要自行实现 typejson

typejson 整体设计有2大特点:

1. 基于静态类型语言
2. 要求json定义时消除 undefined null  等bug之源
3. 扁平式声明,避免产生类似 json schema 式的不易于阅读的结构
4. 多语言版本实现统一
5. http接口要基于实现而不是基于文档



**基于静态类型语言**

基于静态类型语言的意思是 typejson 只实现静态类型语言版本,动态类型编程语言不会有 typejson的实现.

各个语言的 typejson 要尽量实现生成客户端语言的接口/类型定义.

例如必须生成 TypeScript 的 interface,便于web前端基于后端语言的类型用typejson 生成 TypeScript interface 作为前后端交互数据的接口.



因为基于静态类型语言,所以不要让使用者通过数据结构配置,而是通过函数语法配置.最终的数据结构只是用通用协议格式

```ts
// 不要这样设计接口
tj.check(v, {
  "age":{type:"number", min:18}
})
// 而是应该封装函数让使用者更方便的写出代码
tj.check(v,[
	tj.number(age).min(18)
])
```

在语言中通过函数调用配置 typejson rule 可以利用类型系统控制不出现`_.number(age).notEmpty()` 这种错误

> notEmpty 只能给 string 使用

**消除 undefined null **

typejson 中没有 json schema 的 required ,而是通过 notEmpty min 等方法代替,还会出现利用多级结构代替 undefined 的情况 [参考notEmpty](#notEmpty).目的是消除代码中的 undefined null 等容易导出bug的设计.让 json 接口更加的文档易于维护和理解.



**扁平式声明**

无论结构多复杂 typejson阅读起来都会很方便

```j
{
  "name": {type:"string"},
  "age":{type:"number"},
  "children": {type:"array"},
  "children.*": {type:"object"},
  "children.*.name": {type:"string"}
  "children.*.age": {type:"number"}
}
```



**多语言版本实现统一**

在此声明,为了保持多语言实现统一,不同编程语言库的接口一致.不允许自行实现 typejson ,必须通过  github.com/typejson/语言  实现, [nimo](https://github.com/nimoc) 会管理 typejson 的各个编程语言接口设计. 如果你硬要实现与typejson接口不一致的库,请不要叫 typejson 以免出现使用者误以为你的实现与其他版本typejson实现一致.

比如typejson在实现一个 equal 时候会要求有

```j
{ equalString: "nimo" }
{ equalInt: 18 }
{ equalFloat: 22.5 }
{ equalBoolean: true }
```

可能你会觉得明明实现一个 `equal` 就可以了

```js
{ equal: "nimo" }
{ equal: 18 }
{ equal: 22.5 }
{ equal: true }
```

但  [nimo](https://github.com/nimoc) 考虑的是要做到在静态类型语言中的 typejson尽量消除泛型

```ts
// 类型严格是好的设计
var _ = trule
tj.check(v, {
	trule.when("anonymityName", {
  	"name": {equalString: "anonymity"}
	}),
  trule.when("useCreditCard", {
  	"creditCard": {equalBoolean: false}
	}),
})
```



**http接口要基于实现而不是基于文档**

参考 [生成客户端接口类型代码](#生成客户端接口类型代码)

## 规则设计

### type

> 类型

```js
{ type:"string" }
{ type:"number" }
{ type:"boolean" }
{ type:"array" }
{	type:"object"}
```

typejson定义的类型在 类型注解性质的静态语言比如 TypeScript 中会做自动修复,例如定义类型是 string ,而原始数据是number,则会自动转换为字符串.强静态类型语言则不需要考虑这个问题.

### pattern

> 正则表达式匹配后通过

```js
{pattern:"\d{11}"} // 手机号码
{pattern:"^[a-z0-9]*$"} // 只允许小写字母和数字
```

`pattern`匹配结果为 `true` 时候才会通过



### banPattern

> 正则表达式匹配后不通过

与 `pattern` 相反

```js
{pattern:"^[0-9]*$"} // 不允许纯数字
```

`pattern`匹配结果为 `false` 时候才会通过



### enum

> 只允许指定选项



```js
{enum: ["male", "female"]} // 只允许男性女性
```

不在 `enum` 的值会校验失败

### notEmpty

> 不允许为空字符串

```js
{notEmpty:true}
```

typejson 取消了 required 的设计,只支持 `notEmpty`验证 `string` 是否必填

如果要验证其他类型必填则需做如下定义

```js
{
  'quantity': {
    type: "number",
    // 通过 min 1 控制商品购买数量这种不可能为0的数据必填
    min:1,
    name: "商品购买数量",
  },
  'numberOfChildren.enabled': {
		type:"boolean",
    note:"填写子女数量",
  },
  'numberOfChildren.value': {
    type:"number",
    name:"子女数量",
  },
	'emailList': {
    type: "array",
    minlen: 1, // 通过 minlen 控制 array 必填
    name:"邮箱地址列表",
  },
  'emailList.*':{type:"string"}
}
```

### min

> number 最小值

```js
{type:"number",min:18} // 值必须大于等于18,例如为17时不通过验证,18则通过验证
```

### max

> number 最大值

```js
{type:"number",min:65} // 值必须小于等于65,例如为66时不通过验证,65则通过验证
```

> 不实现 `minmax(18,65)` 这种语法糖,为了避免出现接口重复导致的使用者重复定义了 min max

### minlen

> array string 最小长度,不设置则不限制最小长度

```js
{type:"array", minlen:1} // 数组至少要有一个数组项
```

```js
var rule = {
  	"emailList": {type:"array", minlen:1},
	  "emailList.*"{type:"string", name:"email"},
}
tj.chekc(
  {
    emailList:[],
  },
  rule
) // 校验不通过
tj.chekc(
  {
    emailList:["x@nimoc.io"],
  },
  rule
) // 校验通过
```

```js
var rule = {
  	"name": {type:"string", minlen:2},
}
tj.chekc(
  {
    name:"n",
  },
  rule
) // 校验不通过
tj.chekc(
  {
		name:"nimo",
  },
  rule
) // 校验通过
```



### maxlen

> array string 最大长度,不设置则不限制最大长度

与 `minlen` 功能类似,只是一个是限制最小,一个是限制最大



> 不实现 `minlmaxlen(18,65)` 这种语法糖,为了避免出现接口重复导致的使用者重复定义了 minlen maxlen

### when

> 当满足条件时才开始校验

举一个选择使用信用卡才校验信用卡信息的例子:

```js
// json
{
  "@when": {
    "chooseUseCreditCard": {
      "useCreditCard": {equalBoolean: true},
    }
  },
	"useCreditCard": {type:"boolean", name:"是否使用信用卡"}
  "creditCard": {type:"object"},
  "creditCard.number": {
    type: "string", notEmpty:true,
    when: "chooseUseCreditCard",
  }
  "creditCard.userName":{
    type:"string", notEmpty:true,
    when: "chooseUseCreditCard",
  }
}
```

```ts
// TypeScript
// 定义常量是为了出现多处写死 key 导致修改时可能漏掉修改的问题
const chooseUseCreditCard = "chooseUseCreditCard"
const useCreditCard = "useCreditCard"
tj.check(v, [
	tj.when(whenChooseUseCreditCard, {
    // 这在 TypeScript 中等同于 "useCreditCard":{}
		[useCreditCard]: {equalBoolean: true}
  })
])
```



## 生成客户端接口类型代码

typejson 传递了一种思想,前后端数据约定时候,应该以后端的接口/类型作为依据.

>  http接口要基于实现而不是基于文档

比如先别写在线文档或者本地文档,然后前后端基于文档开发是不好的.

应该通过讨论,在后端语言中指定接口类型,然后使用 typejson 基于接口和类型生成客户端的接口类型和http接口文档.这样可以避免实现改了,但是文档忘记改了或者文档和接口不一致的问题.



以 go作为后端语言为例:



```go
package "eventDTO"
import (
  "github.com/typejson/go"
)
type CreateReq struct {
  Title string `json:"title"`
  EmailList []string `json:"emailList"`
}
// JsonTag 会有单元测试,用来验证返回的 tag 与 CreateReq 中的 struct tag json 是否一致
func (CreateReq) JsonTag() (tag struct {
  Title string
  EmailList string
}) {
  tag.Title = "title"
  tag.EmailList = "emailList"
	return
}
func (v CreateReq) Typejson() (tj.Rules) {
	tag := v.JsonTag()
  return tj.Rules{
    tj.String(tag.Title).NotEmpty()
    tj.Array(tag.EmailList).min(1)
    tj.StringItem(tag.EmailList).notEmpty()
  }
}
```

```go
package "main"
import (
  "github.com/typejson/go"
  "somePath/dto/event"
)
func main(){
	tj.Json(&eventDTO.CreateReq{})
	tj.TypeScriptInterface("iCreateReq", "typejsonCreateReq", &eventDTO.CreateReq{})
}
```



`  tj.Json()` 会返回字符串

```json
{
  "title": {"type":"string", "notEmpty":true},
  "emailList": {"type": "array", "min": 1},
  "emailList.*": {"type":"string", notEmpty: true},
}
```

基于这个 json 就可以生成文档了

`tj.TypeScriptInterface()` 会返回字符串

```ts
interface iCreateReq {
		title:string
    emailList: string[]
}
const iCreateReqJSONTag = {
  title: "title",
  emailList: "emailList",
}
function typejsonCreateReq (v:iEventDTOCreateReq) {
	  const tag = iCreateReqJSONTag
    tj.string(tag.title).notEmpty()
    tj.array(tag.emailList).min(1)
    tj.stringItem(tag.emailList).notEmpty()
}
```



前端在基于生成的 TypeScript 接口进行开发校验.当需要修改校验规则和接口时,通过修改 go 的 struct 后重新生成 TypeScript 接口.

这就是 typejson 提供的

> http接口要基于实现而不是基于文档
