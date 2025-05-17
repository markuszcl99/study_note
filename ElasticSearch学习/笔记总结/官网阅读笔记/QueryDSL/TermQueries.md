# Term-level Queries

## 一、Term Query

Term Query 是 term 精准查询。需要注意的是，在进行 Term Query 的时候，要避免 text 类型的字段，因为 Elasticsearch 构建倒排索引时会改变 text 字段的值，即它会进行分词，这会导致我们在精准匹配 text 字段值时会比较困难。

> 如果想要查询 text 类型的值，需要使用 match query

### 1.1、term 查询的相关参数介绍

```json
{
	"query" : {
		"term": {
			"field": {
				"value": "xxx",
				"boost": 1.0
			}
		}
	}
}
```

- field: 即我们想要查询的字段
- value: 即我们想要查询的单词，只有在 term 完全匹配字段值（包含大小写和空格）时，才会返回文档。*ps: 这里我实验了一下，准确来说应该是精准匹配该字段在分词的最小 term 。*
- boost: 浮点类型，它用来对该查询条件进行提权或降权，在多个查询条件下，该字段对相关性算分的影响
- case_insensitive: 这个字段在 7.10.0 版本添加，控制该查询字段是否考虑大小写，true 的情况忽略大小写，false 的情况考虑大小写

### 1.2、RESTful API 示例

```json
PUT my_index_00001
{
  "mappings": {
    "properties": {
      "full-text":{
        "type": "text"
      }
    }
  }
}

PUT my_index_00001/_doc/1
{
  "full-text":"Quick Brown Foxes!"
}

// 使用 term 查询，会返回无结果，因为 term 查询不会对文本进行分词，而 text 类型的内容会被分词
POST my_index_00001/_search
{
  "query": {
    "term": {
      "full-text": {
        "value": "Quick", // Quick Brown Foxes! 不会返回数据
        "boost": 1.0,
        "case_insensitive": true // 大小写敏感控制， true 忽略大小写、false 考虑大小写
      }
    }
  }
}
```

## 二、Terms Query

在一次查询中指定多个 Term 时，只要文档匹配其中至少一个 Term，即可被检索出来。

Terms Query 是在 Term Query 的基础上，支持多值查询。

### 2.1、terms 查询的相关参数介绍

```json
{
  "query": {
    "terms": {
      "field": [value1,value2,]
      "boost": 1.0
    }
  }
}
```

- field: 即我们想要查询的字段
- value: 这里是一个数组
  - 默认情况，es 限制这个数组的大小是 65536，我们也可以通过使用 setting 中的 [`index.max_terms_count`](https://www.elastic.co/docs/reference/elasticsearch/index-settings/index-modules#index-max-terms-count)  来改变这个限制。
- boost: 浮点类型，它用来对该查询条件进行提权或降权，在多个查询条件下，该字段对相关性算分的影响

### 2.2、Terms Lookup 功能

我们也可以使用索引中已经存在的文档的字段作为查询的 Terms，来检索其他符合条件的文档，这个功能称为 terms lookup。这个功能在查询大量 Terms 时很有帮助。

#### 2.2.1、相关参数介绍

- index: 指定索引
- id: 指定文档 id
- path: 指定文档字段，将该字段的值作为检索的 terms value

### 2.3、RESTful API 示例

```json
PUT my_index_00002
{
  "mappings": {
    "properties": {
      "color": {
        "type": "keyword"
      }
    }
  }
}

PUT my_index_00002/_doc/1
{
  "color": ["blue","green","black"]
}

PUT my_index_00002/_doc/2
{
  "color": ["blue"]
}

// terms 示例
GET my_index_00002/_search
{
  "query": {
    "terms": {
      "color": [
        "blue",
        "white"
      ]
    }
  }
}

// terms lookup 示例
GET my_index_00002/_search?pretty
{
  "query": {
    "terms": {
      "color": {
        "index": "my_index_00002",
        "id": "2",
        "path": "color"
      }
    }
  }
}
```



## 三、、Terms Set Query

Terms Set Query 是一种用于匹配文档字段中多个值的至少一个或多个的高级查询方式，适用于文档中的字段是数组或集合的情况。它比普通的 terms 查询更灵活，可以基于多个值的交集进行过滤。

我们在查询过程中可以：

- 提供一个候选值集合
- Elasticsearch 会检查文档中指定字段的值，与这个集合的交集是否满足某种最小匹配条件
- 这个”最小匹配条件“可以是一个固定数字，或者从文档的另一个字段中动态获取

### 3.1、RESTful API 示例

```json
PUT job-candidates
{
  "mappings": {
    "properties": {
      "name":{
        "type": "keyword"
      },
      "programming_languages": {
        "type": "keyword"
      },
      "required_matches": {
        "type": "long"
      }
    }
  }
}

// refresh 表示文档创建后可以被立即检索
PUT /job-candidates/_doc/1?refresh
{
  "name": "Jane Smith",
  "programming_languages": [ "c++", "java" ],
  "required_matches": 2
}

PUT /job-candidates/_doc/2?refresh
{
  "name": "Jason Response",
  "programming_languages": [ "java", "php" ],
  "required_matches": 2
}

GET /job-candidates/_search
{
  "query": {
    "terms_set": {
      "programming_languages": {
        "terms": [ "c++", "java", "php" ],
        "minimum_should_match_field": "required_matches" // 根据文档的字段动态决定最小匹配数
      }
    }
  }
}

GET /job-candidates/_search
{
  "query": {
    "terms_set": {
      "programming_languages": {
        "terms": [ "c++", "java", "php" ],
        "minimum_should_match": 2 // 固定值
      }
    }
  }
}

GET /job-candidates/_search
{
  "query": {
    "terms_set": {
      "programming_languages": {
        "terms": [
          "c++",
          "java",
          "php"
        ],
        "minimum_should_match_script": {
          // params.num_terms 就是传入的 terms 的数量
          "source": "Math.min(params.num_terms, doc['required_matches'].value)" // 脚本动态指定
        }
      }
    }
  }
}
```

## 四、Exists Query

Exists Query(存在性查询)，用于检索**字段存在**的文档。通俗地说：它就是再问"这个字段在文档里有没有值？"

文档的某个字段可能因为多种原因没有被索引，原因可能如下：

- 该字段的可能是 null 或者 []，因为 Elasticsearch 不会为 null 或空数组的字段建立索引。也就是说，虽然字段在 _source 中存在，但在索引中根本没有它的信息，exists 查询时就不会命中
- 在 mapping 设置时，针对该字段做了如下操作：
  - index: false【关闭倒排索引】 且 doc_values: false【关闭列式存储】，这种字段也不会被索引
  - ignore_above: xxx(number) ，该字段是 keyword 类型字段中常见的参数，用来忽略过长的字符串，如果超出限制，也不会被索引
  - ignore_malformed: xxx(bool)，字段的值格式错误，但映射中设置了 ignore_malformed，也就是当我们传入了格式错误的字段值（比如给整数字段传了字符串），如果该设置为true，Elasticsearch 会忽略该字段的索引，而不是抛出错误。

### 4.1、RESTful API 示例

```json
PUT my_index_00003
{
  "mappings": {
    "properties": {
      "username": {
        "type": "keyword"
      },
      "age": {
        "type": "long",
        "index": false,
        "doc_values": false
      }
    }
  }
}

PUT my_index_00003/_doc/1
{
  "username": "markuszhang",
  "age": 26
}
PUT my_index_00003/_doc/2
{
  "username": "luna",
  "age": 25
}

// 查不到值，因为 age 字段关闭了 index 和 doc_values
GET my_index_00003/_search
{
  "query": {
    "term": {
      "age": {
        "value": 25
      }
    }
  }
}

GET my_index_00003/_search
{
  "query": {
    "bool": {
      "must_not": [
        {
          "exists": {
            "field": "age"
          }
        }
      ]
    }
  }
}

GET my_index_00003/_search
{
  "query": {
    "exists": {
      "field": "username"
    }
  }
}
```

## 五、Fuzzy Query

Fuzzy Query 是用来处理拼写错误或不精确匹配的查询类型。它基于 Levenshtein 编辑距离，允许在查询字符串和被匹配词之间存在一定程度的差异（插入、删除、替换、转置字符）。例如：查询词是 "roam"，它可以模糊匹配 "foam","roams","room" 等。

### 5.1、相关参数介绍

```json
{
  "query": {
    "fuzzy": {
      "field": {
        "value": "xxx",
        "fuzziness": "AUTO",
        "max_expansions": 50,
        "prefix_length": 0,
        "transpositions": true,
        "rewrite": "constant_score"
      }
    }
  }
}
```

- field: 需要查询的字段
- value: 需要查询的 term
- fuzziness: 允许匹配的最大编辑距离
- max_expansions: 最大扩展词数量的限制（根据 fuzziness 生成若干个变体词去倒排索引进行查询，这个参数就是用来限制变体词的数量，默认50）
- prefix_length: 带前缀长度限制的模糊匹配，设置为0 表示不要求前缀必须精确匹配
- transpositions: 用来控制是否允许两个相邻字符的位置调换，也算作一次编辑操作
- rewrite: 当使用 fuzzy 查询时，它会生成多个可能匹配的 term，然后使用布尔查询来组合这些 term 进行匹配，该参数决定了这个布尔查询的类型和执行方式，会影响查询性能、精度、匹配方式
  - 可选的模式如下：
    - constant_score: 默认方式，把所有的匹配的 term 作为 filter，不计算得分
    - scoring_boolean: 用 should 条件组合匹配 term，计算得分
    - constant_score_boolean: 和 scoring_boolean 类似，但不计算得分，性能更好
    - top_terms_N: 只使得得分前 N 个词项参与查询
    - top_terms_boost_N: 和 top_terms_N 类似，但使用原始分值作为 boost
    - top_terms_blended_freqs_N: 综合考虑词频，得分更精细（Lucene 支持）

### 5.2、RESTful API 示例

```json
PUT my_index_00004
{
  "mappings": {
    "properties": {
      "username": {
        "type": "text"
      }
    }
  }
}

PUT my_index_00004/_doc/1
{
  "username": "markuszhang"
}

GET my_index_00004/_search
{
  "query": {
    "fuzzy": {
      "username": {
        "value": "makuszhang",
        "fuzziness": 1,
        "prefix_length": 1,
        "rewrite": "constant_score"
      }
    }
  }
}
```

## 六、IDs Query

基于文档 id 进行查询

```json
GET my_index_00003/_search
{
  "query": {
    "ids": {
      "values": [1,2]
    }
  }
}
```

## 七、Prefix Query

前缀查询，即指定前缀，检索符合条件的文档

### 7.1、相关参数介绍

```json
{
  "query": {
    "prefix": {
      "field": {
        "value": "prefix"
      }
    }
  }
}
```

- field: 需要检索的字段
- value: 需要检索的起始字符
- rewrite: 同 Fuzzy Query 中的参数
- case_insensitive: 在 7.10.0 版本添加，是否考虑大小写

### 7.2、RESTful API 示例

```json

GET my_index_00004/_search
{
  "query": {
    "prefix": {
      "username": {
        "value": "Markus",
        "case_insensitive": true
      }
    }
  }
}
```

## 八、Range Query

范围查询，指定 terms 的范围，返回符合条件的文档

### 8.1、相关参数介绍

```json
{
  "query": {
    "range": {
      "age": {
        "gte": 10,
        "lte": 20,
        "boost": 2.0
      }
    }
  }
}
```

- gt: 大于
- gte: 大于等于
- lt: 小于
- lte: 小于等于
- format: 将value值转换为某种格式，es 默认使用的是 日期格式
- relation: 如果字段是一个 range 类型，则这块会有几个查询模式
  - INTERSECTS（默认），即有交集就算符合条件
  - CONTAINS，文档范围完全包含查询范围
  - WITHIN, 文档范围完全被查询范围包含
- time_zone: 时区
- boost: 对此查询条件做加降权

### 8.2、RESTful API 示例

```json
PUT my_index_00005
{
  "mappings": {
    "properties": {
      "username": {
        "type": "keyword"
      },
      "age": {
        "type": "long"
      }
    }
  }
}

PUT my_index_00005/_doc/1
{
  "username": "markuszhang",
  "age": 26
}
PUT my_index_00005/_doc/2
{
  "username": "luna",
  "age": 25
}

GET my_index_00005/_search
{
  "query": {
    "range": {
      "age": {
        "gte": 26
      }
    }
  }
}
```

## 九、Regexp Query

正则表达式匹配查询，返回符合正则表达式的文档

### 9.1、相关参数介绍

```json
{
  "query": {
    "regexp": {
      "field": {
        "value": "k.*y",
        "flags": "ALL",
        "case_insensitive": true,
        "max_determinized_states": 10000,
        "rewrite": "constant_score_blended"
      }
    }
  }
}
```

- field: 需要检索的字段
- value: 正则表达式，Elasticsearch 默认限制 1000 characters，我们可以通过 index.max_regex_length setting 设置
  - 正则查询的性能受正则表达式影响，为提升性能，尽量避免直接使用通配符，例如 .*, .*?+，没有前缀或者后缀。
- flags: 可以启用（或组合启用）不同的 **正则表达式特性**
- case_insensitive: 是否考虑大小写
- max_determinized_states: 自动机状态数，默认10000，避免出现性能问题
- rewrite: 同 Fuzzy Query 中的参数

### 9.2、RESTful API 示例

```json
GET my_index_00005/_search
{
  "query": {
    "regexp": {
      "username": "markus.*"
    }
  }
}
```

## 十、Wildcard Query

通配符查询，与 regexp query相比，`wildcard` 查询语法简单、性能更高，适用于常见的通配搜索。

### 10.1、相关参数介绍

```json

{
  "query": {
    "wildcard": {
      "field": {
        "value": "ki*y",
        "boost": 1.0,
        "rewrite": "constant_score_blended"
      }
    }
  }
}
```

- field: 要查询的字段
- boost: 相关性算法加降权
- case_insensitive: 大小写敏感
- rewrite: 同 Fuzzy Query 中的参数
- value: 带通配符的 terms
  - ? 表示匹配任意单个字符串
  - \* 表示匹配0或任意个字符，包含空字符
  - 需要注意，避免使用 * or ? 作为 term 的起始字符，这会增加查询匹配 term 的次数和降低查询性能
- wildcard: 和 value 对应，是 value 参数的别名，如果 value 和 wildcard 同时被指定，会选择 request body 中最后一个参数所指定的 term 进行查询

> 使用 * 的通配符查询可能会消耗大量资源，特别是前缀是 * 的查询。为了提高查询性能，应尽量减少使用这种方式，并考虑其他替代方案，比如使用 n-gram 分词器。虽然 n-gram 能让搜索更高效，但会导致索引变大。为了更好的性能和准确性，建议将 wildcard 查询和其他查询（如 match 或 bool）组合使用，先缩小候选结果，再进行通配匹配

### 10.2、RESTful API 示例

```json
GET my_index_00005/_search
{
  "query": {
    "wildcard": {
      "username": "markus*hang"
    }
  }
}
```

