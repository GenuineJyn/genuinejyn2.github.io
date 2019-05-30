# golang json反序列化number类型字符串到interface精度损失问题

## 1.json.Unmarshal操作number字符串可能导致精度损失问题
在[json规范](http://json.org)中对数字类型处理并没有区分整型和浮点型：
"A value can be a string in double quotes, or a number, or true or false or null, or an object or an array. These structures can be nested."
![json](https://github.com/GenuineJyn/genuinejyn.github.io/blob/master/pictures/json.png)

如果反序列化的时候指定明确的结构体和变量类型，反序列化不存在问题，但当把json解析成一个interface{}值的时候，golang标准库json把数字类型的统一反序列化成float64。

```
“To unmarshal JSON into an interface value, Unmarshal stores one of these in the interface value:
    bool, for JSON booleans
    float64, for JSON numbers
    string, for JSON strings
    []interface{}, for JSON arrays
    map[string]interface{}, for JSON objects
    nil for JSON null”   
``` 
 而对大整型来说，json转换成float64存在精度损失，再类型断言后转换成整型就存在问题。
 
## 2.浮点数精度

IEEE754二进制浮点数算术标准（IEEE754）被广泛使用，这个标准定义了表示浮点数的格式。golang也采用了，在《the golang programming language》的3.2节中描述了浮点数相关内容，可以详细看看，其中：

```
Go provides two sizes of floating-point numbers, float32 and float64. 
Their arit hmetic properties are governed by the IEEE 754 standard implemented by all moder n CPUs.
```
当时为了快速学习golang比较粗略的过了，没有去仔细回忆一下IEEE754，倒是牢牢记住了书中那个有趣的例子：

```
    var f float32 = 16777216    // 1 << 24    【标注点1】
    fmt.Println(f == f+1)       // "true"!
```
是时候来理解一下了。

在IEEE754中一个浮点数 (Value) 的表示其实可以这样表示：

![expression](https://github.com/GenuineJyn/genuinejyn.github.io/blob/master/pictures/expression.png)

![float](https://github.com/GenuineJyn/genuinejyn.github.io/blob/master/pictures/float.png)
以单精度浮点数float32为例，单精度浮点数使用32个bit来存储，如下图。
![float32](https://github.com/GenuineJyn/genuinejyn.github.io/blob/master/pictures/float32.png)

* S: 符号位， 0正1负
* 阶码：8位，以2为底的指数，阶码 = 阶码真值 + 127   //127的计算可以查看IEEE754  127 = 2^(8-1)-1
* 尾数：23位，采用规约形式的浮点数，隐含尾数最高位1的表示方法，实际24位，尾数真值 = 1 + 尾数

从上面的定义上来看，如果一个浮点数转换成IEEE754定义的格式，尾数部分超过23位就会出现精度问题。

```
单精度浮点数(float)总共用32位来表示浮点数，其中尾数用23位存储，加上小数点前有一位隐藏的1(IEEE754规约数表示法)，
`2^(23+1) = 16777216。因为 10^7 < 16777216 < 10^8，`
所以说10进制单精度浮点数的有效位数是7位。考虑到第7位可能截断，所以单精度最少有6位有效数字（最小尺寸）。 

同样地：双精度浮点数(double)总共用64位来表示浮点数，其中尾数用52位存储，
`2^(52+1) = 9007199254740992，10^16 < 9007199254740992 < 10^17，`所以10进制双精度浮点数的有效位数是16位。同样考虑到第16位可能的截断，最少15位。
```
这就解释了《the golang programming language》section3.2中的这句话：

```
A float32 provides approximately six decimal digits of precision, whereas a float64 provides about 15 digits;
```
 
顺便可以理解【标注点1】的代码了，看一下最后浮点数二进制表示就一目了然了，借助[工具](https://www.h-schmidt.net/FloatConverter/IEEE754.html)看一下。
![float1](https://github.com/GenuineJyn/genuinejyn.github.io/blob/master/pictures/float1.png)
![float2](https://github.com/GenuineJyn/genuinejyn.github.io/blob/master/pictures/float2.png)
如果看尾数的第24位：16777216为0， 16777217为1，因此存不下丢掉了(精度损失，也被称为“四舍五入”)。

# 3.窥探json相关实现代码

使用json.Decoder可以来解决这个问题，大致像这个样子：

```
body := []byte("{\"code\":0,\"message\":\"success\",\"response\":{\"id\":280123412341234123, \"idstr\": \"280123412341234123\"}}")
d := json.NewDecoder(bytes.NewBuffer(body))
d.UseNumber()
var res map[string]interface{}
if err := d.Decode(&res); err != nil {
    ......
}
```

继续来看看标准库中的这部分代码内容之前，简单说一下Unmarshal和Decoder，这两部底层都是使用了decodeState，decodeState定义如下：

```
// file： encoding/json/decode.go 
 269 // decodeState represents the state while decoding a JSON value.
 270 type decodeState struct {
 271     data         []byte
 272     off          int // read offset in data
 273     scan         scanner
 274     nextscan     scanner  // for calls to nextValue
 275     errorContext struct { // provides context for type errors
 276         Struct string
 277         Field  string
 278     }
 279     savedError            error
 280     useNumber             bool
 281     disallowUnknownFields bool
 282 }
```
 
decodeState并不开辟新内存也不拷贝内存，而是wrap指向内存的指针的形式并完成反序列化的过程，Unmarshal和Decoder的区别在于：
Unmarshal每次创建一个decodeState对象，并没有提供设置useNumber和disallowUnknownFields的参数或者闭包，而Decoder是组合了decodeState，这样便提供了设置useNumber和disallowUnknownFields的接口。

```
// file: encoding/json/stream.go 
 39 // DisallowUnknownFields causes the Decoder to return an error when the destination
 40 // is a struct and the input contains object keys which do not match any
 41 // non-ignored, exported fields in the destination.
 42 func (dec *Decoder) DisallowUnknownFields() { dec.d.disallowUnknownFields = true }
```

看了注释就明白这是个好东西，设置置位后，decoder反序列化的时候要完全匹配一个结构体的定义。

```
// file: encoding/json/decode.go
 753             } else if d.disallowUnknownFields {
 754                 d.saveError(fmt.Errorf("json: unknown field %q", key))
 755             }
 
```
UserNumber()告诉decoder把数字类型转换成json.Number而不是float64，相关代码如下。

```
// file: encoding/json/stream.go
 35 // UseNumber causes the Decoder to unmarshal a number into an interface{} as a
 36 // Number instead of as a float64.
 37 func (dec *Decoder) UseNumber() { dec.d.useNumber = true }

// file: encoding/json/decode.go
 845 // convertNumber converts the number literal s to a float64 or a Number
 846 // depending on the setting of d.useNumber.
 847 func (d *decodeState) convertNumber(s string) (interface{}, error) {
 848     if d.useNumber {
 849         return Number(s), nil
 850     }
 851     f, err := strconv.ParseFloat(s, 64)
 852     if err != nil {
 853         return nil, &UnmarshalTypeError{Value: "number " + s, Type: reflect.TypeOf(0.0), Offset: int64(d.off)}
 854     }
 855     return f, nil
 856 }
```

回归主题，继续窥探一下json.Number，底层类型原来是string，用字符串存数字啊，想用到具体类型再调用相关方法获得相应float64或int64类型的值。

```
 193 // A Number represents a JSON number literal.
 194 type Number string
 195
 196 // String returns the literal text of the number.
 197 func (n Number) String() string { return string(n) }
 198
 199 // Float64 returns the number as a float64.
 200 func (n Number) Float64() (float64, error) {
 201     return strconv.ParseFloat(string(n), 64)
 202 }
 203
 204 // Int64 returns the number as an int64.
 205 func (n Number) Int64() (int64, error) {
 206     return strconv.ParseInt(string(n), 10, 64)
 207 }
```

看到这里感觉Golang对JSON反序列化处理好像刚开始没想好然后不断再丰富功能，对代码而言，貌似并不是一个完美的解决方案，这可能是语言发展的快速发展的过程。

那最后一个问题来了，经过Decoder反序列出来interface{}可不可以被json.Marshal再次使用并且结果正确呢？  没有问题，可以正确的转化：`字符串整型 --Decoder.Decode()-->"字符串整型" --son.Marshal() -->字符串整型`
在Marshal的相关代码中也可以看到处理字符串的时候有特殊的考虑，考虑了numberType的情况：

```
// file: encoding/json/decode.go
 858 var numberType = reflect.TypeOf(Number(""))

// file: encoding/json/encode.go
 581 func stringEncoder(e *encodeState, v reflect.Value, opts encOpts) {
 582     if v.Type() == numberType {   //定义见上面代码👆👆
 583         numStr := v.String()
 584         // In Go1.5 the empty string encodes to "0", while this is not a valid number literal
 585         // we keep compatibility so check validity after this.
 586         if numStr == "" {
 587             numStr = "0" // Number's zero-val
 588         }
 589         if !isValidNumber(numStr) {
 590             e.error(fmt.Errorf("json: invalid number literal %q", numStr))
 591         }
 592         e.WriteString(numStr)
 593         return
 594     }
 595     if opts.quoted {
 596         sb, err := Marshal(v.String())
 597         if err != nil {
 598             e.error(err)
 599         }
 600         e.string(string(sb), opts.escapeHTML)
 601     } else {
 602         e.string(v.String(), opts.escapeHTML)
 603     }
 604 }
```

举例来验证一下：

```
package main

import (
    "bytes"
    "encoding/json"
    "fmt"
)

// Payload is the actual response to client.
type Payload struct {
    Code     int                    `json:"code"`
    Message  string                 `json:"message"`
    Response map[string]interface{} `json:"response,omitempty"`
}

func main() {
    body := []byte("{\"code\":0,\"message\":\"success\",\"response\":{\"id\":280123412341234123, \"idstr\": \"280123412341234123\"}}")

    d := json.NewDecoder(bytes.NewBuffer(body))
    d.UseNumber()

    var res map[string]interface{}
    if err := d.Decode(&res); err != nil {
        fmt.Printf("error: %s", err.Error())
        return
    }

    fmt.Printf("%#v\n", res)

    a, _ := json.Marshal(res)
    fmt.Printf("%s\n", a)

    code, _ := res["code"].(json.Number).Int64()
    j := &Payload{
        Code:     int(code),
        Message:  res["message"].(string),
        Response: res["response"].(map[string]interface{}),
    }

    b, _ := json.Marshal(j)
    fmt.Printf("%s\n", b)
}
```

输出内容如下：

```
map[string]interface {}{"code":"0", "message":"success", "response":map[string]interface {}{"id":"280123412341234123", "idstr":"280123412341234123"}}
{"code":0,"message":"success","response":{"id":280123412341234123,"idstr":"280123412341234123"}}
{"code":0,"message":"success","response":{"id":280123412341234123,"idstr":"280123412341234123"}}
```


# 参考

1. https://zh.wikipedia.org/wiki/IEEE_754
2. https://www.h-schmidt.net/FloatConverter/IEEE754.html
3. https://golang.org/pkg/encoding/json/
4. https://www.oreilly.com/library/view/the-go-programming/9780134190570/
