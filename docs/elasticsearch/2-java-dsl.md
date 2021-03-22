##dsl
参考资料
[Elasticsearch Query DSL 整理总结（一）](https://www.cnblogs.com/reycg-blog/p/10000052.html)  
[Elasticsearch Query DSL 整理总结（二）](https://www.cnblogs.com/reycg-blog/p/10002794.html)  
[Elasticsearch Query DSL 整理总结（三）](https://www.cnblogs.com/reycg-blog/p/10012238.html)  

[Elasticsearch Java Rest Client API 整理总结 (二)](https://www.cnblogs.com/reycg-blog/p/9946821.html)  
[Elasticsearch Java Rest Client API 整理总结 (三)](https://www.cnblogs.com/reycg-blog/p/9993094.html)  

平时常用主要是复合查询种的Bool查询以及基于词项的查询
[官网Building Querries](https://www.elastic.co/guide/en/elasticsearch/client/java-rest/6.3/java-rest-high-query-builders.html#java-rest-high-query-builders)  
词项为构建基本的叶子节点，可参考官网term level queries
bool查询为符合查询用于整合叶子节点，可参考官网compound queries中BoolQueryBuilder的api
其中或查询可以是用should参数为list的接口来构建and(X or Y)的效果