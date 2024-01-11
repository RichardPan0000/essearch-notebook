#  Query DSL





## 1、简介

es基于JSON的DSL语言（Domain Specific Language, 领域特定语言）来定义查询。查询DSL主要基于两种类型子句组成。

### 1.1叶查询子句

​	叶查询子句在特定字段中查找特定值，例如match、term或range查询。 这些查询本身也是可以用它们自己。（也就是嵌套使用的情况）

### 1.2 复合查询子句

​     复合查询子句包装其他叶查询或复合查询，用于以逻辑方式组合多个查询（例如 bool 或 dis_max 查询），或更改它们的行为（例如constant_score 查询）。查询子句的行为有所不同，具体取决于它们是在查询上下文还是过滤器上下文中使用。

### 1.3 允许昂贵的查询

​      某些类型的查询由于其实现方式通常会**执行缓慢**，这可能会影响集群的稳定性。 这些查询可以分类如下：

[`search.allow_expensive_queries`](https://www.elastic.co/guide/en/elasticsearch/reference/8.7/query-dsl.html#query-dsl-allow-expensive-queries) 开关必须要打开，才能执行一下的某些查询。否则es默认不会执行下面花销昂贵的查询。

- 需要进行**线性扫描来识别匹配的查询**：

  - [`script` queries](https://www.elastic.co/guide/en/elasticsearch/reference/8.7/query-dsl-script-query.html) 脚本查询

  - queries on [numeric](https://www.elastic.co/guide/en/elasticsearch/reference/8.7/number.html), [date](https://www.elastic.co/guide/en/elasticsearch/reference/8.7/date.html), [boolean](https://www.elastic.co/guide/en/elasticsearch/reference/8.7/boolean.html), [ip](https://www.elastic.co/guide/en/elasticsearch/reference/8.7/ip.html), [geo_point](https://www.elastic.co/guide/en/elasticsearch/reference/8.7/geo-point.html) or [keyword](https://www.elastic.co/guide/en/elasticsearch/reference/8.7/keyword.html) fields that are not indexed but have [doc values](https://www.elastic.co/guide/en/elasticsearch/reference/8.7/doc-values.html) enabled  查询数字、日期、布尔、ip ，keyword的字段，并没有索引，但是又有值的字段。

- 前期成本比较高的查询：

  - [`fuzzy` queries](https://www.elastic.co/guide/en/elasticsearch/reference/8.7/query-dsl-fuzzy-query.html) (except on [`wildcard`](https://www.elastic.co/guide/en/elasticsearch/reference/8.7/keyword.html#wildcard-field-type) fields)  模糊查询

  - [`regexp` queries](https://www.elastic.co/guide/en/elasticsearch/reference/8.7/query-dsl-regexp-query.html) (except on [`wildcard`](https://www.elastic.co/guide/en/elasticsearch/reference/8.7/keyword.html#wildcard-field-type) fields)  正则匹配查询

  - [`prefix` queries](https://www.elastic.co/guide/en/elasticsearch/reference/8.7/query-dsl-prefix-query.html) (except on [`wildcard`](https://www.elastic.co/guide/en/elasticsearch/reference/8.7/keyword.html#wildcard-field-type) fields or those without [`index_prefixes`](https://www.elastic.co/guide/en/elasticsearch/reference/8.7/index-prefixes.html))  前缀查询

  - [`wildcard` queries](https://www.elastic.co/guide/en/elasticsearch/reference/8.7/query-dsl-wildcard-query.html) (except on [`wildcard`](https://www.elastic.co/guide/en/elasticsearch/reference/8.7/keyword.html#wildcard-field-type) fields)  通配符匹配

  - [`range` queries](https://www.elastic.co/guide/en/elasticsearch/reference/8.7/query-dsl-range-query.html) on [`text`](https://www.elastic.co/guide/en/elasticsearch/reference/8.7/text.html) and [`keyword`](https://www.elastic.co/guide/en/elasticsearch/reference/8.7/keyword.html) fields  范围查询

- Joining queries 联合查询（---nested query 和has_child has_parent这两种查询都是花销比较大的查询）

- Queries that maybe have high per-document cost：  ---每个文档成本比较高的查询

  - [`script_score` queries](https://www.elastic.co/guide/en/elasticsearch/reference/8.7/query-dsl-script-score-query.html)  --脚本score 的查询（用特定的脚本去给返回的doc计算得分）----这种一般在想要自定义得分的时候用，有时候有些得分func也是花销比较大的。

  - [`percolate` queries](https://www.elastic.co/guide/en/elasticsearch/reference/8.7/query-dsl-percolate-query.html) 渗透查询--

    渗透查询可用于匹配存储在索引中的查询。 渗透查询本身包含将用作查询以匹配存储的查询的文档。

    其实就是存储 query本身，然后触发，然后可以通过query进行查询，获取结果。（这个可以--使用事件通知之类的触发，可以获取通知、警告等效果）

    

    [Elasticsearch：理解 Elasticsearch Percolate 查询 - 掘金 (juejin.cn)](https://juejin.cn/post/7020585236433993735)  ---这篇文档。

    [Elasticsearch：理解 Elasticsearch Percolate 查询-CSDN博客](https://blog.csdn.net/UbuntuTouch/article/details/120427651?ops_request_misc=&request_id=&biz_id=102&utm_term=elasticsearch 渗透查询&utm_medium=distribute.pc_search_result.none-task-blog-2~all~sobaiduweb~default-0-120427651.142^v99^pc_search_result_base9&spm=1018.2226.3001.4187)



​		如果，设置search.allow_expensive_queries=false，那么，上述的query都会失效，默认是true的。



## 2、Query and filter context（查询和过滤器上下文）

### 2.1 Relevance scores (相关性分数)

相关性分数
默认情况下，Elasticsearch 按相关性得分对匹配的搜索结果进行排序，相关性得分衡量每个文档与查询的匹配程度。

相关性分数是一个正浮点数，在搜索 API 的 _score 元数据字段中返回。 _score 越高，文档越相关。 虽然每种查询类型可以以不同的方式计算相关性分数，但分数计算还取决于查询子句是在*查询上下文中运行还是在过滤器上下文中运行。*

###  2.2 Query context（查询上下文）

	在查询上下文中，查询子句回答“此文档与此查询子句的匹配程度如何？”的问题。 除了确定文档是否匹配之外，查询子句还会计算 _score 元数据字段中的相关性分数。

每当查询子句传递给查询参数（例如搜索 API 中的查询参数）时，查询上下文就会生效。

### 2.3 Filter context（过滤器上下文）

在过滤器上下文中，查询子句回答“此文档与此查询子句匹配吗？”的问题。 答案是简单的“是”或“否”——“不计算分数”。 过滤器上下文主要用于过滤结构化数据，例如

- 该时间戳是否在 2015 年至 2016 年范围内？
- 状态字段是否设置为“已发布”？

Elasticsearch 将自动缓存常用的过滤器，以提高性能。

每当查询子句传递给过滤器参数（例如 bool 查询中的 filter 或must_not 参数、constant_score 查询中的过滤器参数或过滤器聚合）时，过滤器上下文就会生效。



### 2.3 query和filter contexts的例子



下面是在“搜索” API 的查询和过滤上下文中使用查询子句的示例。 此查询将匹配满足以下所有条件的文档：

- The `title` field contains the word `search`.
- The `content` field contains the word `elasticsearch`.
- The `status` field contains the exact word `published`.
- The `publish_date` field contains a date from 1 Jan 2015 onwards.

```
GET /_search
{
  "query": {  //1
    "bool": { //2
      "must": [
        { "match": { "title":   "Search"        }},
        { "match": { "content": "Elasticsearch" }}
      ],
      "filter": [//3 
        { "term":  { "status": "published" }},
        { "range": { "publish_date": { "gte": "2015-01-01" }}}
      ]
    }
  }
}
```



1 查询参数指示查询上下文。

2 bool 和两个 match 子句在查询上下文中使用，这意味着它们用于对每个文档的匹配程度进行评分。

3 过滤器参数指示过滤器上下文。 它的术语和范围子句用于过滤器上下文中。 它们会过滤掉不匹配的文档，但不会影响匹配文档的分数。
​

warning :为查询上下文中的查询计算的分数表示为单精度浮点数； 它们只有 24 位的有效位数精度。 超过有效数精度的分数计算将转换为浮点数，但会损失精度。

 tips: 在查询上下文中使用查询子句来获取应影响匹配文档分数（即文档匹配程度）的条件，并在过滤器上下文中使用所有其他查询子句。

## 3、Compound queries

复合查询包装其他复合查询或叶查询，以组合它们的结果和分数，改变它们的行为，或者从查询切换到过滤上下文。

以下是复合查询的种类:

- **[`bool` query](https://www.elastic.co/guide/en/elasticsearch/reference/8.7/query-dsl-bool-query.html)** 布尔查询

  用于组合多个叶或复合查询子句的默认查询，如“must”、“should”、“must_not”或“filter”子句。 Must 和 Should 子句的得分相加 ——“匹配的子句越多越好”——而must_not 和filter 子句则在过滤器上下文中执行。

- **[`boosting` query](https://www.elastic.co/guide/en/elasticsearch/reference/8.7/query-dsl-boosting-query.html)** 增强查询

  返回与正查询匹配的文档，但降低也与负查询匹配的文档的分数。

- **[`constant_score` query](https://www.elastic.co/guide/en/elasticsearch/reference/8.7/query-dsl-constant-score-query.html)**

  A query which wraps another query, but executes it in filter context. All matching documents are given the same “constant” `_score`.

  包装另一个查询的查询，但在过滤器上下文中执行它。 所有匹配的文档都被赋予相同的“常量”_score。

- **[`dis_max` query](https://www.elastic.co/guide/en/elasticsearch/reference/8.7/query-dsl-dis-max-query.html)**

  A query which accepts multiple queries, and returns any documents which match any of the query clauses. While the `bool` query combines the scores from all matching queries, the `dis_max` query uses the score of the single best- matching query clause.

  接受多个查询并返回与任何查询子句匹配的任何文档的查询。 bool 查询组合了所有匹配查询的分数，而 dis_max 查询则使用单个最佳匹配查询子句的分数。

- **[`function_score` query](https://www.elastic.co/guide/en/elasticsearch/reference/8.7/query-dsl-function-score-query.html)**

  Modify the scores returned by the main query with functions to take into account factors like popularity, recency, distance, or custom algorithms implemented with scripting.

  使用函数修改主查询返回的分数，以考虑流行度、新近度、距离或通过脚本实现的自定义算法等因素。



### 3.1 bool query

与其他查询的布尔组合相匹配的文档的查询。 bool 查询映射到 Lucene BooleanQuery。 它是使用一个或多个布尔子句构建的，每个子句都有一个类型化的出现。 发生类型有：

发生情况 描述
must

子句（查询）必须出现在匹配的文档中，并将有助于得分。----类似于sql中的and

filter

子句（查询）必须出现在匹配的文档中。 然而，与必须不同的是，查询的分数将被忽略。 过滤器子句在过滤器上下文中执行，这意味着忽略评分并考虑对子句进行缓存。

should

子句（查询）应出现在匹配的文档中。--类似于sql中的or

must_not

子句（查询）不得出现在匹配文档中。 子句在过滤器上下文中执行，这意味着忽略评分并考虑对子句进行缓存。 由于评分被忽略，因此所有文档的分数都返回 0。

bool 查询采用“匹配越多越好”的方法，因此每个匹配的“must”或“should”子句的分数将相加，以提供每个文档的最终 _score。

```
POST _search
{
  "query": {
    "bool" : {
      "must" : {
        "term" : { "user.id" : "kimchy" }
      },
      "filter": {
        "term" : { "tags" : "production" }
      },
      "must_not" : {
        "range" : {
          "age" : { "gte" : 10, "lte" : 20 }
        }
      },
      "should" : [
        { "term" : { "tags" : "env1" } },
        { "term" : { "tags" : "deployed" } }
      ],
      "minimum_should_match" : 1,
      "boost" : 1.0
    }
  }
}
```



**使用minimum_should_matchedit**
您可以使用minimum_should_match参数来指定should子句返回的文档必须匹配的数量或百分比。

如果 bool 查询至少包含一个should子句且没有must或filter子句，则默认值为1。否则，默认值为0。

对于其他有效值，请参阅minimum_should_match参数。

### Scoring with `bool.filter` 用bool.filter 来进行打分

在过滤器元素下指定的查询对评分没有影响 - 分数返回为 0。分数仅受指定查询的影响。 例如，以下所有三个查询都会返回状态字段包含术语“活动”的所有文档。

第一个查询为所有文档分配 0 分，因为未指定评分查询：

```
GET _search
{
  "query": {
    "bool": {
      "filter": {
        "term": {
          "status": "active"
        }
      }
    }
  }
}
```



This `bool` query has a `match_all` query, which assigns a score of `1.0` to all documents.

```
GET _search
{
  "query": {
    "bool": {
      "must": {
        "match_all": {}
      },
      "filter": {
        "term": {
          "status": "active"
        }
      }
    }
  }
}
```

 This `constant_score` query behaves in exactly the same way as the second example above. The `constant_score` query assigns a score of `1.0` to all documents matched by the filter.

``` 
GET _search
{
  "query": {
    "constant_score": {
      "filter": {
        "term": {
          "status": "active"
        }
      }
    }
  }
}
```

###  Named queries--命名查询

Each query accepts a `_name` in its top level definition. You can use named queries to track which queries matched returned documents. If named queries are used, the response includes a `matched_queries` property for each hit.

Supplying duplicate `_name` values in the same request results in undefined behavior. Queries with duplicate names may overwrite each other. Query names are assumed to be unique within a single request.

每个查询在其顶级定义中接受一个“_name”。 您可以使用命名查询来跟踪哪些查询与返回的文档匹配。 如果使用命名查询，则响应将为每个命中包含一个“matched_queries”属性。

在同一请求中提供重复的“_name”值会导致未定义的行为。 具有重复名称的查询可能会相互覆盖。 假设查询名称在单个请求中是唯一的。

```
GET /_search
{
  "query": {
    "bool": {
      "should": [
        { "match": { "name.first": { "query": "shay", "_name": "first" } } },
        { "match": { "name.last": { "query": "banon", "_name": "last" } } }
      ],
      "filter": {
        "terms": {
          "name.last": [ "banon", "kimchy" ],
          "_name": "test"
        }
      }
    }
  }
}
```

名为 include_named_queries_score 的请求参数控制是否返回与匹配查询关联的分数。 设置后，响应包含一个 matches_queries 映射，其中包含作为键匹配的查询的名称及其作为值的关联分数。

请注意，分数可能不会对文档的最终分数产生影响，例如出现在过滤器或must_not上下文中的命名查询，或者出现在忽略或修改分数的子句（如constant_score或function_score_query）内。

```
GET /_search?include_named_queries_score
{
  "query": {
    "bool": {
      "should": [
        { "match": { "name.first": { "query": "shay", "_name": "first" } } },
        { "match": { "name.last": { "query": "banon", "_name": "last" } } }
      ],
      "filter": {
        "terms": {
          "name.last": [ "banon", "kimchy" ],
          "_name": "test"
        }
      }
    }
  }
}
```

​       warning 此功能会在搜索响应中的每次命中时重新运行每个命名查询。 通常，这会给请求增加少量开销。 然而，对大量命中使用计算成本昂贵的命名查询可能会增加显着的开销。 例如，命名查询与许多存储桶上的“top_hits”聚合相结合可能会导致响应时间更长。



在 Elasticsearch 查询中，Named Queries（命名查询）通常是指使用 "named_query" 或 "query_name" 参数来为查询定义一个名称。这个名称可以用于在复杂的查询中引用或组织查询。

命名查询的主要用途包括：

1. **可重用性：** 通过命名查询，你可以定义一次查询并在多个地方引用它，而无需多次编写相同的查询逻辑。这提高了查询的可重用性和维护性。
2. **清晰度和组织性：** 在复杂的查询中，可以使用命名查询来提高查询语句的可读性和组织性。通过为查询片段分配有意义的名称，可以更容易理解查询的结构和意图。

下面是一个简单的示例，演示如何在 Elasticsearch 查询中使用命名查询：

``` 
{
  "query": {
    "bool": {
      "must": [
        {
          "term": {
            "field1": {
              "value": "value1"
            }
          }
        },
        {
          "named_query": {
            "name": "additional_filter"
          }
        }
      ]
    }
  },
  "named_queries": {
    "additional_filter": {
      "range": {
        "field2": {
          "gte": 100
        }
      }
    }
  }
}
```



### 3.2  Boosting 增强查询

返回与肯定查询匹配的文档，同时降低也与否定查询匹配的文档的相关性得分。

您可以使用提升查询来降级某些文档，而不将它们从搜索结果中排除。

请求示例

```
GET /_search
{
  "query": {
    "boosting": {
      "positive": {
        "term": {
          "text": "apple"
        }
      },
      "negative": {
        "term": {
          "text": "pie tart fruit crumble tree"
        }
      },
      "negative_boost": 0.5
    }
  }
}
```

**用于提升的顶级参数编辑**
positive
（必需，查询对象）您希望运行的查询。 任何返回的文档都必须与此查询匹配。
negative
（必填，查询对象）用于降低匹配文档的相关性得分的查询。

如果返回的文档与正查询和此查询匹配，则提升查询将计算文档的最终相关性得分，如下所示：

从正查询中获取原始相关性得分。
将分数乘以 negative_boost 值。
negative_boost
（必需，浮点数）0 到 1.0 之间的浮点数，用于降低与否定查询匹配的文档的相关性分数。

### 3.3 Constant score query-- 恒定分数查询

包装过滤器查询并返回相关性分数等于 boost 参数值的每个匹配文档。

``` 
GET /_search
{
  "query": {
    "constant_score": {
      "filter": {
        "term": { "user.id": "kimchy" }
      },
      "boost": 1.2
    }
  }
}
```

Constant_score的顶级参数编辑
filter
（必需，查询对象）您希望运行的过滤器查询。 任何返回的文档都必须与此查询匹配。

过滤查询不计算相关性分数。 为了提高性能，Elasticsearch 自动缓存常用的过滤器查询。

boost
（可选，浮点数）浮点数，用作与过滤器查询匹配的每个文档的恒定相关性得分。 默认为 1.0。



### 3.4 Disjunction max 析取最大查询

https://blog.csdn.net/epitomizelu/article/details/106293028

当需要对同一个[字符串](https://so.csdn.net/so/search?q=字符串&spm=1001.2101.3001.7020)在多个字段中进行查询时，用bool查询在算分时会对多个查询结果的算分进行平均，而实际上有可能我们需要的是最匹配的那个字段对应的那条记录，这个时候就可以用到disjunciton max query 了。

返回与一个或多个包装查询匹配的文档，称为查询子句或子句。

如果返回的文档与多个查询子句匹配，则 dis_max 查询会为该文档分配任何匹配子句中的最高相关性分数，并为任何其他匹配子查询分配平局增量。

请求示例

```
GET /_search
{
  "query": {
    "dis_max": {
      "queries": [
        { "term": { "title": "Quick pets" } },
        { "term": { "body": "Quick pets" } }
      ],
      "tie_breaker": 0.7
    }
  }
}
```



dis_maxedit 的顶级参数
queries:
（必需，查询对象数组）包含一个或多个查询子句。 返回的文档必须与这些查询中的一个或多个匹配。 如果文档与多个查询匹配，Elasticsearch 将使用最高的相关性分数。
tie_breaker:
（可选，float）0到1.0之间的浮点数，用于增加匹配多个查询子句的文档的相关性分数。 默认为 0.0。

您可以使用 tie_breaker 值向在多个字段中包含相同术语的文档分配比仅在多个字段中最好的字段中包含该术语的文档更高的相关性分数，而不会将此与多个字段中两个不同术语的更好情况混淆。 字段。

如果文档匹配多个子句，则 dis_max 查询将计算文档的相关性得分，如下所示：

​	从得分最高的匹配子句中获取相关性得分。
​	将任何其他匹配子句的分数乘以 tie_breaker 值。
​	将最高分数添加到相乘分数中。
​	如果 tie_breaker 值大于 0.0，则所有匹配子句都会计数，但分数最高的子句计数最多。



也可以用multi_match的best field模式，得出的结果是一样的。二者的区别在于，dis_max查询可以自动找出得分最高的字段对应的文档，并用tie breaker将得分其次的按照一定的权重进入算分，这个权重是针对结果的，而不是针对某个字段的，是后知的。mutli_match可以给指定字段加权重，是先知的。



### 3.5 Function score



https://blog.csdn.net/u010454030/article/details/132367802

function_score 允许您修改查询检索的文档的分数。 例如，如果评分函数的计算成本很高，并且足以计算经过过滤的文档集的评分，则这可能很有用。

要使用 function_score，用户必须定义一个查询和一个或多个函数，用于为查询返回的每个文档计算新分数。

function_score 只能与一个函数一起使用，如下所示：

``` 
GET /_search
{
  "query": {
    "function_score": {
      "query": { "match_all": {} },
      "boost": "5",
      "random_score": {},  //
      "boost_mode": "multiply"
    }
  }
}
```

此外，可以组合多种功能。 在这种情况下，人们可以选择仅当文档与给定的过滤查询匹配时才应用该函数

```
GET /_search
{
  "query": {
    "function_score": {
      "query": { "match_all": {} },
      "boost": "5", 
      "functions": [
        {
          "filter": { "match": { "test": "bar" } },
          "random_score": {}, 
          "weight": 23
        },
        {
          "filter": { "match": { "test": "cat" } },
          "weight": 42
        }
      ],
      "max_boost": 42,
      "score_mode": "max",
      "boost_mode": "multiply",
      "min_score": 42
    }
  }
}
```

note :每个函数的过滤查询产生的分数并不重要。

如果函数没有给出过滤器，则相当于指定“match_all”：{}

首先，根据定义的函数对每个文档进行评分。 参数 Score_mode 指定如何组合计算的分数：

| `multiply` | scores are multiplied (default)                          |
| ---------- | -------------------------------------------------------- |
| `sum`      | scores are summed                                        |
| `avg`      | scores are averaged                                      |
| `first`    | the first function that has a matching filter is applied |
| `max`      | maximum score is used                                    |
| `min`      | minimum score is used                                    |

由于分数可以采用不同的尺度（例如，对于衰减函数，分数在 0 到 1 之间，但对于 field_value_factor 是任意的），而且有时需要函数对分数产生不同的影响，因此可以使用用户定义的值来调整每个函数的分数 重量。 可以为函数数组中的每个函数定义权重（上面的示例），并乘以相应函数计算的分数。 如果给出权重而没有任何其他函数声明，则权重充当仅返回权重的函数。

如果score_mode设置为avg，各个分数将通过加权平均值合并。 例如，如果两个函数返回分数 1 和 2，并且它们各自的权重分别为 3 和 4，那么它们的分数将组合为 (1\*3+2\*4)/(3+4)，而不是 (1*3+2 *4)/2。

可以通过设置 max_boost 参数来限制新分数不超过一定的限制。 max_boost 的默认值是 FLT_MAX。

新计算的分数与查询的分数相结合。 参数 boost_mode 定义如何：

| `multiply` | query score and function score is multiplied (default)  |
| ---------- | ------------------------------------------------------- |
| `replace`  | only function score is used, the query score is ignored |
| `sum`      | query score and function score are added                |
| `avg`      | average                                                 |
| `max`      | max of query score and function score                   |
| `min`      | min of query score and function score                   |

默认情况下，修改分数不会更改匹配的文档。 要排除不满足特定分数阈值的文档，可以将 min_score 参数设置为所需的分数阈值。

要使 min_score 发挥作用，需要对查询返回的所有文档进行评分，然后一一过滤掉。

function_score 查询提供了多种类型的评分函数。

- 脚本分数
- 重量
- 随机分数
- 字段值因子
- 衰减函数：高斯、线性、exp

**Script score**----function score的评分函数

script_score 函数允许您包装另一个查询，并可以选择使用脚本表达式从文档中的其他数字字段值派生的计算来自定义其评分。 这是一个简单的示例：

``` 
GET /_search
{
  "query": {
    "function_score": {
      "query": {
        "match": { "message": "elasticsearch" }
      },
      "script_score": {
        "script": {
          "source": "Math.log(2 + doc['my-int'].value)"
        }
      }
    }
  }
}
```





使用主查询 的 [TF-IDF](https://so.csdn.net/so/search?q=TF-IDF&spm=1001.2101.3001.7020) 或者 BM25 算法得出来的默认评分简称为： query_score

使用 Function Score 查询结合自定义策略得出来的评分简称为：function_score

最终用于排序的评分称为 sort_score

在使用了 自定义的 Fuction Score 之后，我们最终得出来的 sort_score 就是使用 query_score 和 function_score以某种运算形式 (score_mode) 计算出来的，这个策略默认是相乘，也即：

sort_score = query_score * function_score


## 3、Full text queries 全文查询

