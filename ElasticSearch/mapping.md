# Mapping

Mapping is the process of defining how a document, and the fields it contains, are stored and indexed. For instance, use mappings to define:

- which string fields should be treated as full text fields.
- which fields contain numbers, dates, or geolocations.
- the [format](https://www.elastic.co/guide/en/elasticsearch/reference/current/mapping-date-format.html) of date values.
- custom rules to control the mapping for [dynamically added fields](https://www.elastic.co/guide/en/elasticsearch/reference/current/dynamic-mapping.html).

A mapping definition has:

**Meta-field**

Meta-fields are used to customize how a document’s metadata associated is treated. Examples of meta-fields include the document’s [`_index`](https://www.elastic.co/guide/en/elasticsearch/reference/current/mapping-index-field.html), [`_id`](https://www.elastic.co/guide/en/elasticsearch/reference/current/mapping-id-field.html), and [`_source`](https://www.elastic.co/guide/en/elasticsearch/reference/current/mapping-source-field.html) fields.

**Field or properties**

A mapping contains a list of fields or `properties`pertinent to the document.



## Field datatypes

详见datatypes文档

Each field has a data `type` which can be:

- a simple type like [`text`](https://www.elastic.co/guide/en/elasticsearch/reference/current/text.html), [`keyword`](https://www.elastic.co/guide/en/elasticsearch/reference/current/keyword.html), [`date`](https://www.elastic.co/guide/en/elasticsearch/reference/current/date.html), [`long`](https://www.elastic.co/guide/en/elasticsearch/reference/current/number.html), [`double`](https://www.elastic.co/guide/en/elasticsearch/reference/current/number.html), [`boolean`](https://www.elastic.co/guide/en/elasticsearch/reference/current/boolean.html) or [`ip`](https://www.elastic.co/guide/en/elasticsearch/reference/current/ip.html).
- a type which supports the hierarchical nature of JSON such as [`object`](https://www.elastic.co/guide/en/elasticsearch/reference/current/object.html) or [`nested`](https://www.elastic.co/guide/en/elasticsearch/reference/current/nested.html).
- or a specialised type like [`geo_point`](https://www.elastic.co/guide/en/elasticsearch/reference/current/geo-point.html), [`geo_shape`](https://www.elastic.co/guide/en/elasticsearch/reference/current/geo-shape.html), or [`completion`](https://www.elastic.co/guide/en/elasticsearch/reference/current/search-suggesters.html#completion-suggester).

It is often useful to index the same field in different ways for different purposes. For instance, a `string` field could be [indexed](https://www.elastic.co/guide/en/elasticsearch/reference/current/mapping-index.html) as a `text` field for full-text search, and as a `keyword` field for sorting or aggregations. Alternatively, you could index a string field with the [`standard` analyzer](https://www.elastic.co/guide/en/elasticsearch/reference/current/analysis-standard-analyzer.html), the [`english`](https://www.elastic.co/guide/en/elasticsearch/reference/current/analysis-lang-analyzer.html#english-analyzer) analyzer, and the [`french` analyzer](https://www.elastic.co/guide/en/elasticsearch/reference/current/analysis-lang-analyzer.html#french-analyzer).

This is the purpose of *multi-fields*. Most datatypes support multi-fields via the [`fields`](https://www.elastic.co/guide/en/elasticsearch/reference/current/multi-fields.html) parameter.

### Settings to prevent mappings explosion

Defining too many fields in an index is a condition that can lead to a mapping explosion, which can cause out of memory errors and difficult situations to recover from. This problem may be more common than expected. As an example, consider a situation in which every new document inserted introduces new fields. This is quite common with dynamic mappings. Every time a document contains new fields, those will end up in the index’s mappings. This isn’t worrying for a small amount of data, but it can become a problem as the mapping grows. The following settings allow you to limit the number of field mappings that can be created manually or dynamically, in order to prevent bad documents from causing a mapping explosion:

**`index.mapping.total_fields.limit`**

The maximum number of fields in an index. Field and object mappings, as well as field aliases count towards this limit. The default value is `1000`.

**`index.mapping.depth.limit`**

The maximum depth for a field, which is measured as the number of inner objects. For instance, if all fields are defined at the root object level, then the depth is `1`. If there is one object mapping, then the depth is `2`, etc. The default is `20`.

**`index.mapping.nested_fields.limit`**

The maximum number of distinct `nested` mappings in an index, defaults to `50`.

**`index.mapping.nested_objects.limit`**

The maximum number of `nested` JSON objects within a single document across all nested types, defaults to 10000.

**`index.mapping.field_name_length.limit`**

Setting for the maximum length of a field name. The default value is Long.MAX_VALUE (no limit). This setting isn’t really something that addresses mappings explosion but might still be useful if you want to limit the field length. It usually shouldn’t be necessary to set this setting. The default is okay unless a user starts to add a huge number of fields with really long names.

## Dynamic mapping

Fields and mapping types do not need to be defined before being used. Thanks to *dynamic mapping*, new field names will be added automatically, just by indexing a document. New fields can be added both to the top-level mapping type, and to inner [`object`](https://www.elastic.co/guide/en/elasticsearch/reference/current/object.html) and [`nested`](https://www.elastic.co/guide/en/elasticsearch/reference/current/nested.html) fields.

The [dynamic mapping](https://www.elastic.co/guide/en/elasticsearch/reference/current/dynamic-mapping.html) rules can be configured to customise the mapping that is used for new fields.

## Explicit mappings

You know more about your data than Elasticsearch can guess, so while dynamic mapping can be useful to get started, at some point you will want to specify your own explicit mappings.

You can create field mappings when you [create an index](https://www.elastic.co/guide/en/elasticsearch/reference/current/mapping.html#create-mapping)and [add fields to an existing index](https://www.elastic.co/guide/en/elasticsearch/reference/current/mapping.html#add-field-mapping).

## Create an index with an explicit mapping

You can use the [create index](https://www.elastic.co/guide/en/elasticsearch/reference/current/indices-create-index.html) API to create a new index with an explicit mapping.

```json
PUT /my-index
{
  "mappings": {
    "properties": {
      "age":    { "type": "integer" },  
      "email":  { "type": "keyword"  }, 
      "name":   { "type": "text"  }     
    }
  }
}
```

Creates `age`, an [`integer`](https://www.elastic.co/guide/en/elasticsearch/reference/current/number.html) field

Creates `email`, a [`keyword`](https://www.elastic.co/guide/en/elasticsearch/reference/current/keyword.html) field

Creates `name`, a [`text`](https://www.elastic.co/guide/en/elasticsearch/reference/current/text.html) field

## Add a field to an existing mapping

You can use the [put mapping](https://www.elastic.co/guide/en/elasticsearch/reference/current/indices-put-mapping.html) API to add one or more new fields to an existing index.

The following example adds `employee-id`, a `keyword` field with an [`index`](https://www.elastic.co/guide/en/elasticsearch/reference/current/mapping-index.html) mapping parameter value of `false`. This means values for the `employee-id` field are stored but not indexed or available for search.

```json
PUT /my-index/_mapping
{
  "properties": {
    "employee-id": {
      "type": "keyword",
      "index": false
    }
  }
}
```

## Update the mapping of a field

Except for supported [mapping parameters](https://www.elastic.co/guide/en/elasticsearch/reference/current/mapping-params.html), you can’t change the mapping or field type of an existing field. Changing an existing field could invalidate data that’s already indexed.

If you need to change the mapping of a field, create a new index with the correct mapping and [reindex](https://www.elastic.co/guide/en/elasticsearch/reference/current/docs-reindex.html) your data into that index.

Renaming a field would invalidate data already indexed under the old field name. Instead, add an [`alias`](https://www.elastic.co/guide/en/elasticsearch/reference/current/alias.html) field to create an alternate field name.

## View the mapping of an index

You can use the [get mapping](https://www.elastic.co/guide/en/elasticsearch/reference/current/indices-get-mapping.html) API to view the mapping of an existing index.

```json
GET /my-index/_mapping
```

The API returns the following response:

```json
{
  "my-index" : {
    "mappings" : {
      "properties" : {
        "age" : {
          "type" : "integer"
        },
        "email" : {
          "type" : "keyword"
        },
        "employee-id" : {
          "type" : "keyword",
          "index" : false
        },
        "name" : {
          "type" : "text"
        }
      }
    }
  }
}
```

## View the mapping of specific fields

If you only want to view the mapping of one or more specific fields, you can use the [get field mapping](https://www.elastic.co/guide/en/elasticsearch/reference/current/indices-get-field-mapping.html) API.

This is useful if you don’t need the complete mapping of an index or your index contains a large number of fields.

The following request retrieves the mapping for the `employee-id` field.

```json
GET /my-index/_mapping/field/employee-id
```

The API returns the following response:

```json
{
  "my-index" : {
    "mappings" : {
      "employee-id" : {
        "full_name" : "employee-id",
        "mapping" : {
          "employee-id" : {
            "type" : "keyword",
            "index" : false
          }
        }
      }
    }
  }
}
```

## why are mapping types being removed

Initially, we spoke about an “index” being similar to a “database” in an SQL database, and a “type” being equivalent to a “table”.



This was a bad analogy that led to incorrect assumptions. In an SQL database, tables are independent of each other. The columns in one table have no bearing on columns with the same name in another table. This is not the case for fields in a mapping type.

In an Elasticsearch index, fields that have the same name in different mapping types are backed by the same Lucene field internally. In other words, using the example above, the `user_name` field in the `user` type is stored in exactly the same field as the `user_name` field in the `tweet` type, and both `user_name` fields must have the same mapping (definition) in both types.

This can lead to frustration when, for example, you want `deleted` to be a `date` field in one type and a `boolean` field in another type in the same index.

On top of that, storing different entities that have few or no fields in common in the same index leads to sparse data and interferes with Lucene’s ability to compress documents efficiently.

最重要的是，存储在同一索引中具有很少或没有相同字段的不同实体会导致数据稀疏并干扰Lucene有效压缩文档的能力。

For these reasons, we have decided to remove the concept of mapping types from Elasticsearch.