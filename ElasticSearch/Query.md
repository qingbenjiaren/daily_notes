# Query DSL

Elasticsearch provides a full Query DSL (Domain Specific Language) based on JSON to define queries. Think of the Query DSL as an AST (Abstract Syntax Tree) of queries, consisting of two types of clauses:

**Leaf query clauses**

Leaf query clauses look for a particular value in a particular field, such as the [`match`](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-match-query.html), [`term`](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-term-query.html) or [`range`](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-range-query.html) queries. These queries can be used by themselves.

**Compound query clauses**

Compound query clauses wrap other leaf **or** compound queries and are used to combine multiple queries in a logical fashion (such as the [`bool`](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-bool-query.html) or [`dis_max`](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-dis-max-query.html) query), or to alter their behaviour (such as the [`constant_score`](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-constant-score-query.html) query).

## Query and filter context

### Relevance scores

By default, Elasticsearch sorts matching search results by **relevance score**, which measures how well each document matches a query.

The relevance score is a positive floating point number, returned in the `_score` meta-field of the [search](https://www.elastic.co/guide/en/elasticsearch/reference/current/search-request-body.html) API. The higher the `_score`, the more relevant the document. While each query type can calculate relevance scores differently, score calculation also depends on whether the query clause is run in a **query** or **filter** context.

### Query context

In the query context, a query clause answers the question “*How well does this document match this query clause?*” Besides deciding whether or not the document matches, the query clause also calculates a relevance score in the `_score` meta-field.

Query context is in effect whenever a query clause is passed to a `query` parameter, such as the `query` parameter in the [search](https://www.elastic.co/guide/en/elasticsearch/reference/current/search-request-body.html#request-body-search-query) API.

### Filter context

In a filter context, a query clause answers the question “*Does this document match this query clause?*” The answer is a simple Yes or No — no scores are calculated. Filter context is mostly used for filtering structured data, e.g.

- *Does this `timestamp` fall into the range 2015 to 2016?*
- *Is the `status` field set to `"published"`*?

Frequently used filters will be cached automatically by Elasticsearch, to speed up performance.

Filter context is in effect whenever a query clause is passed to a `filter` parameter, such as the `filter` or `must_not` parameters in the [`bool`](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-bool-query.html) query, the `filter` parameter in the [`constant_score`](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-constant-score-query.html) query, or the [`filter`](https://www.elastic.co/guide/en/elasticsearch/reference/current/search-aggregations-bucket-filter-aggregation.html) aggregation.

### Example of query and filter context

Below is an example of query clauses being used in query and filter context in the `search` API. This query will match documents where all of the following conditions are met:

- The `title` field contains the word `search`.
- The `content` field contains the word `elasticsearch`.
- The `status` field contains the exact word `published`.
- The `publish_date` field contains a date from 1 Jan 2015 onwards.

```json
GET /_search
{
  "query": { 
    "bool": { 
      "must": [
        { "match": { "title":   "Search"        }},
        { "match": { "content": "Elasticsearch" }}
      ],
      "filter": [ 
        { "term":  { "status": "published" }},
        { "range": { "publish_date": { "gte": "2015-01-01" }}}
      ]
    }
  }
}
```

The `query` parameter indicates query context.

The `bool` and two `match` clauses are used in query context, which means that they are used to score how well each document matches.

The `filter` parameter indicates filter context. Its `term` and `range` clauses are used in filter context. They will filter out documents which do not match, but they will not affect the score for matching documents.



## Compound queries

Compound queries wrap other compound or leaf queries, either to combine their results and scores, to change their behaviour, or to switch from query to filter context.

The queries in this group are:

**bool query**

The default query for combining multiple leaf or compound query clauses, as `must`, `should`, `must_not`, or `filter` clauses. The `must` and `should` clauses have their scores combined — the more matching clauses, the better — while the `must_not` and `filter` clauses are executed in filter context.

**boosting query**

Return documents which match a `positive` query, but reduce the score of documents which also match a `negative` query.

**constant_score query**

A query which wraps another query, but executes it in filter context. All matching documents are given the same “constant” `_score`.

**dis_max query**

A query which accepts multiple queries, and returns any documents which match any of the query clauses. While the `bool` query combines the scores from all matching queries, the `dis_max` query uses the score of the single best- matching query clause.

一个查询，它接受多个查询，并返回与任何查询子句匹配的任何文档。 “ bool”查询结合了所有匹配查询的分数，而“ dis_max”查询则使用了单个最佳匹配查询子句的分数。

**function_score query**

Modify the scores returned by the main query with functions to take into account factors like popularity, recency, distance, or custom algorithms implemented with scripting.

使用函数修改主查询返回的分数，以考虑诸如受欢迎程度，新近度，距离或使用脚本实现的自定义算法等因素。



### Boolean query

A query that matches documents matching boolean combinations of other queries. The bool query maps to Lucene `BooleanQuery`. It is built using one or more boolean clauses, each clause with a typed occurrence. The occurrence types are:

| Occur      | Description                                                  |
| ---------- | ------------------------------------------------------------ |
| `must`     | The clause (query) must appear in matching documents and will contribute to the score. |
| `filter`   | The clause (query) must appear in matching documents. However unlike `must` the score of the query will be ignored. Filter clauses are executed in [filter context](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-filter-context.html), meaning that scoring is ignored and clauses are considered for caching. |
| `should`   | The clause (query) should appear in the matching document.   |
| `must_not` | The clause (query) must not appear in the matching documents. Clauses are executed in [filter context](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-filter-context.html) meaning that scoring is ignored and clauses are considered for caching. Because scoring is ignored, a score of `0` for all documents is returned. |

The `bool` query takes a *more-matches-is-better* approach, so the score from each matching `must` or `should` clause will be added together to provide the final `_score` for each document.

```json
POST _search
{
  "query": {
    "bool" : {
      "must" : {
        "term" : { "user" : "kimchy" }
      },
      "filter": {
        "term" : { "tag" : "tech" }
      },
      "must_not" : {
        "range" : {
          "age" : { "gte" : 10, "lte" : 20 }
        }
      },
      "should" : [
        { "term" : { "tag" : "wow" } },
        { "term" : { "tag" : "elasticsearch" } }
      ],
      "minimum_should_match" : 1,
      "boost" : 1.0
    }
  }
}
```

#### Using minimum_should_match

You can use the `minimum_should_match` parameter to specify the number or percentage of `should` clauses returned documents *must* match.

If the `bool` query includes at least one `should` clause and no `must` or `filter` clauses, the default value is `1`. Otherwise, the default value is `0`.

For other valid values, see the [`minimum_should_match` parameter](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-minimum-should-match.html).

### Scoring with bool.filter

Queries specified under the `filter` element have no effect on scoring — scores are returned as `0`. Scores are only affected by the query that has been specified. For instance, all three of the following queries return all documents where the `status` field contains the term `active`.

This first query assigns a score of `0` to all documents, as no scoring query has been specified:

```json
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

```json
ET _search
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

```json
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

### Using named queries to see which clauses matched

If you need to know which of the clauses in the bool query matched the documents returned from the query, you can use [named queries](https://www.elastic.co/guide/en/elasticsearch/reference/current/search-request-body.html#request-body-search-queries-and-filters) to assign a name to each clause.



### Boosting query

Returns documents matching a `positive` query while reducing the [relevance score](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-filter-context.html#relevance-scores) of documents that also match a `negative` query.

You can use the `boosting` query to demote certain documents without excluding them from the search results.

您可以使用boosting查询来降级某些文档，而不必将它们从搜索结果中排除。

#### Top-level parameters for boosting

**`positive`**

(Required, query object) Query you wish to run. Any returned documents must match this query.

**`negative`**

(Required, query object) Query used to decrease the [relevance score](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-filter-context.html#relevance-scores) of matching documents.

If a returned document matches the `positive` query and this query, the `boosting` query calculates the final [relevance score](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-filter-context.html#relevance-scores) for the document as follows:

1. Take the original relevance score from the `positive` query.
2. Multiply the score by the `negative_boost` value.

**`negative_boost`**

(Required, float) Floating point number between `0` and `1.0` used to decrease the [relevance scores](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-filter-context.html#relevance-scores) of documents matching the `negative` query.

用于降低匹配文档的[相关性分数]的查询。

如果返回的文档与“ positive”查询和该查询匹配，则“ boosting”查询将按如下方式计算文档的最终[相关性分数]：

1. 从“正”查询中获取原始的相关性得分。
2. 将分数乘以“ negative_boost”值。

#### Constant score query

Wraps a [filter query](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-bool-query.html) and returns every matching document with a [relevance score](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-filter-context.html#relevance-scores) equal to the `boost` parameter value.

```json
GET /_search
{
    "query": {
        "constant_score" : {
            "filter" : {
                "term" : { "user" : "kimchy"}
            },
            "boost" : 1.2
        }
    }
}
```

#### Top-level parameters for constant_score

**`filter`**

(Required, query object) [Filter query](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-bool-query.html) you wish to run. Any returned documents must match this query.

Filter queries do not calculate [relevance scores](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-filter-context.html#relevance-scores). To speed up performance, Elasticsearch automatically caches frequently used filter queries.

**`boost`**

(Optional, float) Floating point number used as the constant [relevance score](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-filter-context.html#relevance-scores) for every document matching the `filter` query. Defaults to `1.0`.



### Disjunction max query

Returns documents matching one or more wrapped queries, called query clauses or clauses.

If a returned document matches multiple query clauses, the `dis_max` query assigns the document the highest relevance score from any matching clause, plus a tie breaking increment for any additional matching subqueries.

You can use the `dis_max` to search for a term in fields mapped with different [boost](https://www.elastic.co/guide/en/elasticsearch/reference/current/mapping-boost.html) factors.

```json
GET /_search
{
    "query": {
        "dis_max" : {
            "queries" : [
                { "term" : { "title" : "Quick pets" }},
                { "term" : { "body" : "Quick pets" }}
            ],
            "tie_breaker" : 0.7
        }
    }
}
```

> https://www.elastic.co/guide/cn/elasticsearch/guide/current/_best_fields.html#_best_fields

#### Top-level parameters for dis_max

**`queries`**

(Required, array of query objects) Contains one or more query clauses. Returned documents **must match one or more** of these queries. If a document matches multiple queries, Elasticsearch uses the highest [relevance score](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-filter-context.html).

**`tie_breaker`**

(Optional, float) Floating point number between `0` and `1.0` used to increase the [relevance scores](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-filter-context.html#relevance-scores) of documents matching multiple query clauses. Defaults to `0.0`.

You can use the `tie_breaker` value to assign higher relevance scores to documents that contain the same term in multiple fields than documents that contain this term in only the best of those multiple fields, without confusing this with the better case of two different terms in the multiple fields.

If a document matches multiple clauses, the `dis_max` query calculates the relevance score for the document as follows:

1. Take the relevance score from a matching clause with the highest score.
2. Multiply the score from any other matching clauses by the `tie_breaker` value.
3. Add the highest score to the multiplied scores.

If the `tie_breaker` value is greater than `0.0`, all matching clauses count, but the clause with the highest score counts most.



### Function score query

The `function_score` allows you to modify the score of documents that are retrieved by a query. This can be useful if, for example, a score function is computationally expensive and it is sufficient to compute the score on a filtered set of documents.

To use `function_score`, the user has to define a query and one or more functions, that compute a new score for each document returned by the query.

`function_score` can be used with only one function like this:

```json

```

#### TODO





## Full text queries

The full text queries enable you to search [analyzed text fields](https://www.elastic.co/guide/en/elasticsearch/reference/current/analysis.html) such as the body of an email. The query string is processed using the same analyzer that was applied to the field during indexing.

The queries in this group are:

**[`intervals` query](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-intervals-query.html)**

A full text query that allows fine-grained control of the ordering and proximity of matching terms.

**[`match` query](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-match-query.html)**

The standard query for performing full text queries, including fuzzy matching and phrase or proximity queries.

**[`match_bool_prefix` query](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-match-bool-prefix-query.html)**

Creates a `bool` query that matches each term as a `term` query, except for the last term, which is matched as a `prefix` query

**[`match_phrase` query](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-match-query-phrase.html)**

Like the `match` query but used for matching exact phrases or word proximity matches.

**[`match_phrase_prefix` query](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-match-query-phrase-prefix.html)**

Like the `match_phrase` query, but does a wildcard search on the final word.

**[`multi_match` query](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-multi-match-query.html)**

The multi-field version of the `match` query.

**[`common` terms query](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-common-terms-query.html)**

A more specialized query which gives more preference to uncommon words.

**[`query_string` query](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-query-string-query.html)**

Supports the compact Lucene [query string syntax](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-query-string-query.html#query-string-syntax), allowing you to specify AND|OR|NOT conditions and multi-field search within a single query string. For expert users only.

**[`simple_query_string` query](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-simple-query-string-query.html)**

A simpler, more robust version of the `query_string` syntax suitable for exposing directly to users.

### Intervals query

Returns documents based on the order and proximity of matching terms.

### Example request

The following `intervals` search returns documents containing `my favorite food` immediately followed by `hot water` or `cold porridge` in the `my_text` field.

This search would match a `my_text` value of `my favorite food is cold porridge` but not `when it's cold my favorite food is porridge`.

```json
POST _search
{
  "query": {
    "intervals" : {
      "my_text" : {
        "all_of" : {
          "ordered" : true,
          "intervals" : [
            {
              "match" : {
                "query" : "my favorite food",
                "max_gaps" : 0,
                "ordered" : true
              }
            },
            {
              "any_of" : {
                "intervals" : [
                  { "match" : { "query" : "hot water" } },
                  { "match" : { "query" : "cold porridge" } }
                ]
              }
            }
          ]
        }
      }
    }
  }
}
```

### Top-level parameters for intervals

<field>

(Required, rule object) Field you wish to search.

The value of this parameter is a rule object used to match documents based on matching terms, order, and proximity.

Valid rules include:

- [`match`](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-intervals-query.html#intervals-match)
- [`prefix`](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-intervals-query.html#intervals-prefix)
- [`wildcard`](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-intervals-query.html#intervals-wildcard)
- [`fuzzy`](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-intervals-query.html#intervals-fuzzy)
- [`all_of`](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-intervals-query.html#intervals-all_of)
- [`any_of`](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-intervals-query.html#intervals-any_of)



## Match query

Returns documents that match a provided text, number, date or boolean value. The provided text is analyzed before matching.

The `match` query is the standard query for performing a full-text search, including options for fuzzy matching.

### Example request

```json
GET /_search
{
    "query": {
        "match" : {
            "message" : {
                "query" : "this is a test"
            }
        }
    }
}
```

### Top-level parameters for match

<field>

(Required, object) Field you wish to search.

#### Parameters for<field>

##### query

(Required) Text, number, boolean value or date you wish to find in the provided<field>.

The `match` query [analyzes](https://www.elastic.co/guide/en/elasticsearch/reference/current/analysis.html) any provided text before performing a search. This means the `match` query can search [`text`](https://www.elastic.co/guide/en/elasticsearch/reference/current/text.html) fields for analyzed tokens rather than an exact term.

##### analyzer

(Optional, string) [Analyzer](https://www.elastic.co/guide/en/elasticsearch/reference/current/analysis.html) used to convert the text in the `query` value into tokens. Defaults to the [index-time analyzer](https://www.elastic.co/guide/en/elasticsearch/reference/current/specify-analyzer.html#specify-index-time-analyzer) mapped for the <field>. If no analyzer is mapped, the index’s default analyzer is used.

##### auto_generate_synomyms_phrase_query

(Optional, boolean) If `true`, [match phrase](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-match-query-phrase.html) queries are automatically created for multi-term synonyms. Defaults to `true`.

See [Use synonyms with match query](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-match-query.html#query-dsl-match-query-synonyms) for an example.

##### fuzziness

(Optional, string) Maximum edit distance allowed for matching. See [Fuzziness](https://www.elastic.co/guide/en/elasticsearch/reference/current/common-options.html#fuzziness) for valid values and more information. See [Fuzziness in the match query](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-match-query.html#query-dsl-match-query-fuzziness) for an example.

##### max_expansions

(Optional, integer) Maximum number of terms to which the query will expand. Defaults to `50`.



> https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-match-query.html



## Match boolean prefix query

A `match_bool_prefix` query analyzes its input and constructs a [`bool` query](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-bool-query.html) from the terms. Each term except the last is used in a `term` query. The last term is used in a `prefix` query. A `match_bool_prefix` query such as

```json
GET /_search
{
    "query": {
        "match_bool_prefix" : {
            "message" : "quick brown f"
        }
    }
}
```

where analysis produces the terms `quick`, `brown`, and `f` is similar to the following `bool` query

```json
GET /_search
{
    "query": {
        "bool" : {
            "should": [
                { "term": { "message": "quick" }},
                { "term": { "message": "brown" }},
                { "prefix": { "message": "f"}}
            ]
        }
    }
}
```

> https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-match-bool-prefix-query.html

An important difference between the `match_bool_prefix` query and [`match_phrase_prefix`](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-match-query-phrase-prefix.html) is that the `match_phrase_prefix` query matches its terms as a phrase, but the `match_bool_prefix` query can match its terms in any position. The example `match_bool_prefix` query above could match a field containing containing `quick brown fox`, but it could also match `brown fox quick`. It could also match a field containing the term `quick`, the term `brown` and a term starting with `f`, appearing in any position.

## Match phrase query

The `match_phrase` query analyzes the text and creates a `phrase` query out of the analyzed text. For example:

```json
GET /_search
{
    "query": {
        "match_phrase" : {
            "message" : "this is a test"
        }
    }
}
```



## Match phrase prefix query

Returns documents that contain the words of a provided text, in the **same order** as provided. The last term of the provided text is treated as a [prefix](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-prefix-query.html), matching any words that begin with that term.

The following search returns documents that contain phrases beginning with `quick brown f` in the `message` field.

This search would match a `message` value of `quick brown fox` or `two quick brown ferrets` but not `the fox is quick and brown`.

```json
GET /_search
{
    "query": {
        "match_phrase_prefix" : {
            "message" : {
                "query" : "quick brown f"
            }
        }
    }
}
```



## Multi-match query

The `multi_match` query builds on the [`match` query](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-match-query.html) to allow multi-field queries:

```json
GET /_search
{
  "query": {
    "multi_match" : {
      "query":    "this is a test", 
      "fields": [ "subject", "message" ] 
    }
  }
}
```

### fields and per-field boosting

```json
GET /_search
{
  "query": {
    "multi_match" : {
      "query":    "Will Smith",
      "fields": [ "title", "*_name" ] 
    }
  }
}
```

Query the `title`, `first_name` and `last_name` fields.

Individual fields can be boosted with the caret (`^`) notation:

```json
GET /_search
{
  "query": {
    "multi_match" : {
      "query" : "this is a test",
      "fields" : [ "subject^3", "message" ] 
    }
  }
}
```

The `subject` field is three times as important as the `message` field.

If no `fields` are provided, the `multi_match` query defaults to the `index.query.default_field` index settings, which in turn defaults to `*`. `*` extracts all fields in the mapping that are eligible to term queries and filters the metadata fields. All extracted fields are then combined to build a query.

> 一次可以查询的字段数有限制。 它由index.query.bool.max_clause_count搜索设置定义，默认为1024。

### types of multi_match query

he way the `multi_match` query is executed internally depends on the `type` parameter, which can be set to:

| `best_fields`   | (**default**) Finds documents which match any field, but uses the `_score` from the best field. See [`best_fields`](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-multi-match-query.html#type-best-fields). |
| --------------- | ------------------------------------------------------------ |
| `most_fields`   | Finds documents which match any field and combines the `_score` from each field. See [`most_fields`](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-multi-match-query.html#type-most-fields). |
| `cross_fields`  | Treats fields with the same `analyzer` as though they were one big field. Looks for each word in **any** field. See [`cross_fields`](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-multi-match-query.html#type-cross-fields). |
| `phrase`        | Runs a `match_phrase` query on each field and uses the `_score` from the best field. See [`phrase` and `phrase_prefix`](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-multi-match-query.html#type-phrase). |
| `phrase_prefix` | Runs a `match_phrase_prefix` query on each field and uses the `_score` from the best field. See [`phrase` and `phrase_prefix`](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-multi-match-query.html#type-phrase). |
| `bool_prefix`   | Creates a `match_bool_prefix` query on each field and combines the `_score` from each field. See [`bool_prefix`](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-multi-match-query.html#type-bool-prefix). |

#### best-field

The `best_fields` type is most useful when you are searching for multiple words best found in the same field. For instance “brown fox” in a single field is more meaningful than “brown” in one field and “fox” in the other.

当您搜索在同一字段中找到的多个单词时，best_fields类型最有用。 例如，单个字段中的“brown fox”比一个字段中的“brown”和另一字段中的“ fox”更有意义。

The `best_fields` type generates a [`match` query](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-match-query.html) for each field and wraps them in a [`dis_max`](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-dis-max-query.html) query, to find the single best matching field. For instance, this query:

```json
GET /_search
{
  "query": {
    "multi_match" : {
      "query":      "brown fox",
      "type":       "best_fields",
      "fields":     [ "subject", "message" ],
      "tie_breaker": 0.3
    }
  }
}
```

would be executed as:

```json
GET /_search
{
  "query": {
    "dis_max": {
      "queries": [
        { "match": { "subject": "brown fox" }},
        { "match": { "message": "brown fox" }}
      ],
      "tie_breaker": 0.3
    }
  }
}
```

Normally the `best_fields` type uses the score of the **single** best matching field, but if `tie_breaker` is specified, then it calculates the score as follows:

- the score from the best matching field
- plus `tie_breaker * _score` for all other matching fields

#### most_fields

The `most_fields` type is most useful when querying multiple fields that contain the same text analyzed in different ways. For instance, the main field may contain synonyms, stemming and terms without diacritics. A second field may contain the original terms, and a third field might contain shingles. By combining scores from all three fields we can match as many documents as possible with the main field, but use the second and third fields to push the most similar results to the top of the list.

This query:

```json
GET /_search
{
  "query": {
    "multi_match" : {
      "query":      "quick brown fox",
      "type":       "most_fields",
      "fields":     [ "title", "title.original", "title.shingles" ]
    }
  }
}
```

would be executed as:

```json
GET /_search
{
  "query": {
    "bool": {
      "should": [
        { "match": { "title":          "quick brown fox" }},
        { "match": { "title.original": "quick brown fox" }},
        { "match": { "title.shingles": "quick brown fox" }}
      ]
    }
  }
}
```

The score from each `match` clause is added together, then divided by the number of `match` clauses.

Also, accepts `analyzer`, `boost`, `operator`, `minimum_should_match`, `fuzziness`, `lenient`, `prefix_length`, `max_expansions`, `rewrite`, `zero_terms_query` and `cutoff_frequency`, as explained in [match query](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-match-query.html), but **see [`operator` and `minimum_should_match`](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-multi-match-query.html#operator-min)**.



#### phrase and phrase_prefix

The `phrase` and `phrase_prefix` types behave just like [`best_fields`](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-multi-match-query.html#type-best-fields), but they use a `match_phrase` or `match_phrase_prefix` query instead of a `match` query.

This query:

```json
GET /_search
{
  "query": {
    "multi_match" : {
      "query":      "quick brown f",
      "type":       "phrase_prefix",
      "fields":     [ "subject", "message" ]
    }
  }
}
```

would be executed as:

```json
GET /_search
{
  "query": {
    "dis_max": {
      "queries": [
        { "match_phrase_prefix": { "subject": "quick brown f" }},
        { "match_phrase_prefix": { "message": "quick brown f" }}
      ]
    }
  }
}
```

The `fuzziness` parameter cannot be used with the `phrase` or `phrase_prefix` type.

#### cross_fields

The `cross_fields` type is particularly useful with structured documents where multiple fields **should** match. For instance, when querying the `first_name` and `last_name`fields for “Will Smith”, the best match is likely to have “Will” in one field and “Smith” in the other.

```
This sounds like a job for most_fields but there are two problems with that approach. The first problem is that operator and minimum_should_match are applied per-field, instead of per-term (see explanation above).

The second problem is to do with relevance: the different term frequencies in the first_name and last_name fields can produce unexpected results.

For instance, imagine we have two people: “Will Smith” and “Smith Jones”. “Smith” as a last name is very common (and so is of low importance) but “Smith” as a first name is very uncommon (and so is of great importance).

If we do a search for “Will Smith”, the “Smith Jones” document will probably appear above the better matching “Will Smith” because the score of first_name:smith has trumped the combined scores of first_name:will plus last_name:smith.
```

One way of dealing with these types of queries is simply to index the `first_name` and `last_name` fields into a single `full_name` field. Of course, this can only be done at index time.

The `cross_field` type tries to solve these problems at query time by taking a *term-centric* approach. It first analyzes the query string into individual terms, then looks for each term in any of the fields, as though they were one big field.

A query like:

```json
GET /_search
{
  "query": {
    "multi_match" : {
      "query":      "Will Smith",
      "type":       "cross_fields",
      "fields":     [ "first_name", "last_name" ],
      "operator":   "and"
    }
  }
}
```

is executed as:

```
+(first_name:will  last_name:will)
+(first_name:smith last_name:smith)
```

In other words, **all terms** must be present **in at least one field** for a document to match. (Compare this to [the logic used for `best_fields` and `most_fields`](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-multi-match-query.html#operator-min).)

Note that cross_fields is usually only useful on short string fields that all have a boost of 1. Otherwise boosts, term freqs and length normalization contribute to the score in such a way that the blending of term statistics is not meaningful anymore.

#### cross_field and analysis

The `cross_field` type can only work in term-centric mode on fields that have the same analyzer. Fields with the same analyzer are grouped together as in the example above. If there are multiple groups, they are combined with a `bool`query.

#### bool_prefix

bool_prefix类型的评分行为与most_fields相似，但是使用match_bool_prefix查询而不是match查询。

```json
GET /_search
{
  "query": {
    "multi_match" : {
      "query":      "quick brown f",
      "type":       "bool_prefix",
      "fields":     [ "subject", "message" ]
    }
  }
}
```

### Common Terms Query

> Use [Match](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-match-query.html) instead, which skips blocks of documents efficiently, without any configuration, provided that the total number of hits is not tracked.

The `common` terms query is a modern alternative to stopwords which improves the precision and recall of search results (by taking stopwords into account), without sacrificing performance.

#### The problem

Every term in a query has a cost. A search for `"The brown fox"` requires three term queries, one for each of `"the"`, `"brown"` and `"fox"`, all of which are executed against all documents in the index. The query for `"the"` is likely to match many documents and thus has a much smaller impact on relevance than the other two terms.

Previously, the solution to this problem was to ignore terms with high frequency. By treating `"the"` as a *stopword*, we reduce the index size and reduce the number of term queries that need to be executed.

The problem with this approach is that, while stopwords have a small impact on relevance, they are still important. If we remove stopwords, we lose precision, (eg we are unable to distinguish between `"happy"` and `"not happy"`) and we lose recall (eg text like `"The The"` or `"To be or not to be"`would simply not exist in the index).

#### The solution

The `common` terms query divides the query terms into two groups: more important (ie *low frequency* terms) and less important (ie *high frequency* terms which would previously have been stopwords).

First it searches for documents which match the more important terms. These are the terms which appear in fewer documents and have a greater impact on relevance.

Then, it executes a second query for the less important terms — terms which appear frequently and have a low impact on relevance. But instead of calculating the relevance score for **all** matching documents, it only calculates the `_score` for documents already matched by the first query. In this way the high frequency terms can improve the relevance calculation without paying the cost of poor performance.

If a query consists only of high frequency terms, then a single query is executed as an `AND` (conjunction) query, in other words all terms are required. Even though each individual term will match many documents, the combination of terms narrows down the resultset to only the most relevant. The single query can also be executed as an `OR` with a specific [`minimum_should_match`](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-minimum-should-match.html), in this case a high enough value should probably be used.

Terms are allocated to the high or low frequency groups based on the `cutoff_frequency`, which can be specified as an absolute frequency (`>=1`) or as a relative frequency (`0.0 .. 1.0`). (Remember that document frequencies are computed on a per shard level as explained in the blog post [Relevance is broken](https://www.elastic.co/guide/en/elasticsearch/guide/2.x/relevance-is-broken.html).)

Perhaps the most interesting property of this query is that it adapts to domain specific stopwords automatically. For example, on a video hosting site, common terms like `"clip"` or `"video"` will automatically behave as stopwords without the need to maintain a manual list.

#### Example

In this example, words that have a document frequency greater than 0.1% (eg `"this"` and `"is"`) will be treated as *common terms*.

```java
GET /_search
{
    "query": {
        "common": {
            "body": {
                "query": "this is bonsai cool",
                "cutoff_frequency": 0.001
            }
        }
    }
}
```

The number of terms which should match can be controlled with the [`minimum_should_match`](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-minimum-should-match.html) (`high_freq`, `low_freq`), `low_freq_operator` (default `"or"`) and `high_freq_operator` (default `"or"`) parameters.

For low frequency terms, set the `low_freq_operator` to `"and"` to make all terms required:

```json
GET /_search
{
    "query": {
        "common": {
            "body": {
                "query": "nelly the elephant as a cartoon",
                "cutoff_frequency": 0.001,
                "low_freq_operator": "and"
            }
        }
    }
}
```

which is roughly equivalent to:

```json
GET /_search
{
    "query": {
        "bool": {
            "must": [
            { "term": { "body": "nelly"}},
            { "term": { "body": "elephant"}},
            { "term": { "body": "cartoon"}}
            ],
            "should": [
            { "term": { "body": "the"}},
            { "term": { "body": "as"}},
            { "term": { "body": "a"}}
            ]
        }
    }
}
```

Alternatively use [`minimum_should_match`](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-minimum-should-match.html) to specify a minimum number or percentage of low frequency terms which must be present, for instance:

```json
GET /_search
{
    "query": {
        "common": {
            "body": {
                "query": "nelly the elephant as a cartoon",
                "cutoff_frequency": 0.001,
                "minimum_should_match": 2
            }
        }
    }
}
```

which is roughly equivalent to:

```json
GET /_search
{
    "query": {
        "bool": {
            "must": {
                "bool": {
                    "should": [
                    { "term": { "body": "nelly"}},
                    { "term": { "body": "elephant"}},
                    { "term": { "body": "cartoon"}}
                    ],
                    "minimum_should_match": 2
                }
            },
            "should": [
                { "term": { "body": "the"}},
                { "term": { "body": "as"}},
                { "term": { "body": "a"}}
                ]
        }
    }
}
```

A different [`minimum_should_match`](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-minimum-should-match.html) can be applied for low and high frequency terms with the additional `low_freq` and `high_freq` parameters. Here is an example when providing additional parameters (note the change in structure):

```json
GET /_search
{
    "query": {
        "common": {
            "body": {
                "query": "nelly the elephant not as a cartoon",
                "cutoff_frequency": 0.001,
                "minimum_should_match": {
                    "low_freq" : 2,
                    "high_freq" : 3
                }
            }
        }
    }
}
```

which is roughly equivalent to:

```json
GET /_search
{
    "query": {
        "bool": {
            "must": {
                "bool": {
                    "should": [
                    { "term": { "body": "nelly"}},
                    { "term": { "body": "elephant"}},
                    { "term": { "body": "cartoon"}}
                    ],
                    "minimum_should_match": 2
                }
            },
            "should": {
                "bool": {
                    "should": [
                    { "term": { "body": "the"}},
                    { "term": { "body": "not"}},
                    { "term": { "body": "as"}},
                    { "term": { "body": "a"}}
                    ],
                    "minimum_should_match": 3
                }
            }
        }
    }
}
```

In this case it means the high frequency terms have only an impact on relevance when there are at least three of them. But the most interesting use of the [`minimum_should_match`](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-minimum-should-match.html)for high frequency terms is when there are only high frequency terms:

```json
GET /_search
{
    "query": {
        "common": {
            "body": {
                "query": "how not to be",
                "cutoff_frequency": 0.001,
                "minimum_should_match": {
                    "low_freq" : 2,
                    "high_freq" : 3
                }
            }
        }
    }
}

```

which is roughly equivalent to:

```json
GET /_search
{
    "query": {
        "bool": {
            "should": [
            { "term": { "body": "how"}},
            { "term": { "body": "not"}},
            { "term": { "body": "to"}},
            { "term": { "body": "be"}}
            ],
            "minimum_should_match": "3<50%"
        }
    }
}
```



### Query string query

Returns documents based on a provided query string, using a parser with a strict syntax.

This query uses a [syntax](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-query-string-query.html#query-string-syntax) to parse and split the provided query string based on operators, such as `AND` or `NOT`. The query then [analyzes](https://www.elastic.co/guide/en/elasticsearch/reference/current/analysis.html) each split text independently before returning matching documents.

You can use the `query_string` query to create a complex search that includes wildcard characters, searches across multiple fields, and more. While versatile, the query is strict and returns an error if the query string includes any invalid syntax.

> Because it returns an error for any invalid syntax, we don’t recommend using the `query_string` query for search boxes.
>
> If you don’t need to support a query syntax, consider using the [`match`](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-match-query.html) query. If you need the features of a query syntax, use the [`simple_query_string`](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-simple-query-string-query.html) query, which is less strict.

### Example request

When running the following search, the `query_string` query splits `(new york city) OR (big apple)` into two parts: `new york city` and `big apple`. The `content` field’s analyzer then independently converts each part into tokens before returning matching documents. Because the query syntax does not use whitespace as an operator, `new york city` is passed as-is to the analyzer.

```json
GET /_search
{
    "query": {
        "query_string" : {
            "query" : "(new york city) OR (big apple)",
            "default_field" : "content"
        }
    }
}
```

#### Top-level parameters for query_string

##### query

(Required, string) Query string you wish to parse and use for search. See [Query string syntax](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-query-string-query.html#query-string-syntax).

##### default_field

(Optional, string) Default field you wish to search if no field is provided in the query string.

Defaults to the `index.query.default_field` index setting, which has a default value of `*`. The `*` value extracts all fields that are eligible to term queries and filters the metadata fields. All extracted fields are then combined to build a query if no `prefix` is specified.

> There is a limit on the number of fields that can be queried at once. It is defined by the `indices.query.bool.max_clause_count`[search setting](https://www.elastic.co/guide/en/elasticsearch/reference/current/search-settings.html), which defaults to 1024.

##### allow_leading_wildcard

(Optional, boolean) If `true`, the wildcard characters `*`and `?` are allowed as the first character of the query string. Defaults to `true`.

##### analyze_wildcard

(Optional, boolean) If `true`, the query attempts to analyze wildcard terms in the query string. Defaults to `false`.

##### analyzer

(Optional, string) [Analyzer](https://www.elastic.co/guide/en/elasticsearch/reference/current/analysis.html) used to convert text in the query string into tokens. Defaults to the [index-time analyzer](https://www.elastic.co/guide/en/elasticsearch/reference/current/specify-analyzer.html#specify-index-time-analyzer) mapped for the `default_field`. If no analyzer is mapped, the index’s default analyzer is used.

##### auto_generate_synoyms_phrase_query

(Optional, boolean) If `true`, [match phrase](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-match-query-phrase.html) queries are automatically created for multi-term synonyms. Defaults to `true`. See [Synonyms and the `query_string`query](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-query-string-query.html#query-string-synonyms) for an example.

##### boost

(Optional, float) Floating point number used to decrease or increase the [relevance scores](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-filter-context.html#relevance-scores) of the query. Defaults to `1.0`.

Boost values are relative to the default value of `1.0`. A boost value between `0` and `1.0` decreases the relevance score. A value greater than `1.0` increases the relevance score.

##### default_operator

(Optional, string) Default boolean logic used to interpret text in the query string if no operators are specified. Valid values are:

- **`OR` (Default)**

  For example, a query string of `capital of Hungary`is interpreted as `capital OR of OR Hungary`.

- **`AND`**

  For example, a query string of `capital of Hungary`is interpreted as `capital AND of AND Hungary`.

##### enable_position_increments

(Optional, boolean) If `true`, enable position increments in queries constructed from a `query_string` search. Defaults to `true`.

##### fields

(Optional, array of strings) Array of fields you wish to search.

You can use this parameter query to search across multiple fields. See [Search multiple fields](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-query-string-query.html#query-string-multi-field).

##### fuzziness

(Optional, string) Maximum edit distance allowed for matching. See [Fuzziness](https://www.elastic.co/guide/en/elasticsearch/reference/current/common-options.html#fuzziness) for valid values and more information.

##### fuzzy_max_expansions

(Optional, integer) Maximum number of terms to which the query expands for fuzzy matching. Defaults to `50`.

##### fuzzy_prefix_length

(Optional, integer) Number of beginning characters left unchanged for fuzzy matching. Defaults to `0`.

##### fuzzy_transpositions

(Optional, boolean) If `true`, edits for fuzzy matching include transpositions of two adjacent characters (ab → ba). Defaults to `true`.

##### lenient

(Optional, boolean) If `true`, format-based errors, such as providing a text value for a [numeric](https://www.elastic.co/guide/en/elasticsearch/reference/current/number.html) field, are ignored. Defaults to `false`.

##### max_determinized_states

(Optional, integer) Maximum number of [automaton states](https://en.wikipedia.org/wiki/Deterministic_finite_automaton) required for the query. Default is `10000`.

Elasticsearch uses [Apache Lucene](https://lucene.apache.org/core/) internally to parse regular expressions. Lucene converts each regular expression to a finite automaton containing a number of determinized states.

You can use this parameter to prevent that conversion from unintentionally consuming too many resources. You may need to increase this limit to run complex regular expressions.

##### minimum_should_match

(Optional, string) Minimum number of clauses that must match for a document to be returned. See the [`minimum_should_match` parameter](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-minimum-should-match.html) for valid values and more information. See [How `minimum_should_match`works](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-query-string-query.html#query-string-min-should-match) for an example.

##### quote_analyzer

 [Analyzer](https://www.elastic.co/guide/en/elasticsearch/reference/current/analysis.html) used to convert quoted text in the query string into tokens. Defaults to the [`search_quote_analyzer`](https://www.elastic.co/guide/en/elasticsearch/reference/current/analyzer.html#search-quote-analyzer) mapped for the `default_field`.

For quoted text, this parameter overrides the analyzer specified in the `analyzer` parameter.

##### phrase_slop

(Optional, integer) Maximum number of positions allowed between matching tokens for phrases. Defaults to `0`. If `0`, exact phrase matches are required. Transposed terms have a slop of `2`.

##### quote_field_suffix

Suffix appended to quoted text in the query string.

You can use this suffix to use a different analysis method for exact matches. See [Mixing exact search with stemming](https://www.elastic.co/guide/en/elasticsearch/reference/current/mixing-exact-search-with-stemming.html).

##### rewrite

(Optional, string) Method used to rewrite the query. For valid values and more information, see the [`rewrite`parameter](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-multi-term-rewrite.html).

##### time_zone

(Optional, string) [Coordinated Universal Time (UTC) offset](https://en.wikipedia.org/wiki/List_of_UTC_time_offsets) or [IANA time zone](https://en.wikipedia.org/wiki/List_of_tz_database_time_zones) used to convert `date` values in the query string to UTC.

Valid values are ISO 8601 UTC offsets, such as `+01:00`or -`08:00`, and IANA time zone IDs, such as `America/Los_Angeles`.

> The `time_zone` parameter does **not** affect the [date math](https://www.elastic.co/guide/en/elasticsearch/reference/current/common-options.html#date-math) value of `now`. `now` is always the current system time in UTC. However, the `time_zone` parameter does convert dates calculated using `now` and [date math rounding](https://www.elastic.co/guide/en/elasticsearch/reference/current/common-options.html#date-math). For example, the `time_zone` parameter will convert a value of `now/d`.

> https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-query-string-query.html



### Simple query string query

Returns documents based on a provided query string, using a parser with a limited but fault-tolerant syntax.

This query uses a [simple syntax](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-simple-query-string-query.html#simple-query-string-syntax) to parse and split the provided query string into terms based on special operators. The query then [analyzes](https://www.elastic.co/guide/en/elasticsearch/reference/current/analysis.html) each term independently before returning matching documents.

While its syntax is more limited than the [`query_string`query](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl-query-string-query.html), the `simple_query_string` query does not return errors for invalid syntax. Instead, it ignores any invalid parts of the query string.

```json
GET /_search
{
  "query": {
    "simple_query_string" : {
        "query": "\"fried eggs\" +(eggplant | potato) -frittata",
        "fields": ["title^5", "body"],
        "default_operator": "and"
    }
  }
}
```



### minimum_should_match parameter

The `minimum_should_match` parameter’s possible values:

| Type                  | Example       | Description                                                  |
| --------------------- | ------------- | ------------------------------------------------------------ |
| Integer               | `3`           | Indicates a fixed value regardless of the number of optional clauses. |
| Negative integer      | `-2`          | Indicates that the total number of optional clauses, minus this number should be mandatory. |
| Percentage            | `75%`         | Indicates that this percent of the total number of optional clauses are necessary. The number computed from the percentage is rounded down and used as the minimum. |
| Negative percentage   | `-25%`        | Indicates that this percent of the total number of optional clauses can be missing. The number computed from the percentage is rounded down, before being subtracted from the total to determine the minimum. |
| Combination           | `3<90%`       | A positive integer, followed by the less-than symbol, followed by any of the previously mentioned specifiers is a conditional specification. It indicates that if the number of optional clauses is equal to (or less than) the integer, they are all required, but if it’s greater than the integer, the specification applies. In this example: if there are 1 to 3 clauses they are all required, but for 4 or more clauses only 90% are required. |
| Multiple combinations | `2<-25% 9<-3` | Multiple conditional specifications can be separated by spaces, each one only being valid for numbers greater than the one before it. In this example: if there are 1 or 2 clauses both are required, if there are 3-9 clauses all but 25% are required, and if there are more than 9 clauses, all but three are required. |



# Search across cluster

**Cross-cluster search** lets you run a single search request against one or more [remote clusters](https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-remote-clusters.html). For example, you can use a cross-cluster search to filter and analyze log data stored on clusters in different data centers.

跨集群搜索使您可以针对一个或多个远程集群运行单个搜索请求。 例如，您可以使用跨集群搜索来过滤和分析存储在不同数据中心的集群中的日志数据。

## Supported APIs

- [Search](https://www.elastic.co/guide/en/elasticsearch/reference/current/search-search.html)
- [Multi search](https://www.elastic.co/guide/en/elasticsearch/reference/current/search-multi-search.html)
- [Search template](https://www.elastic.co/guide/en/elasticsearch/reference/current/search-template.html)
- [Multi search template](https://www.elastic.co/guide/en/elasticsearch/reference/current/multi-search-template.html)

## Cross-cluster search examples

### Remote cluster setup

To perform a cross-cluster search, you must have at least one remote cluster configured.

The following [cluster update settings](https://www.elastic.co/guide/en/elasticsearch/reference/current/cluster-update-settings.html) API request adds three remote clusters:`cluster_one`, `cluster_two`, and `cluster_three`.

```json
PUT _cluster/settings
{
  "persistent": {
    "cluster": {
      "remote": {
        "cluster_one": {
          "seeds": [
            "127.0.0.1:9300"
          ]
        },
        "cluster_two": {
          "seeds": [
            "127.0.0.1:9301"
          ]
        },
        "cluster_three": {
          "seeds": [
            "127.0.0.1:9302"
          ]
        }
      }
    }
  }
}
```

### Search a single remote cluster

The following [search](https://www.elastic.co/guide/en/elasticsearch/reference/current/search-search.html) API request searches the `twitter`index on a single remote cluster, `cluster_one`.

```json
GET /cluster_one:twitter/_search
{
  "query": {
    "match": {
      "user": "kimchy"
    }
  }
}
```

The API returns the following response:

```json
{
  "took": 150,
  "timed_out": false,
  "_shards": {
    "total": 1,
    "successful": 1,
    "failed": 0,
    "skipped": 0
  },
  "_clusters": {
    "total": 1,
    "successful": 1,
    "skipped": 0
  },
  "hits": {
    "total" : {
        "value": 1,
        "relation": "eq"
    },
    "max_score": 1,
    "hits": [
      {
        "_index": "cluster_one:twitter", 
        "_type": "_doc",
        "_id": "0",
        "_score": 1,
        "_source": {
          "user": "kimchy",
          "date": "2009-11-15T14:12:12",
          "message": "trying out Elasticsearch",
          "likes": 0
        }
      }
    ]
  }
}
```

### Search multiple remote clusters

The following [search](https://www.elastic.co/guide/en/elasticsearch/reference/current/search.html) API request searches the `twitter`index on three clusters:

- Your local cluster
- Two remote clusters, `cluster_one` and `cluster_two`

```json
GET /twitter,cluster_one:twitter,cluster_two:twitter/_search
{
  "query": {
    "match": {
      "user": "kimchy"
    }
  }
}
```

The API returns the following response:

```json
{
  "took": 150,
  "timed_out": false,
  "num_reduce_phases": 4,
  "_shards": {
    "total": 3,
    "successful": 3,
    "failed": 0,
    "skipped": 0
  },
  "_clusters": {
    "total": 3,
    "successful": 3,
    "skipped": 0
  },
  "hits": {
    "total" : {
        "value": 3,
        "relation": "eq"
    },
    "max_score": 1,
    "hits": [
      {
        "_index": "twitter", 
        "_type": "_doc",
        "_id": "0",
        "_score": 2,
        "_source": {
          "user": "kimchy",
          "date": "2009-11-15T14:12:12",
          "message": "trying out Elasticsearch",
          "likes": 0
        }
      },
      {
        "_index": "cluster_one:twitter", 
        "_type": "_doc",
        "_id": "0",
        "_score": 1,
        "_source": {
          "user": "kimchy",
          "date": "2009-11-15T14:12:12",
          "message": "trying out Elasticsearch",
          "likes": 0
        }
      },
      {
        "_index": "cluster_two:twitter", 
        "_type": "_doc",
        "_id": "0",
        "_score": 1,
        "_source": {
          "user": "kimchy",
          "date": "2009-11-15T14:12:12",
          "message": "trying out Elasticsearch",
          "likes": 0
        }
      }
    ]
  }
}
```

 This document’s `_index` parameter doesn’t include a cluster name. This means the document came from the local cluster.

This document came from `cluster_one`.

This document came from `cluster_two`.

### Skip unavailable clusters

By default, a cross-cluster search returns an error if **any**cluster in the request is unavailable.

To skip an unavailable cluster during a cross-cluster search, set the [`skip_unavailable`](https://www.elastic.co/guide/en/elasticsearch/reference/current/cluster-remote-info.html#skip-unavailable) cluster setting to `true`.

The following [cluster update settings](https://www.elastic.co/guide/en/elasticsearch/reference/current/cluster-update-settings.html) API request changes `cluster_two`'s `skip_unavailable` setting to `true`.

```json
PUT _cluster/settings
{
  "persistent": {
    "cluster.remote.cluster_two.skip_unavailable": true
  }
}
```

If `cluster_two` is disconnected or unavailable during a cross-cluster search, Elasticsearch won’t include matching documents from that cluster in the final results.

## How cross-cluster search works

Remote cluster connections work by configuring a remote cluster and connecting only to a limited number of nodes in that remote cluster. Each remote cluster is referenced by a name and a list of seed nodes. When a remote cluster is registered, its cluster state is retrieved from one of the seed nodes and up to three *gateway nodes* are selected to be connected to as part of remote cluster requests.