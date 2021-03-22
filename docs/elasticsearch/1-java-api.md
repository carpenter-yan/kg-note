##java api
参考资料
[elasticsearch系列七：ES Java客户端-Elasticsearch Java client](https://blog.csdn.net/qq_26676207/article/details/81019677?utm_medium=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-2.control&dist_request_id=&depth_1-utm_source=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-2.control)

### 1. Java REST Client
每个API 支持 同步/异步 两种方式，同步方法直接返回一个结果对象。异步的方法以async为后缀，通过listener参数来通知结果。
http协议9200端口

### 2. Java Transport Client
tcp协议9300端口
各种操作本质上都是异步的(可以用 listener，或返回 Future）
high level rest client 中的操作API和java transport client大多是一样的。

使用XXXRequest和XXXResponse分别代表请求和返回（与Client类型无关，属于服务端jar包公共组件）
java transport client在请求时支持PrepareXXX方式请求，high level rest client不支持。
建议使用两种client都支持的XXX（操作类型名）的方式。如client.index(indexRequest)索引文档

### 3.支持的操作和组件
注释:按照  rest / transport 方式对比(prepareXXX方式不写)
这里rest是同步调用，transport是future异步调用，主要是为了对比接口差异，rest异步调用方式可以参考资料
3.1 客户端创建
RestHighLevelClient / PreBuiltTransportClient

3.2 create index
org.elasticsearch.action.admin.indices.create.CreateIndexRequest
org.elasticsearch.action.admin.indices.create.CreateIndexResponse
client.indices().create(request) / client.admin().indices().create(request).get()

3.3 index document
org.elasticsearch.action.index.IndexRequest
org.elasticsearch.action.index.IndexResponse
client.index(request) / client.index(request).get()

3.4 get document
org.elasticsearch.action.get.GetRequest
org.elasticsearch.action.get.GetResponse
org.elasticsearch.search.fetch.subphase.FetchSourceContext
client.get(request) / client.get(request).get()

3.5 bulk
org.elasticsearch.action.bulk.BulkRequest
org.elasticsearch.action.bulk.BulkResponse
client.bulk(request) / client.bulk(request).get()

3.6 search
org.elasticsearch.action.search.SearchRequest
org.elasticsearch.action.search.SearchResponse
org.elasticsearch.index.query.QueryBuilders
org.elasticsearch.search.builder.SearchSourceBuilder
client.search(request) / client.search(request).get()

3.7 highlight
3.8 suggest
3.9 aggregation
以上3个不常用，可以参考资料。

