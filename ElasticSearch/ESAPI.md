# TermVectorsRequest

elasticsearch的termVectors包括了term的位置，词频等信息，这些信息用于相应的数据统计或开发其他功能。

要使用termVectors首先要配置mapping中field的"term_vector"属性，默认状态es不启用termvector，因为这样会增加索引的体积，多存了不少的元数据

```json
PUT qa_test_02      //create an index called qa_test_02
{
  "mappings": {
    "qa_test": {    //add a mapping type called qa_test
      "dynamic": "strict",
      "_all": {
        "enabled": false
      },
      "properties": {   //specify fields or properties
        "question": {   //specify that the question properties
          "properties": { //specify fields or properties
            "cate": {     //specify that the cate field contains keyword
              "type": "keyword"
            },
            "desc": {
              "type": "text",
              "store": true,
              "term_vector": "with_positions_offsets_payloads",
              "analyzer": "ik_smart"
            },
            "time": {
              "type": "date",
              "store": true,
              "format": "strict_date_optional_time||epoch_millis||yyyy-MM-dd HH:mm:ss"
            },
            "title": {
              "type": "text",
              "store": true,
              "term_vector": "with_positions_offsets_payloads",
              "analyzer": "ik_smart"
            }
          }
        },
        "updatetime": {
          "type": "date",
          "store": true,
          "format": "strict_date_optional_time||epoch_millis||yyyy-MM-dd HH:mm:ss"
        }
      }
    }
  },
  "settings": {
    "index": {
      "number_of_shards": "1",
      "requests": {
        "cache": {
          "enable": "true"
        }
      },
      "number_of_replicas": "1"
    }
  }
}
```

注意示例中的"title"的"term_vector"属性

接下来为索引创建一条数据

```json
PUT qa_test_02/qa_test/1
{
  "question": {
    "cate": [
      "装修流程",
      "其它"
    ],
    "desc": "筒灯，大洋和索正这两个牌子，哪个好？希望内行的朋友告知一下，谢谢！",
    "time": "2016-07-02 19:59:00",
    "title": "筒灯大洋和索正这两个牌子哪个好"
  },
  "updatetime": 1467503940000
}
```

下面我们看看这条数据上question.title字段的termvector信息

```json
GET qa_test_02/qa_test/1/_termvectors
{
  "fields": [
    "question.title"
  ],
  "offsets": true,
  "payloads": true,
  "positions": true,
  "term_statistics": true,
  "field_statistics": true
}
```

结果大概这个样子

```json
{
  "_index": "qa_test_02",
  "_type": "qa_test",
  "_id": "1",
  "_version": 1,
  "found": true,
  "took": 0,
  "term_vectors": {
    "question.title": {
      "field_statistics": {
        "sum_doc_freq": 9,
        "doc_count": 1,
        "sum_ttf": 9
      },
      "terms": {
        "和": {
          "doc_freq": 1,
          "ttf": 1,
          "term_freq": 1,
          "tokens": [
            {
              "position": 2,
              "start_offset": 4,
              "end_offset": 5
            }
          ]
        },
        "哪个": {
          "doc_freq": 1,
          "ttf": 1,
          "term_freq": 1,
          "tokens": [
            {
              "position": 7,
              "start_offset": 12,
              "end_offset": 14
            }
          ]
        },
        "大洋": {
          "doc_freq": 1,
          "ttf": 1,
          "term_freq": 1,
          "tokens": [
            {
              "position": 1,
              "start_offset": 2,
              "end_offset": 4
            }
          ]
        },
        "好": {
          "doc_freq": 1,
          "ttf": 1,
          "term_freq": 1,
          "tokens": [
            {
              "position": 8,
              "start_offset": 14,
              "end_offset": 15
            }
          ]
        },
        "正": {
          "doc_freq": 1,
          "ttf": 1,
          "term_freq": 1,
          "tokens": [
            {
              "position": 4,
              "start_offset": 6,
              "end_offset": 7
            }
          ]
        },
        "牌子": {
          "doc_freq": 1,
          "ttf": 1,
          "term_freq": 1,
          "tokens": [
            {
              "position": 6,
              "start_offset": 10,
              "end_offset": 12
            }
          ]
        },
        "筒灯": {
          "doc_freq": 1,
          "ttf": 1,
          "term_freq": 1,
          "tokens": [
            {
              "position": 0,
              "start_offset": 0,
              "end_offset": 2
            }
          ]
        },
        "索": {
          "doc_freq": 1,
          "ttf": 1,
          "term_freq": 1,
          "tokens": [
            {
              "position": 3,
              "start_offset": 5,
              "end_offset": 6
            }
          ]
        },
        "这两个": {
          "doc_freq": 1,
          "ttf": 1,
          "term_freq": 1,
          "tokens": [
            {
              "position": 5,
              "start_offset": 7,
              "end_offset": 10
            }
          ]
        }
      }
    }
  }
}
```

通过java代码实现termvector的获取

```java
 TermVectorsResponse     termVectorResponse = client.prepareTermVectors().setIndex(sourceindexname).setType(sourceindextype)               .setId(id).setSelectedFields(fieldname).setTermStatistics(true).execute()
                        .actionGet();
                XContentBuilder builder = XContentFactory.contentBuilder(XContentType.JSON);
                termVectorResponse.toXContent(builder, null);
                System.out.println(builder.string());
                Fields fields = termVectorResponse.getFields();
                Iterator<String> iterator = fields.iterator();
                while (iterator.hasNext()) {
                    String field = iterator.next();
                    Terms terms = fields.terms(field);
                    TermsEnum termsEnum = terms.iterator();
                    while (termsEnum.next() != null) {
                        BytesRef term = termsEnum.term();
                        if (term != null) {
                            System.out.println(term.utf8ToString() + termsEnum.totalTermFreq());
                        }
                    }
                }
```

获取TermVectorsResponse的代码很好理解，主要是设置索引名称、索引type、索引id以及需要展示的若干属性。

## term的基本介绍

- term_freq：term在该字段中的频率
- position：term在该字段中的位置
- start_offset：从什么偏移量开始
- end_offset：到什么偏移量结束

## term的统计信息

如果启用了term的统计信息，即term_statistic设置为true

doc_freq：该词在文档中出现的频率

ttf：total term frequency的缩写，一个term在所有document中出现的频率

## 字段的统计信息

如果启用了字段统计信息，即field_statistics设为true

- sum_doc_freq：一个field中所有term的df之和
- doc_count：有多少个文档包含这个字段
- sum_ttf：sum total term frequency的缩写，一个field中的所有term的tf之和
- term statistics和field statistics并不精准，不会考虑有的doc可能已经被删除了

## 采集term信息的方式

采集term信息的方式有两种，index-time 和 query-time

### index-time方式

需要在mapping配置一下，然后建立索引的时候，就直接生成这些词条和文档的统计信息

//TODO



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



# Meta-Fields

Each document has metadata associated with it, such as the `_index`, mapping [`_type`](https://www.elastic.co/guide/en/elasticsearch/reference/6.8/mapping-type-field.html), and `_id` meta-fields. The behaviour of some of these meta-fields can be customised when a mapping type is created.

### Identity meta-fields

| [`_index`](https://www.elastic.co/guide/en/elasticsearch/reference/6.8/mapping-index-field.html) | The index to which the document belongs.                     |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| [`_uid`](https://www.elastic.co/guide/en/elasticsearch/reference/6.8/mapping-uid-field.html) | A composite field consisting of the `_type` and the `_id`.   |
| [`_type`](https://www.elastic.co/guide/en/elasticsearch/reference/6.8/mapping-type-field.html) | The document’s [mapping type](https://www.elastic.co/guide/en/elasticsearch/reference/6.8/mapping.html#mapping-type). |
| [`_id`](https://www.elastic.co/guide/en/elasticsearch/reference/6.8/mapping-id-field.html) | The document’s ID.                                           |

### Document source meta-fields

[`_source`](https://www.elastic.co/guide/en/elasticsearch/reference/6.8/mapping-source-field.html)

​	The original JSON representing the body of the document.

[`_size`](https://www.elastic.co/guide/en/elasticsearch/plugins/6.8/mapper-size.html)

​	The size of the `_source` field in bytes, provided by the [`mapper-size` plugin](https://www.elastic.co/guide/en/elasticsearch/plugins/6.8/mapper-size.html).

### Indexing meta-fields

**[`_all`](https://www.elastic.co/guide/en/elasticsearch/reference/6.8/mapping-all-field.html)**

A *catch-all* field that indexes the values of all other fields. Disabled by default.

**[`_field_names`](https://www.elastic.co/guide/en/elasticsearch/reference/6.8/mapping-field-names-field.html)**

All fields in the document which contain non-null values.

**[`_ignored`](https://www.elastic.co/guide/en/elasticsearch/reference/6.8/mapping-ignored-field.html)**

All fields in the document that have been ignored at index time because of [`ignore_malformed`](https://www.elastic.co/guide/en/elasticsearch/reference/6.8/ignore-malformed.html).

### Routing meta-field

**[`_routing`](https://www.elastic.co/guide/en/elasticsearch/reference/6.8/mapping-routing-field.html)**

A custom routing value which routes a document to a particular shard.

### Other meta-field

**[`_meta`](https://www.elastic.co/guide/en/elasticsearch/reference/6.8/mapping-meta-field.html)**

Application specific metadata.



## _all field

_all字段是一个特殊的通用字段，它将所有其他字段的值连接到一个大字符串中，使用空格作为分隔符，然后对其进行分析和索引，但不进行存储。 这意味着它可以被搜索，但不能被检索。

_all字段使您可以在文档中搜索值，而无需知道哪个字段包含该值。 在开始使用新数据集时，这使其成为有用的选项。 例如：

```json
PUT /my_index
{
  "mapping": {
    "user": {
      "_all": {
        "enabled": true   
      }
    }
  }
}

PUT /my_index/user/1      
{
  "first_name":    "John",
  "last_name":     "Smith",
  "date_of_birth": "1970-10-24"
}

GET /my_index/_search
{
  "query": {
    "match": {
      "_all": "john smith 1970"
    }
  }
}
```

### Deprecated in 6.0.0.

_all可能不再为在6.0+中创建的索引启用，请使用自定义字段和映射copy_to参数



### Disabling _field_names

_field_names已被弃用，并将在以后的主要版本中删除。

### _ignored field

Added in 6.4.0.

