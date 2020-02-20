## ElasticSearch简介
这段说明来自ES官方文档----[你知道的，为了搜索...](https://www.elastic.co/guide/cn/elasticsearch/guide/current/intro.html)
Elasticsearch 是一个开源的搜索引擎，建立在一个全文搜索引擎库 Apache Lucene™ 基础之上。 Lucene 可以说是当下最先进、高性能、全功能的搜索引擎库—无论是开源还是私有。
Elasticsearch 也是使用 Java 编写的，它的内部使用 Lucene 做索引与搜索，但是它的目的是使全文检索变得简单， 通过隐藏 Lucene 的复杂性，取而代之的提供一套简单一致的 RESTful API。
然而，Elasticsearch 不仅仅是 Lucene，并且也不仅仅只是一个全文搜索引擎。 它可以被下面这样准确的形容：
- 一个分布式的实时文档存储，每个字段 可以被索引与搜索
- 一个分布式实时分析搜索引擎
- 能胜任上百个服务节点的扩展，并支持 PB 级别的结构化或者非结构化数据

PS:在学习的ES的时候最好装个kibana，方便学习过程中印证你的猜想，事半功倍，在拥有正反馈时的学习效率比没有反馈时的效率高很多。
## 1.依赖引入
``` xml
   <!--启动配置-->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-elasticsearch</artifactId>
        </dependency>
        <!--一些相关的依赖，可以不引入-->
        <dependency>
            <groupId>org.springframework.data</groupId>
            <artifactId>spring-data-elasticsearch</artifactId>
        </dependency>
        <!--highLevelClient依赖，必须要-->
        <dependency>
            <groupId>org.elasticsearch.client</groupId>
            <artifactId>elasticsearch-rest-high-level-client</artifactId>
        </dependency>
        <!--es的相关源码，可以学习一下，可以不引入-->
        <dependency>
            <groupId>org.elasticsearch</groupId>
            <artifactId>elasticsearch</artifactId>
        </dependency>
```
## 2.入门级配置
配置信息在springboot的autoconfigure包下面
``` java
@ConfigurationProperties(prefix = "spring.elasticsearch.rest")
public class RestClientProperties {

	/**
	 * 这里可以配置多个地址，在配置文件里面可以有：spring.elasticsearch.rest.uris=http://192.168.164.135:9200
	 */
	private List<String> uris = new ArrayList<>(Collections.singletonList("http://localhost:9200"));

	/**
	 * 连接用户名.
	 */
	private String username;

	/**
	 * 连接密码.
	 */
	private String password;

	public List<String> getUris() {
		return this.uris;
	}

	public void setUris(List<String> uris) {
		this.uris = uris;
	}

	public String getUsername() {
		return this.username;
	}

	public void setUsername(String username) {
		this.username = username;
	}

	public String getPassword() {
		return this.password;
	}

	public void setPassword(String password) {
		this.password = password;
	}

}
```
``` java
// 这个是加载配置信息的类
class RestClientConfigurations {

	@Configuration
	static class RestClientBuilderConfiguration {

		@Bean
		@ConditionalOnMissingBean
		public RestClientBuilder elasticsearchRestClientBuilder(RestClientProperties properties,
				ObjectProvider<RestClientBuilderCustomizer> builderCustomizers) {
			HttpHost[] hosts = properties.getUris().stream().map(HttpHost::create).toArray(HttpHost[]::new);
			RestClientBuilder builder = RestClient.builder(hosts);
			PropertyMapper map = PropertyMapper.get();
			map.from(properties::getUsername).whenHasText().to((username) -> {
				CredentialsProvider credentialsProvider = new BasicCredentialsProvider();
				Credentials credentials = new UsernamePasswordCredentials(properties.getUsername(),
						properties.getPassword());
				credentialsProvider.setCredentials(AuthScope.ANY, credentials);
				builder.setHttpClientConfigCallback(
						(httpClientBuilder) -> httpClientBuilder.setDefaultCredentialsProvider(credentialsProvider));
			});
			builderCustomizers.orderedStream().forEach((customizer) -> customizer.customize(builder));
			return builder;
		}
	}

	@Configuration
    // 如果不引入elasticsearch-rest-high-level-client这个包，那么这个bean是不会被加载的
	@ConditionalOnClass(RestHighLevelClient.class)
	static class RestHighLevelClientConfiguration {

		@Bean
		@ConditionalOnMissingBean
		public RestHighLevelClient elasticsearchRestHighLevelClient(RestClientBuilder restClientBuilder) {
			return new RestHighLevelClient(restClientBuilder);
		}

		@Bean
		@ConditionalOnMissingBean
		public RestClient elasticsearchRestClient(RestClientBuilder builder,
				ObjectProvider<RestHighLevelClient> restHighLevelClient) {
			RestHighLevelClient client = restHighLevelClient.getIfUnique();
			if (client != null) {
				return client.getLowLevelClient();
			}
			return builder.build();
		}

	}

	@Configuration
	static class RestClientFallbackConfiguration {

		@Bean
		@ConditionalOnMissingBean
		public RestClient elasticsearchRestClient(RestClientBuilder builder) {
			return builder.build();
		}

	}

}
```
## 3.ES基础操作
使用RestHighLevelClient类，可以完成对ES的增删改查操作，首先我们自定义一个接口类，来申明我们需要使用的接口，入参以及返回结果。
``` java
public interface EsCurdService {

    /**
     * 功能描述: 创建索引
     *
     * @param indexName 索引名称
     * @return:void
     * @since: v1.0
     * @Author:wangcanfeng
     * @Date: 2019/10/31-16:51
     */
    String createIndex(String indexName) throws IOException;

    /**
     * 功能描述: 删除索引
     *
     * @param indexName 索引名称
     * @return:void
     * @since: v1.0
     * @Author:wangcanfeng
     * @Date: 2019/10/31-16:51
     */
    void deleteIndex(String indexName) throws IOException;

    /**
     * 功能描述: 查询索引
     *
     * @param indexName 索引名称
     * @return:void
     * @since: v1.0
     * @Author:wangcanfeng
     * @Date: 2019/10/31-16:51
     */
    boolean existIndex(String indexName) throws IOException;

    /**
     * 功能描述: 往索引里插入数据
     *
     * @param indexName 索引名称
     * @param docType   数据对象类型，可以把它设置成对象类的类名
     * @param object    数据对象
     * @return:void
     * @since: v1.0
     * @Author:wangcanfeng
     * @Date: 2019/10/31-17:26
     */
    RestStatus insert(String indexName, String docType, Object object) throws IOException;

    /**
     * 功能描述: 根据索引名称和类型去查询匹配的信息
     *
     * @param indexName 索引名称
     * @param docType   数据对象类型，可以把它设置成对象类的类名
     * @return:java.util.List<java.lang.String>
     * @since: v1.0
     * @Author:wangcanfeng
     * @Date: 2019/10/31-21:20
     */
    List<String> search(String indexName, String docType) throws IOException;

    /**
     * 功能描述: 更新数据
     *
     * @param indexName 索引名称
     * @param docType   数据对象类型，可以把它设置成对象类的类名
     * @param object    数据对象
     * @return:void
     * @since: v1.0
     * @Author:wangcanfeng
     * @Date: 2019/10/31-17:26
     */
    RestStatus update(String indexName, String docType, String id, Object object) throws IOException;

    /**
     * 功能描述: 根据id删除数据
     *
     * @param id _id编号
     * @return:org.elasticsearch.rest.RestStatus
     * @since: v1.0
     * @Author:wangcanfeng
     * @Date: 2019/10/31-21:39
     */
    RestStatus delete(String id) throws IOException;
}
```
接口实现类
``` java
import com.alibaba.fastjson.JSON;
import org.elasticsearch.action.admin.indices.create.CreateIndexRequest;
import org.elasticsearch.action.admin.indices.create.CreateIndexResponse;
import org.elasticsearch.action.admin.indices.delete.DeleteIndexRequest;
import org.elasticsearch.action.admin.indices.get.GetIndexRequest;
import org.elasticsearch.action.delete.DeleteRequest;
import org.elasticsearch.action.delete.DeleteResponse;
import org.elasticsearch.action.index.IndexRequest;
import org.elasticsearch.action.index.IndexResponse;
import org.elasticsearch.action.search.SearchRequest;
import org.elasticsearch.action.search.SearchResponse;
import org.elasticsearch.action.update.UpdateRequest;
import org.elasticsearch.action.update.UpdateResponse;
import org.elasticsearch.client.RequestOptions;
import org.elasticsearch.client.RestHighLevelClient;
import org.elasticsearch.common.settings.Settings;
import org.elasticsearch.common.xcontent.XContentType;
import org.elasticsearch.index.query.BoolQueryBuilder;
import org.elasticsearch.index.query.MatchQueryBuilder;
import org.elasticsearch.index.query.QueryBuilders;
import org.elasticsearch.rest.RestStatus;
import org.elasticsearch.search.builder.SearchSourceBuilder;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

import java.io.IOException;
import java.util.LinkedList;
import java.util.List;

/**
 * @author wangcanfeng
 * @description es的基础增删改查操作实现类
 * @Date Created in 16:54-2019/10/31
 */
@Service
public class EsCurdServiceImpl implements EsCurdService {

    @Autowired
    private RestHighLevelClient rhlClient;

    /**
     * 功能描述: 创建索引
     *
     * @param indexName 索引名称
     * @return:void
     * @since: v1.0
     * @Author:wangcanfeng
     * @Date: 2019/10/31-16:51
     */
    @Override
    public String createIndex(String indexName) throws IOException {
        // 创建索引
        CreateIndexRequest createIndexRequest = new CreateIndexRequest();
        createIndexRequest.index(indexName).settings(Settings.builder()
                // 分片数
                .put("index.number_of_shards", 3)
                // 备份数
                .put("index.number_of_replicas", 2));
        CreateIndexResponse response = rhlClient.indices().create(createIndexRequest, RequestOptions.DEFAULT);
        return response.index();
    }

    /**
     * 功能描述: 删除索引
     *
     * @param indexName 索引名称
     * @return:void
     * @since: v1.0
     * @Author:wangcanfeng
     * @Date: 2019/10/31-16:51
     */
    @Override
    public void deleteIndex(String indexName) throws IOException {
        DeleteIndexRequest deleteIndexRequest = new DeleteIndexRequest();
        deleteIndexRequest.indices(indexName);
        rhlClient.indices().delete(deleteIndexRequest, RequestOptions.DEFAULT);
    }

    /**
     * 功能描述: 查询索引
     *
     * @param indexName 索引名称
     * @return:void
     * @since: v1.0
     * @Author:wangcanfeng
     * @Date: 2019/10/31-16:51
     */
    @Override
    public boolean existIndex(String indexName) throws IOException {
        // 判断索引是否存在
        GetIndexRequest getIndexRequest = new GetIndexRequest();
        getIndexRequest.indices(indexName);
        return rhlClient.indices().exists(getIndexRequest, RequestOptions.DEFAULT);
    }

    /**
     * 功能描述: 往索引里插入数据
     *
     * @param indexName 索引名称
     * @param docType   数据对象类型，可以把它设置成对象类的类名
     * @param object    数据对象
     * @return:void
     * @since: v1.0
     * @Author:wangcanfeng
     * @Date: 2019/10/31-17:26
     */
    @Override
    public RestStatus insert(String indexName, String docType, Object object) throws IOException {
        IndexRequest indexRequest = new IndexRequest();
        // source方法里面需要填写对应数据的类型，默认数据类型为json
        indexRequest.index(indexName).type(docType).source(JSON.toJSONString(object), XContentType.JSON);
        IndexResponse response = rhlClient.index(indexRequest, RequestOptions.DEFAULT);
        return response.status();
    }

    @Override
    public List<String> search(String indexName, String docType) throws IOException {
        SearchRequest searchRequest = new SearchRequest();
        SearchSourceBuilder sourceBuilder = new SearchSourceBuilder();
        // 分页参数
        sourceBuilder.from(0);
        sourceBuilder.size(10);
        BoolQueryBuilder boolBuilder = QueryBuilders.boolQuery();
        // 关键词查找匹配，这个不是精确查找，他会对字段中的内容进行切词匹配
        MatchQueryBuilder matchQueryBuilder = QueryBuilders.matchQuery("name", "wo");
        // 模糊查找
//        FuzzyQueryBuilder fuzzyQueryBuilder = QueryBuilders.fuzzyQuery("name", "wangcagceng").fuzziness(Fuzziness.TWO);
        // 前缀查找
//        PrefixQueryBuilder prefixQueryBuilder = QueryBuilders.prefixQuery("name", "wang");
        boolBuilder.must(matchQueryBuilder);
        // 设置需要返回的字段
//        sourceBuilder.fetchSource(new String[]{"name", "age"}, new String[]{}).query(boolBuilder);
        // 设置索引和类型
        searchRequest.indices("test").types("person").source(sourceBuilder);
        SearchResponse response = rhlClient.search(searchRequest, RequestOptions.DEFAULT);
        List<String> res = new LinkedList<>();
        response.getHits().iterator().forEachRemaining(hit -> {
            res.add(hit.getSourceAsString());
        });
        return res;
    }

    /**
     * 功能描述: 更新数据
     *
     * @param indexName 索引名称
     * @param docType   数据对象类型，可以把它设置成对象类的类名
     * @param object    数据对象
     * @return:void
     * @since: v1.0
     * @Author:wangcanfeng
     * @Date: 2019/10/31-17:26
     */
    @Override
    public RestStatus update(String indexName, String docType, String id, Object object) throws IOException {
        UpdateRequest request = new UpdateRequest();
        request.index(indexName).type(docType).id(id).doc(JSON.toJSONString(object), XContentType.JSON);
        UpdateResponse response = rhlClient.update(request, RequestOptions.DEFAULT);
        return response.status();
    }

    /**
     * 功能描述: 根据id删除数据
     *
     * @param id _id编号
     * @return:org.elasticsearch.rest.RestStatus
     * @since: v1.0
     * @Author:wangcanfeng
     * @Date: 2019/10/31-21:39
     */
    @Override
    public RestStatus delete(String id) throws IOException {
        DeleteRequest request = new DeleteRequest();
        request.index("test").type("person").id("N1UfIW4BRvOvOTozKH0w");
        DeleteResponse response = rhlClient.delete(request, RequestOptions.DEFAULT);
        return response.status();
    }
}
```
# 总结
关于springboot集成elasticSearch的基础知识就分享到这里了，有疑问可以留言。更多深入的知识请关注我@_@
原创不易，转载请复制原文链接