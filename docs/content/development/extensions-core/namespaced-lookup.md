---
layout: doc_page
---

# Namespaced Lookup

<div class="note caution">
Lookups are an <a href="../development/experimental.html">experimental</a> feature.
</div>

Make sure to [include](../../operations/including-extensions.html) `druid-namespace-lookup` as an extension.

## Configuration

Namespaced lookups are appropriate for lookups which are not possible to pass at query time due to their size, 
or are not desired to be passed at query time because the data is to reside in and be handled by the Druid servers. 
Namespaced lookups can be specified as part of the runtime properties file. The property is a list of the namespaces 
described as per the sections on this page. For example:

 ```json
 druid.query.extraction.namespace.lookups=
   [
     {
       "type": "uri",
       "namespace": "some_uri_lookup",
       "uri": "file:/tmp/prefix/",
       "namespaceParseSpec": {
         "format": "csv",
         "columns": [
           "key",
           "value"
         ]
       },
       "pollPeriod": "PT5M"
     },
     {
       "type": "jdbc",
       "namespace": "some_jdbc_lookup",
       "connectorConfig": {
         "createTables": true,
         "connectURI": "jdbc:mysql:\/\/localhost:3306\/druid",
         "user": "druid",
         "password": "diurd"
       },
       "table": "lookupTable",
       "keyColumn": "mykeyColumn",
       "valueColumn": "MyValueColumn",
       "tsColumn": "timeColumn"
     }
   ]
 ```

Proper functionality of Namespaced lookups requires the following extension to be loaded on the broker, peon, and historical nodes: 
`druid-namespace-lookup`

## Cache Settings

Lookups are cached locally on historical nodes. The following are settings used by the nodes which service queries when 
setting namespaces (broker, peon, historical)

|Property|Description|Default|
|--------|-----------|-------|
|`druid.query.extraction.namespace.cache.type`|Specifies the type of caching to be used by the namespaces. May be one of [`offHeap`, `onHeap`]. `offHeap` uses a temporary file for off-heap storage of the namespace (memory mapped files). `onHeap` stores all cache on the heap in standard java map types.|`onHeap`|

The cache is populated in different ways depending on the settings below. In general, most namespaces employ 
a `pollPeriod` at the end of which time they poll the remote resource of interest for updates.

# Supported Lookups

For additional lookups, please see our [extensions list](../extensions.html).

## URI namespace update

The remapping values for each namespaced lookup can be specified by a json object as per the following examples:

```json
{
  "type":"uri",
  "namespace":"some_lookup",
  "uri": "s3://bucket/some/key/prefix/renames-0003.gz",
  "namespaceParseSpec":{
    "format":"csv",
    "columns":["key","value"]
  },
  "pollPeriod":"PT5M",
}
```

```json
{
  "type":"uri",
  "namespace":"some_lookup",
  "uriPrefix": "s3://bucket/some/key/prefix/",
  "fileRegex":"renames-[0-9]*\\.gz",
  "namespaceParseSpec":{
    "format":"csv",
    "columns":["key","value"]
  },
  "pollPeriod":"PT5M",
}
```
|Property|Description|Required|Default|
|--------|-----------|--------|-------|
|`namespace`|The namespace to define|Yes||
|`pollPeriod`|Period between polling for updates|No|0 (only once)|
|`uri`|URI for the file of interest|No|Use `uriPrefix`|
|`uriPrefix`|A URI which specifies a directory (or other searchable resource) in which to search for files|No|Use `uri`|
|`fileRegex`|Optional regex for matching the file name under `uriPrefix`. Only used if `uriPrefix` is used|No|`".*"`|
|`namespaceParseSpec`|How to interpret the data at the URI|Yes||

One of either `uri` xor `uriPrefix` must be specified.

The `pollPeriod` value specifies the period in ISO 8601 format between checks for replacement data for the lookup. If the source of the lookup is capable of providing a timestamp, the lookup will only be updated if it has changed since the prior tick of `pollPeriod`. A value of 0, an absent parameter, or `null` all mean populate once and do not attempt to look for new data later. Whenever an poll occurs, the updating system will look for a file with the most recent timestamp and assume that one with the most recent data set, replacing the local cache of the lookup data.

The `namespaceParseSpec` can be one of a number of values. Each of the examples below would rename foo to bar, baz to bat, and buck to truck. All parseSpec types assumes each input is delimited by a new line. See below for the types of parseSpec supported.

Only ONE file which matches the search will be used. For most implementations, the discriminator for choosing the URIs is by whichever one reports the most recent timestamp for its modification time.

### csv lookupParseSpec

|Parameter|Description|Required|Default|
|---------|-----------|--------|-------|
|`columns`|The list of columns in the csv file|yes|`null`|
|`keyColumn`|The name of the column containing the key|no|The first column|
|`valueColumn`|The name of the column containing the value|no|The second column|

*example input*

```
bar,something,foo
bat,something2,baz
truck,something3,buck
```

*example namespaceParseSpec*

```json
"namespaceParseSpec": {
  "format": "csv",
  "columns": ["value","somethingElse","key"],
  "keyColumn": "key",
  "valueColumn": "value"
}
```

### tsv lookupParseSpec

|Parameter|Description|Required|Default|
|---------|-----------|--------|-------|
|`columns`|The list of columns in the csv file|yes|`null`|
|`keyColumn`|The name of the column containing the key|no|The first column|
|`valueColumn`|The name of the column containing the value|no|The second column|
|`delimiter`|The delimiter in the file|no|tab (`\t`)|


*example input*

```
bar|something,1|foo
bat|something,2|baz
truck|something,3|buck
```

*example namespaceParseSpec*

```json
"namespaceParseSpec": {
  "format": "tsv",
  "columns": ["value","somethingElse","key"],
  "keyColumn": "key",
  "valueColumn": "value",
  "delimiter": "|"
}
```

### customJson lookupParseSpec

|Parameter|Description|Required|Default|
|---------|-----------|--------|-------|
|`keyFieldName`|The field name of the key|yes|null|
|`valueFieldName`|The field name of the value|yes|null|

*example input*

```json
{"key": "foo", "value": "bar", "somethingElse" : "something"}
{"key": "baz", "value": "bat", "somethingElse" : "something"}
{"key": "buck", "somethingElse": "something", "value": "truck"}
```

*example namespaceParseSpec*

```json
"namespaceParseSpec": {
  "format": "customJson",
  "keyFieldName": "key",
  "valueFieldName": "value"
}
```


### simpleJson lookupParseSpec
The `simpleJson` lookupParseSpec does not take any parameters. It is simply a line delimited json file where the field is the key, and the field's value is the value.

*example input*
 
```json
{"foo": "bar"}
{"baz": "bat"}
{"buck": "truck"}
```

*example namespaceParseSpec*

```json
"namespaceParseSpec":{
  "format": "simpleJson"
}
```

## JDBC namespaced lookup

The JDBC lookups will poll a database to populate its local cache. If the `tsColumn` is set it must be able to accept comparisons in the format `'2015-01-01 00:00:00'`. For example, the following must be valid sql for the table `SELECT * FROM some_lookup_table WHERE timestamp_column >  '2015-01-01 00:00:00'`. If `tsColumn` is set, the caching service will attempt to only poll values that were written *after* the last sync. If `tsColumn` is not set, the entire table is pulled every time.

|Parameter|Description|Required|Default|
|---------|-----------|--------|-------|
|`namespace`|The namespace to define|Yes||
|`connectorConfig`|The connector config to use|Yes||
|`table`|The table which contains the key value pairs|Yes||
|`keyColumn`|The column in `table` which contains the keys|Yes||
|`valueColumn`|The column in `table` which contains the values|Yes||
|`tsColumn`| The column in `table` which contains when the key was updated|No|Not used|
|`pollPeriod`|How often to poll the DB|No|0 (only once)|

```json
{
  "type":"jdbc",
  "namespace":"some_lookup",
  "connectorConfig":{
    "createTables":true,
    "connectURI":"jdbc:mysql://localhost:3306/druid",
    "user":"druid",
    "password":"diurd"
  },
  "table":"some_lookup_table",
  "keyColumn":"the_old_dim_value",
  "valueColumn":"the_new_dim_value",
  "tsColumn":"timestamp_column",
  "pollPeriod":600000
}
```
