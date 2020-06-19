# QueryBuilder 

顶级接口

## AbstractQueryBuilder

Base class for all classes producing lucene queries.Supports conversion to BytesReference and creation of lucene Query objects.

### MatchQueryBuilder

Match query is a query that analyzes the text and constructs a query as the result of the analysis.

### MultiMatchQueryBuilder

Same as  **MatchQueryBuilder** but supports multiple fields.



### BoolQueryBuilder

A Query that matches documents matching boolean combinations of other queries.

### TermsQueryBuilder

A filter for a field based on several terms matching on any of them.



### RangeQueryBuilder

A Query that matches documents within an range of terms.