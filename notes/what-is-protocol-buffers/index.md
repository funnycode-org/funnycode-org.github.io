# Protocol Buffers | Language Guide(proto2)


<!--more-->

## 一、前言

跟着 Protocol Buffers 官方学习

> **What are protocol buffers?** 
> Protocol buffers are Google's language-neutral, platform-neutral, extensible mechanism for serializing structured data – think XML, but smaller, faster, and simpler. You define how you want your data to be structured once, then you can use special generated source code to easily write and read your structured data to and from a variety of data streams and using a variety of languages.

Google 用来序列化结构数据，具有语言中立、平台中立和可扩展机制。比XML更小、更快、更简单。定义一次数据结构后，用特定语言的源码成功工具创建各个语言的代码。

## 二、proto2

### 2.1 定义消息类型

- 定义类型

```protobuf
message SearchRequest {
  required string query = 1;
  optional int32 page_number = 2;
  optional int32 result_per_page = 3;
}
```

- 可分配字段编号
  -  最小数字 1，最大数字 2^29 - 1, or 536,870,911
  - 范围为1到15的字段编号需要一个字节来编码，16到2047之间的字段号占用两个字节，应该为非常频繁出现的消息元素保留字段编号1到15
  - 不能使用数字19000到19999（`FieldDescriptor::kFirstReservedNumber`至`FieldDescriptor::kLastReservedNumber`）

- 指定字段规则
  - `required`: 必须存在值的字段
  - `optional`: 可选字段，能够存在值或者为空的字段
  - `repeated`: 数组类型，可以放空或者值. 数组值的顺序会被保护（不改变）

  > 使用 required 要小心，从 required 改 optional 可能会导致使用者出错。有些人甚至认为它的害处大于利处。

- 定义多个类型

一个 `.proto` 文件可以定义多个类，但是会导致消息膨胀，建议把不同类型的消息定义到不同的 `.proto` 文件

- 添加注解

```protobuf
/* SearchRequest represents a search query, with pagination options to
 * indicate which results to include in the response. */

message SearchRequest {
  required string query = 1;
  optional int32 page_number = 2;  // Which page number do we want?
  optional int32 result_per_page = 3;  // Number of results to return per page.
}
```

- 设置字段备用

```protobuf
message Foo {
  reserved 2, 15, 9 to 11;
  reserved "foo", "bar";
}
```

不建议直接删除字段，容易引起异常。可以把字段设置成备用。

-  通过 .proto 生成文件
  - C++，生成 .h 和 .cc 文件
  - Java，生成 .java 文件
  - Python，作为 *metaclass* 不懂嘻嘻
  - Go，生成 .pb.go 文件

### 2.2 标准值类型

直接上图

| .proto Type | Notes                                                        | C++ Type | Java Type  | Python Type[2]                       | Go Type  |
| :---------- | :----------------------------------------------------------- | :------- | :--------- | :----------------------------------- | :------- |
| double      |                                                              | double   | double     | float                                | *float64 |
| float       |                                                              | float    | float      | float                                | *float32 |
| int32       | Uses variable-length encoding. Inefficient for encoding negative numbers – if your field is likely to have negative values, use sint32 instead. | int32    | int        | int                                  | *int32   |
| int64       | Uses variable-length encoding. Inefficient for encoding negative numbers – if your field is likely to have negative values, use sint64 instead. | int64    | long       | int/long[3]                          | *int64   |
| uint32      | Uses variable-length encoding.                               | uint32   | int[1]     | int/long[3]                          | *uint32  |
| uint64      | Uses variable-length encoding.                               | uint64   | long[1]    | int/long[3]                          | *uint64  |
| sint32      | Uses variable-length encoding. Signed int value. These more efficiently encode negative numbers than regular int32s. | int32    | int        | int                                  | *int32   |
| sint64      | Uses variable-length encoding. Signed int value. These more efficiently encode negative numbers than regular int64s. | int64    | long       | int/long[3]                          | *int64   |
| fixed32     | Always four bytes. More efficient than uint32 if values are often greater than 228. | uint32   | int[1]     | int/long[3]                          | *uint32  |
| fixed64     | Always eight bytes. More efficient than uint64 if values are often greater than 256. | uint64   | long[1]    | int/long[3]                          | *uint64  |
| sfixed32    | Always four bytes.                                           | int32    | int        | int                                  | *int32   |
| sfixed64    | Always eight bytes.                                          | int64    | long       | int/long[3]                          | *int64   |
| bool        |                                                              | bool     | boolean    | bool                                 | *bool    |
| string      | A string must always contain UTF-8 encoded or 7-bit ASCII text. | string   | String     | unicode (Python 2) or str (Python 3) | *string  |
| bytes       | May contain any arbitrary sequence of bytes.                 | string   | ByteString | bytes                                | []byte   |

### 2.3 可选字段的默认值

```protobuf
optional int32 result_per_page = 3 [default = 10];
```

对于字符串，默认值为空字符串。对于字节，默认值为空字节字符串。对于布尔值，默认值为false。对于数字类型，默认值为零。对于枚举，默认值为枚举类型定义中列出的第一个值。

### 2.4 枚举

- 标准用法

```protobuf
message SearchRequest {
  required string query = 1;
  optional int32 page_number = 2;
  optional int32 result_per_page = 3 [default = 10];
  enum Corpus {
    UNIVERSAL = 0;
    WEB = 1;
    IMAGES = 2;
    LOCAL = 3;
    NEWS = 4;
    PRODUCTS = 5;
    VIDEO = 6;
  }
  optional Corpus corpus = 4 [default = UNIVERSAL];
}
```

给枚举取别名，加上 `option allow_alias = true`

```protobuf
enum EnumAllowingAlias {
  option allow_alias = true;
  UNKNOWN = 0;
  STARTED = 1;
  RUNNING = 1;
}
enum EnumNotAllowingAlias {
  UNKNOWN = 0;
  STARTED = 1;
  // RUNNING = 1;  // Uncommenting this line will cause a compile error inside Google and a warning message outside.
}

```

- 保留值

```protobuf
enum Foo {
  reserved 2, 15, 9 to 11, 40 to max;
  reserved "FOO", "BAR";
}
```

不能在同`reserved`一条语句中混合使用字段名和数字值

### 2.5 使用其它消息类型

- 同一个文件

```protobuf
message SearchResponse {
  repeated Result result = 1;
}

message Result {
  required string url = 1;
  optional string title = 2;
  repeated string snippets = 3;
}
```

- 不是同一个文件

```protobuf
import "myproject/other_protos.proto";
```

- 文件转移

```protobuf
// new.proto
// All definitions are moved here
```

```protobuf
// old.proto
// This is the proto that all clients are importing.
import public "new.proto";
import "other.proto";
```

```protobuf
// client.proto
import "old.proto";
// You use definitions from old.proto and new.proto, but not other.proto
```

### 2.6 嵌套类型

- 标准例子

```protobuf
message SearchResponse {
  message Result {
    required string url = 1;
    optional string title = 2;
    repeated string snippets = 3;
  }
  repeated Result result = 1;
}
```

- 其它类型嵌套

```protobuf
message SomeOtherMessage {
  optional SearchResponse.Result result = 1;
}
```

- 你可以多层嵌套

```protobuf
message Outer {       // Level 0
  message MiddleAA {  // Level 1
    message Inner {   // Level 2
      required int64 ival = 1;
      optional bool  booly = 2;
    }
  }
  message MiddleBB {  // Level 1
    message Inner {   // Level 2
      required int32 ival = 1;
      optional bool  booly = 2;
    }
  }
}
```

### 2.7 更新消息类型

- 不要更改任何现有字段的字段编号。
- 您添加的任何新字段应为`optional`或`repeated`。这意味着任何使用“旧”消息格式通过代码序列化的消息都可以被新生成的代码解析，因为它们不会丢失任何`required`元素。您应该为这些元素设置合理的[默认值](https://developers.google.com/protocol-buffers/docs/overview#optional)，以便新代码可以与旧代码生成的消息正确交互。同样，由新代码创建的消息可以由旧代码解析：旧的二进制文件在解析时只会忽略新字段。但是，未知字段不会被丢弃，并且如果消息在以后进行序列化，则未知字段也会与之一起进行序列化–因此，如果消息传递给新代码，则新字段仍然可用。
- 只要在更新的消息类型中不再使用该字段号，就可以删除不需要的字段。您可能想要重命名该字段，或者添加前缀“ OBSOLETE_”，或者将字段编号[保留为](https://developers.google.com/protocol-buffers/docs/overview#reserved)，以便将来的用户`.proto`不会意外地重用该编号。
- 只要类型和数字保持不变，就可以将不需要的字段转换为[扩展名](https://developers.google.com/protocol-buffers/docs/overview#extensions)，反之亦然。
- `int32`，`uint32`，`int64`，`uint64`，和`bool`都是兼容的-这意味着你可以在现场从这些类型到另一种改变不破坏forwards-或向后兼容。如果从对应的类型不适合的导线中解析出一个数字，则将获得与在C ++中将数字强制转换为该类型一样的效果（例如，如果将64位数字读取为int32，它将被截断为32位）。
- `sint32`并且`sint64`彼此兼容，但与其他整数类型*不*兼容。
- `string`并且`bytes`只要字节是有效的UTF-8即可兼容。
- `bytes`如果字节包含消息的编码版本，则嵌入式消息与之兼容。
- `fixed32`与兼容`sfixed32`，并`fixed64`用`sfixed64`。
- 对于`string`，`bytes`和消息字段，`optional`与兼容`repeated`。给定重复字段的序列化数据作为输入，如果期望该字段`optional`是原始类型字段，则期望该字段的客户端将采用最后一个输入值；如果是消息类型字段，则将合并所有输入元素。请注意，这对于数字类型（包括布尔值和枚举）通常**并不**安全。重复的数字类型字段可以以[打包](https://developers.google.com/protocol-buffers/docs/encoding#packed)格式序列化，当需要一个`optional`字段时，将无法正确解析该格式。
- 只要您记住从未通过网络发送默认值，通常就可以更改默认值。因此，如果程序收到未设置特定字段的消息，则该程序将看到该程序协议版本中定义的默认值。它不会看到在发送者的代码中定义的默认值。
- `enum`与兼容`int32`，`uint32`，`int64`，和`uint64`电线格式条款（请注意，如果他们不适合的值将被截断），但是要知道，客户端代码可以区别对待反序列化的消息时。值得注意的是，`enum`当对消息进行反序列化时，无法识别的值将被丢弃，这会使字段的`has..`访问器返回false，并且其getter返回`enum`定义中列出的第一个值；如果指定了默认值，则返回默认值。对于重复的枚举字段，所有无法识别的值将从列表中删除。但是，整数字段将始终保留其值。因此，在将整数升级为a时，`enum`在接收在线上的枚举枚举值方面需要非常小心。
- 在当前的Java和C ++实现中，当`enum`去除了无法识别的值时，它们会与其他未知字段一起存储。请注意，如果将此数据序列化然后由识别这些值的客户端重新解析，则可能导致奇怪的行为。对于可选字段，即使在反序列化原始消息后写入了新值，识别该值的客户端仍会读取旧值。在重复字段的情况下，旧值将出现在任何已识别的值和新添加的值之后，这意味着将不保留顺序。
- 将单个`optional`值更改为**新** 值的成员`oneof`是安全且二进制兼容的。如果您确定没有代码一次设置多个`optional`字段，那么将多个字段移动到新字段中`oneof`可能是安全的。将任何字段移到现有字段中`oneof`都是不安全的。
- 在`map<K, V>`和对应的`repeated`消息字段之间更改字段是二进制兼容的（有关消息布局和其他限制，请参见下面的[Maps](https://developers.google.com/protocol-buffers/docs/overview#maps)）。但是，更改的安全性取决于应用程序：在反序列化和重新序列化消息时，使用`repeated`字段定义的客户端将产生语义上相同的结果；但是，使用`map`字段定义的客户端可以对条目进行重新排序，并删除具有重复键的条目。

### 2.8 扩展名

- 标准用法

```protobuf
message Foo {
  // ...
  extensions 100 to 199;
}
```

```protobuf
extend Foo {
  optional int32 bar = 126;
}
```

实际例子待补充

- 嵌套扩展

```protobuf
message Baz {
  extend Foo {
    optional int32 bar = 126;
  }
  ...
}
```

实际例子待补充

### 2.9 一个

- 标准使用

```proto
message SampleMessage {
  oneof test_oneof {
     string name = 4;
     SubMessage sub_message = 9;
  }
}
```

- oneof特性
  - 设置oneof字段将自动清除oneof的所有其他成员。因此，如果您设置了多个字段，则只有您设置的*最后一个*字段仍具有值
  - 如果解析器在线路上遇到同一个对象的多个成员，则在解析的消息中仅使用最后看到的成员。
  - 其中之一不支持扩展。
  - 一个不能是`repeated`。
  - 反射API适用于其中一个字段。
  - 如果将oneof字段设置为默认值（例如将int32 oneof字段设置为0），则将设置该oneof字段的“大小写”，并且该值将在线路上序列化。
- 向后兼容问题

添加或删除字段之一时请多加注意。如果检查oneof的值返回`None`/ `NOT_SET`，则可能意味着尚未设置oneof或已将其设置为不同版本的oneof中的字段。由于无法知道导线上的未知字段是否是oneof的成员，因此无法分辨出差异。

### 2.10 Maps

### 2.11 Packages

### 2.12 定义服务

### 2.13 Options

### 2.14 生成你的类

## 三、proto3

## 四、参考

[https://developers.google.com/protocol-buffers/docs/overview](https://developers.google.com/protocol-buffers/docs/overview)
