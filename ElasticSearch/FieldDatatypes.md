# Field datatypes

Elasticsearch supports a number of different datatypes for the fields in a document:

## Core datatypes

### string

text and keyword

### Numeric datatypes

long, integer, short, byte, double, float, half_float, scaled_float

### Date datatype

data

### Boolean datatype

boolean

### Binary datatype

binary

### Range datatypes

integer_range, float_range, long_range, double_range, data_range

## Complex datatypes

### Object datatypes

object for single JSON objecs

### Nested datatype

nested for arrays of JSON Objects

## Geo datatypes

### Geo-point datatype

geo_point for lat/lon points

### Geo-shape datatype

geo_shape for complex shapes like polygons

## Specialised datatypes

### IP datatype

ip for IPV4 and IPV6 address

### Completion datatype

completion to provide auto-complete suggestions

### Token count datatype

token_count to count the number of tokens in a string

### mapper-murmur3

murmur3 to compute hashes of values at index-time and store them in the index

### mapper-annotated-text

annotated-text to index text containing special markup(typically used for identifying named entities).

### Percolator type

Accepts queries from the query-dsl

### join datatype

Defines parent/child relation for documents within the same index

### Alias datatype

Defines an alias to an existing field

## Arrays

在Elasticsearch中，数组不需要专用的字段数据类型。 默认情况下，任何字段都可以包含零个或多个值，但是，数组中的所有值都必须具有相同的数据类型。 请参阅数组。

## Multi-fields

为不同的目的以不同的方式为同一字段建立索引通常很有用。 例如，可以将字符串字段映射为用于全文搜索的文本字段，并映射为用于排序或聚合的关键字字段。 另外，您可以使用标准分析仪，英语分析仪和法语分析仪为文本字段建立索引。

This is the purpose of *multi-fields*. Most datatypes support multi-fields via the [`fields`](https://www.elastic.co/guide/en/elasticsearch/reference/6.8/multi-fields.html) parameter.

## Alias datatype

An `alias` mapping defines an alternate name for a field in the index. The alias can be used in place of the target field in [search](https://www.elastic.co/guide/en/elasticsearch/reference/6.8/search.html) requests, and selected other APIs like [field capabilities](https://www.elastic.co/guide/en/elasticsearch/reference/6.8/search-field-caps.html).

```json
PUT trips
{
  "mappings": {
    "_doc": {
      "properties": {
        "distance": {
          "type": "long"
        },
        "route_length_miles": {
          "type": "alias",
          "path": "distance" 
        },
        "transit_mode": {
          "type": "keyword"
        }
      }
    }
  }
}

GET _search
{
  "query": {
    "range" : {
      "route_length_miles" : {
        "gte" : 39
      }
    }
  }
}
```

**The path to the target field. Note that this must be the full path, including any parent objects (e.g. object1.object2.field).**

搜索请求的几乎所有组件都接受字段别名。 特别是，别名可用于查询，聚合和排序字段，以及在请求docvalue_fields，stored_fields，建议和突出显示时使用。 访问字段值时，脚本还支持别名。 请参阅关于不支持的API的部分以了解例外情况。Please see the section on [unsupported APIs](https://www.elastic.co/guide/en/elasticsearch/reference/6.8/alias.html#unsupported-apis) for exceptions.

在搜索请求的某些部分以及请求字段功能时，可以提供字段通配符模式。 在这些情况下，通配符模式除了具体字段外还将匹配字段别名：

```json
GET trips/_field_caps?fields=route_*,transit_mode
```

### Alias targets

There are a few **restrictions** on the target of an alias:

- The target must be a concrete field, and not an object or another field alias.

- The target field must exist at the time the alias is created.

- If nested objects are defined, a field alias must have the same nested **scope** as its target.

Additionally, a field alias can only have one target. This means that it is not possible to use a field alias to query over multiple target fields in a single clause.

An alias can be changed to refer to a new target through a mappings update. A known limitation is that if any stored percolator queries contain the field alias, they will still refer to its original target. More information can be found in the [percolator documentation](https://www.elastic.co/guide/en/elasticsearch/reference/6.8/percolator.html).

### Unsupported APIs

Writes to field aliases are not supported: attempting to use an alias in an index or update request will result in a failure. Likewise, aliases cannot be used as the target of `copy_to` or in multi-fields.

Because alias names are not present in the document source, aliases cannot be used when performing source filtering. For example, the following request will return an empty result for `_source`:

```json
GET /_search
{
  "query" : {
    "match_all": {}
  },
  "_source": "route_length_miles"
}
```

Currently only the search and field capabilities APIs will accept and resolve field aliases. Other APIs that accept field names, such as [term vectors](https://www.elastic.co/guide/en/elasticsearch/reference/6.8/docs-termvectors.html), cannot be used with field aliases.

## Arrays

In Elasticsearch, there is no **dedicated** `array` datatype. Any field can contain zero or more values by default, however, all values in the array **must be of the same datatype**. For instance:

- an array of strings: [ `"one"`, `"two"` ]
- an array of integers: [ `1`, `2` ]
- an array of arrays: [ `1`, [ `2`, `3` ]] which is the equivalent of [ `1`, `2`, `3` ]
- an array of objects: [ `{ "name": "Mary", "age": 12 }`, `{ "name": "John", "age": 10 }`]

> 对象数组无法按预期工作：无法独立于数组中的其他对象查询每个对象。 如果需要执行此操作，则应使用嵌套数据类型而不是对象数据类型。This is explained in more detail in [Nested datatype](https://www.elastic.co/guide/en/elasticsearch/reference/6.8/nested.html).

动态添加字段时，数组中的第一个值确定字段类型。 所有后续值必须具有相同的数据类型，或者至少必须能够将后续值强制转换为相同的数据类型。

Arrays with a mixture of datatypes are *not* supported: [ `10`, `"some string"` ]

Nothing needs to be pre-configured in order to use arrays in documents, they are supported out of the box:

```json
PUT my_index/_doc/1
{
  "message": "some arrays in this document...",
  "tags":  [ "elasticsearch", "wow" ], 
  "lists": [ 
    {
      "name": "prog_list",
      "description": "programming list"
    },
    {
      "name": "cool_list",
      "description": "cool stuff list"
    }
  ]
}

PUT my_index/_doc/2 
{
  "message": "no arrays in this document...",
  "tags":  "elasticsearch",
  "lists": {
    "name": "prog_list",
    "description": "programming list"
  }
}

GET my_index/_search
{
  "query": {
    "match": {
      "tags": "elasticsearch" 
    }
  }
}
```

The `tags` field is dynamically added as a `string` field.

The `lists` field is dynamically added as an `object` field.

The second document contains no arrays, but can be indexed into the same fields.

The query looks for `elasticsearch` in the `tags` field, and matches both documents.

## Binary datatype

The `binary` type accepts a binary value as a [Base64](https://en.wikipedia.org/wiki/Base64) encoded string. The field is not stored by default and is not searchable:

```json
PUT my_index
{
  "mappings": {
    "_doc": {
      "properties": {
        "name": {
          "type": "text"
        },
        "blob": {
          "type": "binary"
        }
      }
    }
  }
}

PUT my_index/_doc/1
{
  "name": "Some binary blob",
  "blob": "U29tZSBiaW5hcnkgYmxvYg==" 
}
```

The Base64 encoded binary value must not have embedded newlines `\n`.



## Range datatypes

The following range types are supported:

### integer_range

A range of signed 32-bit integers with a minimum value of -2^{31} and  and maximum of `2^{31}-1`.



### float_range

A range of single-precision 32-bit IEEE 754 floating point values.



### long_range

A range of signed 64-bit integers with a minimum value of `-2^{63}` and maximum of `2^{63}-1`.

### double_range

A range of double-precision 64-bit IEEE 754 floating point values.

### data_range

A range of date values represented as unsigned 64-bit integer milliseconds elapsed since system epoch.

### ip_range

A range of ip values supporting either [IPv4](https://en.wikipedia.org/wiki/IPv4) or [IPv6](https://en.wikipedia.org/wiki/IPv6) (or mixed) addresses.

Below is an example of configuring a mapping with various range fields followed by an example that indexes several range types:

```json
PUT range_index
{
  "settings": {
    "number_of_shards": 2
  },
  "mappings": {
    "_doc": {
      "properties": {
        "expected_attendees": {
          "type": "integer_range"
        },
        "time_frame": {
          "type": "date_range", 
          "format": "yyyy-MM-dd HH:mm:ss||yyyy-MM-dd||epoch_millis"
        }
      }
    }
  }
}

PUT range_index/_doc/1?refresh
{
  "expected_attendees" : { 
    "gte" : 10,
    "lte" : 20
  },
  "time_frame" : { 
    "gte" : "2015-10-31 12:00:00", 
    "lte" : "2015-11-01"
  }
}
```

`date_range` types accept the same field parameters defined by the [`date`](https://www.elastic.co/guide/en/elasticsearch/reference/6.8/date.html)type.

Example indexing a meeting with 10 to 20 attendees.

Date ranges accept the same format as described in [date range queries](https://www.elastic.co/guide/en/elasticsearch/reference/6.8/query-dsl-range-query.html#ranges-on-dates).

Example date range using date time stamp. This also accepts [date math](https://www.elastic.co/guide/en/elasticsearch/reference/6.8/common-options.html#date-math)formatting. Note that "now" cannot be used at indexing time.



The following is an example of a [term query](https://www.elastic.co/guide/en/elasticsearch/reference/6.8/query-dsl-term-query.html) on the `integer_range` field named "expected_attendees".

```json
GET range_index/_search
{
  "query" : {
    "term" : {
      "expected_attendees" : {
        "value": 12
      }
    }
  }
}
```



The result produced by the above query.

```json
{
  "took": 13,
  "timed_out": false,
  "_shards" : {
    "total": 2,
    "successful": 2,
    "skipped" : 0,
    "failed": 0
  },
  "hits" : {
    "total" : 1,
    "max_score" : 1.0,
    "hits" : [
      {
        "_index" : "range_index",
        "_type" : "_doc",
        "_id" : "1",
        "_score" : 1.0,
        "_source" : {
          "expected_attendees" : {
            "gte" : 10, "lte" : 20
          },
          "time_frame" : {
            "gte" : "2015-10-31 12:00:00", "lte" : "2015-11-01"
          }
        }
      }
    ]
  }
}
```

The following is an example of a `date_range` query over the `date_range` field named "time_frame".

```json
GET range_index/_search
{
  "query" : {
    "range" : {
      "time_frame" : { 
        "gte" : "2015-10-31",
        "lte" : "2015-11-01",
        "relation" : "within" 
      }
    }
  }
}
```

Range queries work the same as described in [range query](https://www.elastic.co/guide/en/elasticsearch/reference/6.8/query-dsl-range-query.html).

Range queries over range [fields](https://www.elastic.co/guide/en/elasticsearch/reference/6.8/mapping-types.html) support a `relation` parameter which can be one of `WITHIN`, `CONTAINS`, `INTERSECTS` (default).

This query produces a similar result:

```json
{
  "took": 13,
  "timed_out": false,
  "_shards" : {
    "total": 2,
    "successful": 2,
    "skipped" : 0,
    "failed": 0
  },
  "hits" : {
    "total" : 1,
    "max_score" : 1.0,
    "hits" : [
      {
        "_index" : "range_index",
        "_type" : "_doc",
        "_id" : "1",
        "_score" : 1.0,
        "_source" : {
          "expected_attendees" : {
            "gte" : 10, "lte" : 20
          },
          "time_frame" : {
            "gte" : "2015-10-31 12:00:00", "lte" : "2015-11-01"
          }
        }
      }
    ]
  }
}
```

### IP range

In addition to the range format above, IP ranges can be provided in [CIDR](https://en.wikipedia.org/wiki/Classless_Inter-Domain_Routing#CIDR_notation)notation:

```json
PUT range_index/_mapping/_doc
{
  "properties": {
    "ip_whitelist": {
      "type": "ip_range"
    }
  }
}

PUT range_index/_doc/2
{
  "ip_whitelist" : "192.168.0.0/16"
}
```



## Boolean datatype

Boolean fields accept JSON `true` and `false` values, but can also accept strings which are interpreted as either true or false:

For example:

```json
PUT my_index
{
  "mappings": {
    "_doc": {
      "properties": {
        "is_published": {
          "type": "boolean"
        }
      }
    }
  }
}

POST my_index/_doc/1
{
  "is_published": "true" 
}

GET my_index/_search
{
  "query": {
    "term": {
      "is_published": true 
    }
  }
}
```

Aggregations like the [`terms` aggregation](https://www.elastic.co/guide/en/elasticsearch/reference/6.8/search-aggregations-bucket-terms-aggregation.html) use `1` and `0` for the `key`, and the strings `"true"` and `"false"` for the `key_as_string`. Boolean fields when used in scripts, return `1` and `0`:

```json
POST my_index/_doc/1
{
  "is_published": true
}

POST my_index/_doc/2
{
  "is_published": false
}

GET my_index/_search
{
  "aggs": {
    "publish_state": {
      "terms": {
        "field": "is_published"
      }
    }
  },
  "script_fields": {
    "is_published": {
      "script": {
        "lang": "painless",
        "source": "doc['is_published'].value"
      }
    }
  }
}
```

### Date datatype

```json
PUT my_index
{
  "mappings": {
    "_doc": {
      "properties": {
        "date": {
          "type": "date" 
        }
      }
    }
  }
}

PUT my_index/_doc/1
{ "date": "2015-01-01" } 

PUT my_index/_doc/2
{ "date": "2015-01-01T12:10:30Z" } 

PUT my_index/_doc/3
{ "date": 1420070400001 } 

GET my_index/_search
{
  "sort": { "date": "asc"} 
}
```

### Multiple date formats

Multiple formats can be specified by separating them with `||` as a separator. Each format will be tried in turn until a matching format is found. The first format will be used to convert the *milliseconds-since-the-epoch* value back into a string.

```json
PUT my_index
{
  "mappings": {
    "_doc": {
      "properties": {
        "date": {
          "type":   "date",
          "format": "yyyy-MM-dd HH:mm:ss||yyyy-MM-dd||epoch_millis"
        }
      }
    }
  }
}
```

