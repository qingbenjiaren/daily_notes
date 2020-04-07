# Index API

The index API allows one to index a typed JSON document into a specific index and make it searchable.

## Generate JSON document

There are several different ways of generating a JSON document:

- Manually (aka do it yourself) using native `byte[]` or as a `String`
- Using a `Map` that will be automatically converted to its JSON equivalent
- Using a third party library to serialize your beans such as [Jackson](https://github.com/FasterXML/jackson)
- Using built-in helpers XContentFactory.jsonBuilder()

Internally, each type is converted to `byte[]` (so a String is converted to a `byte[]`). Therefore, if the object is in this form already, then use it. The `jsonBuilder` is highly optimized JSON generator that directly constructs a `byte[]`.

### Do It Yourself

Nothing really difficult here but note that you will have to encode dates according to the [Date Format](https://www.elastic.co/guide/en/elasticsearch/reference/7.6/mapping-date-format.html).

```java
String json = "{" +
        "\"user\":\"kimchy\"," +
        "\"postDate\":\"2013-01-30\"," +
        "\"message\":\"trying out Elasticsearch\"" +
    "}";
```

### Using Map

Map is a key:values pair collection. It represents a JSON structure:

```java
Map<String, Object> json = new HashMap<String, Object>();
json.put("user","kimchy");
json.put("postDate",new Date());
json.put("message","trying out Elasticsearch");
```

### Serialize your bean

You can use [Jackson](https://github.com/FasterXML/jackson) to serialize your beans to JSON. Please add [Jackson Databind](http://search.maven.org/#search|ga|1|jackson-databind) to your project. Then you can use `ObjectMapper` to serialize your beans:

```java
import com.fasterxml.jackson.databind.*;

// instance a json mapper
ObjectMapper mapper = new ObjectMapper(); // create once, reuse

// generate json
byte[] json = mapper.writeValueAsBytes(yourbeaninstance);
```

### Use Elasticseach helpers

Elasticsearch provides built-in helpers to generate JSON content.

```java
import static org.elasticsearch.common.xcontent.XContentFactory.*;

XContentBuilder builder = jsonBuilder()
    .startObject()
        .field("user", "kimchy")
        .field("postDate", new Date())
        .field("message", "trying out Elasticsearch")
    .endObject()
```

Note that you can also add arrays with `startArray(String)` and `endArray()` methods. By the way, the `field` method
accepts many object types. You can directly pass numbers, dates and even other XContentBuilder objects.

If you need to see the generated JSON content, you can use the `Strings.toString()` method.

```java
import org.elasticsearch.common.Strings;

String json = Strings.toString(builder);
```

## Index document

The following example indexes a JSON document into an index called twitter, under a type called `_doc``, with id valued 1:

```java
import static org.elasticsearch.common.xcontent.XContentFactory.*;

IndexResponse response = client.prepareIndex("twitter", "_doc", "1")
        .setSource(jsonBuilder()
                    .startObject()
                        .field("user", "kimchy")
                        .field("postDate", new Date())
                        .field("message", "trying out Elasticsearch")
                    .endObject()
                  )
        .get();
```

Note that you can also index your documents as JSON String and that you don’t have to give an ID:

```java
String json = "{" +
        "\"user\":\"kimchy\"," +
        "\"postDate\":\"2013-01-30\"," +
        "\"message\":\"trying out Elasticsearch\"" +
    "}";

IndexResponse response = client.prepareIndex("twitter", "_doc")
        .setSource(json, XContentType.JSON)
        .get();
```

`IndexResponse` object will give you a report:

```java
// Index name
String _index = response.getIndex();
// Type name
String _type = response.getType();
// Document ID (generated or not)
String _id = response.getId();
// Version (if it's the first time you index this document, you will get: 1)
long _version = response.getVersion();
// status has stored current instance statement.
RestStatus status = response.status();
```

For more information on the index operation, check out the REST [index](https://www.elastic.co/guide/en/elasticsearch/reference/7.6/docs-index_.html) docs.

# Get API

The get API allows to get a typed JSON document from the index based on its id. The following example gets a JSON document from an index called twitter, under a type called `_doc``, with id valued 1:

```java
GetResponse response = client.prepareGet("twitter", "_doc", "1").get();
```

## Request

```
GET /_doc/<_id>
HEAD /_doc/<_id>
GET /_source/<_id>
HEAD /_source/<_id>
```

### Description

You use GET to retrieve a document and its source or stored fields from a particular index. Use HEAD to verify that a document exists. You can use the `_source` resource retrieve just the document source or verify that it exists.

### Realtime

By default, the get API is realtime, and is not affected by the refresh rate of the index (when data will become visible for search). In case where stored fields are requested (see `stored_fields` parameter) and the document has been updated but is not yet refreshed, the get API will have to parse and analyze the source to extract the stored fields. In order to disable realtime GET, the `realtime` parameter can be set to `false`.

### Source filtering

By default, the get operation returns the contents of the `_source` field unless you have used the `stored_fields` parameter or if the `_source` field is disabled. You can turn off `_source` retrieval by using the `_source` parameter:

```java
GET twitter/_doc/0?_source=false
```

If you only need one or two fields from the `_source`, use the `_source_includes` or `_source_excludes` parameters to include or filter out particular fields. This can be especially helpful with **large documents** where partial retrieval can save on network overhead. Both parameters take a comma separated list of fields or wildcard expressions. Example:

```java
GET twitter/_doc/0?_source_includes=*.id&_source_excludes=entities
```

If you only want to specify includes, you can use a shorter notation:

```
GET twitter/_doc/0?_source=*.id,retweeted
```

### Routing

If routing is used during indexing, the routing value also needs to be specified to retrieve a document. For example:

```java
GET twitter/_doc/2?routing=user1
```



# Delete API

The delete API allows one to delete a typed JSON document from a specific index based on its id. The following example deletes the JSON document from an index called twitter, under a type called `_doc`, with id valued 1:

```java
DeleteResponse response = client.prepareDelete("twitter", "_doc", "1").get();
```



# Delete By Query API

The delete by query API allows one to delete a given set of documents based on the result of a query:

```java
BulkByScrollResponse response =
  new DeleteByQueryRequestBuilder(client, DeleteByQueryAction.INSTANCE)
    .filter(QueryBuilders.matchQuery("gender", "male")) //1
    .source("persons")     //2                             
    .get();                 //3                            
long deleted = response.getDeleted();   //4
```

1. query
2. index
3. execute the operation
4. number of delete documents

As it can be a long running operation, if you wish to do it asynchronously, you can call `execute` instead of `get` and provide a listener like:

```java
new DeleteByQueryRequestBuilder(client, DeleteByQueryAction.INSTANCE)
    .filter(QueryBuilders.matchQuery("gender", "male"))     //1
    .source("persons")          //2                            
    .execute(new ActionListener<BulkByScrollResponse>() {   //3
        @Override
        public void onResponse(BulkByScrollResponse response) {
            long deleted = response.getDeleted();   //4        
        }
        @Override
        public void onFailure(Exception e) {
            // Handle the exception
        }
    });
```

1. query
2. index
3. execute the operation
4. number of delete documents



# Update API

You can either create an `UpdateRequest` and send it to the client:

```java
UpdateRequest updateRequest = new UpdateRequest();
updateRequest.index("index");
updateRequest.type("_doc");
updateRequest.id("1");
updateRequest.doc(jsonBuilder()
        .startObject()
            .field("gender", "male")
        .endObject());
client.update(updateRequest).get();
```

Or you can use `prepareUpdate()` method:

```java
client.prepareUpdate("ttl", "doc", "1")
        .setScript(new Script(
            "ctx._source.gender = \"male\"",//1 
            ScriptType.INLINE, null, null))
        .get();

client.prepareUpdate("ttl", "doc", "1")
        .setDoc(jsonBuilder()     //2          
            .startObject()
                .field("gender", "male")
            .endObject())
        .get();
```

1. Your script. It could also be a locally stored script name. In that case, you’ll need to use `ScriptType.FILE`.
2. Document which will be merged to the existing one.

## Update by script

The update API allows to update a document based on a script provided:

```java
UpdateRequest updateRequest = new UpdateRequest("ttl", "doc", "1")
        .script(new Script("ctx._source.gender = \"male\""));
client.update(updateRequest).get();
```

## Update by merging documents

The update API also support passing a partial document, which will be merged into the existing document (simple recursive merge, inner merging of objects, replacing core "keys/values" and arrays). For example:

```java
UpdateRequest updateRequest = new UpdateRequest("index", "type", "1")
        .doc(jsonBuilder()
            .startObject()
                .field("gender", "male")
            .endObject());
client.update(updateRequest).get();
```

## Upsert

There is also support for `upsert`. If the document does not exist, the content of the `upsert` element will be used to index the fresh doc:

```java
IndexRequest indexRequest = new IndexRequest("index", "type", "1")
        .source(jsonBuilder()
            .startObject()
                .field("name", "Joe Smith")
                .field("gender", "male")
            .endObject());
UpdateRequest updateRequest = new UpdateRequest("index", "type", "1")
        .doc(jsonBuilder()
            .startObject()
                .field("name", "Joe Dalton")
            .endObject())
        .upsert(indexRequest);       //1       
client.update(updateRequest).get();
```

1.  If the document does not exist, the one in `indexRequest` will be added

If the document `index/_doc/1` already exists, we will have after this operation a document like:

```json
{
    "name"  : "Joe Dalton",      //1  
    "gender": "male"
}
```

1.  This field is updated by the update request

If it does not exist, we will have a new document:

```json
{
    "name" : "Joe Smith",
    "gender": "male"
}
```



# Multi Get API

The multi get API allows to get a list of documents based on their `index` and `id`:

```java
MultiGetResponse multiGetItemResponses = client.prepareMultiGet()
    .add("twitter", "_doc", "1")      //1      
    .add("twitter", "_doc", "2", "3", "4")//2  
    .add("another", "_doc", "foo")        //3  
    .get();

for (MultiGetItemResponse itemResponse : multiGetItemResponses) { //4
    GetResponse response = itemResponse.getResponse();
    if (response.isExists()) {//5                      
        String json = response.getSourceAsString(); //6
    }
}
```

1.  get by a single id
2. or by a list of ids for the same index
3. you can also get from another index
4.  iterate over the result set
5.  you can check if the document exists
6. access to the `_source` field



# Bulk API

The bulk API allows one to index and delete several documents in a single request. Here is a sample usage:

```java
import static org.elasticsearch.common.xcontent.XContentFactory.*;

BulkRequestBuilder bulkRequest = client.prepareBulk();

// either use client#prepare, or use Requests# to directly build index/delete requests
bulkRequest.add(client.prepareIndex("twitter", "_doc", "1")
        .setSource(jsonBuilder()
                    .startObject()
                        .field("user", "kimchy")
                        .field("postDate", new Date())
                        .field("message", "trying out Elasticsearch")
                    .endObject()
                  )
        );

bulkRequest.add(client.prepareIndex("twitter", "_doc", "2")
        .setSource(jsonBuilder()
                    .startObject()
                        .field("user", "kimchy")
                        .field("postDate", new Date())
                        .field("message", "another post")
                    .endObject()
                  )
        );

BulkResponse bulkResponse = bulkRequest.get();
if (bulkResponse.hasFailures()) {
    // process failures by iterating through each bulk response item
}
```



# Using Bulk Processor

The `BulkProcessor` class offers a simple interface to flush bulk operations automatically based on the number or size of requests, or after a given period.

To use it, first create a `BulkProcessor` instance:

```java
import org.elasticsearch.action.bulk.BackoffPolicy;
import org.elasticsearch.action.bulk.BulkProcessor;
import org.elasticsearch.common.unit.ByteSizeUnit;
import org.elasticsearch.common.unit.ByteSizeValue;
import org.elasticsearch.common.unit.TimeValue;

BulkProcessor bulkProcessor = BulkProcessor.builder(
        client,  //1
        new BulkProcessor.Listener() {
            @Override
            public void beforeBulk(long executionId,
                                   BulkRequest request) { ... } //2

            @Override
            public void afterBulk(long executionId,
                                  BulkRequest request,
                                  BulkResponse response) { ... } //3

            @Override
            public void afterBulk(long executionId,
                                  BulkRequest request,
                                  Throwable failure) { ... } //4
        })
        .setBulkActions(10000) //5
        .setBulkSize(new ByteSizeValue(5, ByteSizeUnit.MB)) //6
        .setFlushInterval(TimeValue.timeValueSeconds(5)) //7
        .setConcurrentRequests(1) //8
        .setBackoffPolicy(
            BackoffPolicy.exponentialBackoff(TimeValue.timeValueMillis(100), 3)) //9
        .build();
```

1. Add your Elasticsearch client
2. This method is called just before bulk is executed. You can for example see the numberOfActions with `request.numberOfActions()`
3. This method is called after bulk execution. You can for example check if there was some failing requests with `response.hasFailures()`
4.  This method is called when the bulk failed and raised a `Throwable`
5. We want to execute the bulk every 10 000 requests
6.  We want to flush the bulk every 5mb
7. We want to flush the bulk every 5 seconds whatever the number of requests
8. Set the number of concurrent requests. A value of 0 means that only a single request will be allowed to be executed. A value of 1 means 1 concurrent request is allowed to be executed while accumulating new bulk requests.
9. Set a custom backoff policy which will initially wait for 100ms, increase exponentially and retries up to three times. A retry is attempted whenever one or more bulk item requests have failed with an `EsRejectedExecutionException` which indicates that there were too little compute resources available for processing the request. To disable backoff, pass `BackoffPolicy.noBackoff()`.

By default, `BulkProcessor`:

- sets bulkActions to `1000`
- sets bulkSize to `5mb`
- does not set flushInterval
- sets concurrentRequests to 1, which means an asynchronous execution of the flush operation.
- sets backoffPolicy to an exponential backoff with 8 retries and a start delay of 50ms. The total wait time is roughly 5.1 seconds.

## add Request

Then you can simply add your requests to the `BulkProcessor`:

```java
bulkProcessor.add(new IndexRequest("twitter", "_doc", "1").source(/* your doc here */));
bulkProcessor.add(new DeleteRequest("twitter", "_doc", "2"));
```

## Closing the Bulk Processor

When all documents are loaded to the `BulkProcessor` it can be closed by using `awaitClose` or `close` methods:

```java
bulkProcessor.awaitClose(10, TimeUnit.MINUTES);
```

or

```java
bulkProcessor.close();
```

Both methods flush any remaining documents and disable all other scheduled flushes, if they were scheduled by setting `flushInterval`. If concurrent requests were enabled, the `awaitClose` method waits for up to the specified timeout for all bulk requests to complete then returns `true`; if the specified waiting time elapses before all bulk requests complete, `false` is returned. The `close` method doesn’t wait for any remaining bulk requests to complete and exits immediately.

## Using Bulk Processor in tests

If you are running tests with Elasticsearch and are using the `BulkProcessor` to populate your dataset you should better set the number of concurrent requests to `0` so the flush operation of the bulk will be executed in a synchronous manner:

```java
BulkProcessor bulkProcessor = BulkProcessor.builder(client, new BulkProcessor.Listener() { /* Listener methods */ })
        .setBulkActions(10000)
        .setConcurrentRequests(0)
        .build();

// Add your requests
bulkProcessor.add(/* Your requests */);

// Flush any remaining requests
bulkProcessor.flush();

// Or close the bulkProcessor if you don't need it anymore
bulkProcessor.close();

// Refresh your indices
client.admin().indices().prepareRefresh().get();

// Now you can start searching!
client.prepareSearch().get();
```

## Global Parameters

Global parameters can be specified on the BulkRequest as well as BulkProcessor, similar to the REST API. These global parameters serve as defaults and can be overridden by local parameters specified on each sub request. Some parameters have to be set before any sub request is added - index, type - and you have to specify them during BulkRequest or BulkProcessor creation. Some are optional - pipeline, routing - and can be specified at any point before the bulk is sent.

```java
try (BulkProcessor processor = initBulkProcessorBuilder(listener)
        .setGlobalIndex("tweets")
        .setGlobalType("_doc")
        .setGlobalRouting("routing")
        .setGlobalPipeline("pipeline_id")
        .build()) {


    processor.add(new IndexRequest() //1
        .source(XContentType.JSON, "user", "some user"));
    processor.add(new IndexRequest("blogs").id("1") //2
        .source(XContentType.JSON, "title", "some title"));
}
```

1. global parameters from the BulkRequest will be applied on a sub request
2.  local pipeline parameter on a sub request will override global parameters from BulkRequest

```java
BulkRequest request = new BulkRequest();
request.pipeline("globalId");

request.add(new IndexRequest("test").id("1")
    .source(XContentType.JSON, "field", "bulk1")
    .setPipeline("perIndexId")); //1

request.add(new IndexRequest("test").id("2")
    .source(XContentType.JSON, "field", "bulk2")); //2
```

- local pipeline parameter on a sub request will override global pipeline from the BulkRequest
- global parameter from the BulkRequest will be applied on a sub request

# Update By Query API

The simplest usage of `updateByQuery` updates each document in an index without changing the source. This usage enables picking up a new property or another online mapping change.

```java
UpdateByQueryRequestBuilder updateByQuery =
  new UpdateByQueryRequestBuilder(client, UpdateByQueryAction.INSTANCE);
updateByQuery.source("source_index").abortOnVersionConflict(false);
BulkByScrollResponse response = updateByQuery.get();
```

