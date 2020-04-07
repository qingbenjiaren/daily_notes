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