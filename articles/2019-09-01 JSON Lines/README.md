---
title: JSON Lines
---

[JSON Lines](http://jsonlines.org/) 是一种文本格式，适用于存储大量结构相似的嵌套数据、在协程之间传递信息等。

例子如下：

```json-ld
{"name": "Gilbert", "wins": [["straight", "7♣"], ["one pair", "10♥"]]}
{"name": "Alexa", "wins": [["two pair", "4♠"], ["two pair", "9♠"]]}
{"name": "May", "wins": []}
{"name": "Deloise", "wins": [["three of a kind", "5♣"]]}
```

它有如下特点：

1. 每一行都是完整、合法的 JSON 值；采用 \n 或 \r\n 作为行分隔符；
2. 采用 UTF-8 编码；
3. 使用 jsonl 作为文件扩展名；建议使用 gzip 或 bzip2 压缩并生成 .jsonl.gz 或 .jsonl.bz2 文件，以便节省空间。

## 对比

```
// CSV

id,father,mother,children
1,Mark,Charlotte,1
2,John,Ann,3
3,Bob,Monika,2
```

```json
// JSON

[
   {
      "id": 1,
      "father": "Mark",
      "mother": "Charlotte",
      "children": 1
   },
   {
      "id": 2,
      "father": "John",
      "mother": "Ann",
      "children": 3
   },
   {
      "id": 3,
      "father": "Bob",
      "mother": "Monika",
      "children": 2
   }
]
```

对于同样的数据内容，CSV 比 JSON 更简洁，但却不易读。此外，CSV 无法表示嵌套数据，例如一个家庭下全部成员的名字，但 JSON 却可以很容易地表示出来：

```json
{
  "familyMembers": ["JuZhiyuan", "JuShouChang"]
}
```

既然 JSON 如此灵活，那为何还需要 JSON Lines 呢？

考虑如下场景：一个大小为 1GB 的 JSON 文件，当我们需要读取/写入内容时，需要读取整个文件、存储至内存并将其解析、操作，这是不可取的。

若采用 JSON Lines 保存该文件，则操作数据时，我们无需读取整个文件后再解析、操作，而可以根据 JSON Lines 文件中每一行便为一个 JSON 值 的特性，边读取边解析、操作。例如：在插入 JSON 值时，我们只需要 append 值到文件中即可。

因此，操作 JSON Lines 文件时，只需要：

1. 读取一行值；
2. 将值解析为 JSON；
3. 重复1、2步骤。

那么如何将 JSON Lines 转换为 JSON 格式呢？下方代码为 JavaScript 示例：

```json
const jsonLinesString = `{"name": "Gilbert", "wins": [["straight", "7♣"], ["one pair", "10♥"]]}
{"name": "Alexa", "wins": [["two pair", "4♠"], ["two pair", "9♠"]]}
{"name": "May", "wins": []}
{"name": "Deloise", "wins": [["three of a kind", "5♣"]]}`;

const jsonLines = jsonLinesString.split(/\n/);
const jsonString = "[" + jsonLines.join(",") + "]";
const jsonValue = JSON.parse(jsonString);

console.log(jsonValue);
```

## 参考
1. [JSON Lines](http://jsonlines.org/examples/)
2. [JSON Lines format: Why jsonl is better than a regular JSON for web scraping](https://hackernoon.com/json-lines-format-76353b4e588d)
