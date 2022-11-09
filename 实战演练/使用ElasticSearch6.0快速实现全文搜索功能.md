工作中需要实现一个搜索功能，并且导入现有数据库数据，组长推荐用ElasticSearch实现，网上翻一通教程，都是比较古老的文章了，无奈只能自己摸索，参考ES的文档，总算是把服务搭起来了，记录下，希望有同样需求的朋友可以少走弯路，能按照这篇教程快速的搭建一个可用的ElasticSearch服务。

## ES的搭建

ES搭建有直接下载zip文件，也有docker容器的方式，相对来说，docker更适合我们跑ES服务。可以方便的搭建集群或建立测试环境。这里使用的也是容器方式，首先我们需要一份Dockerfile:

```bash
FROM docker.elastic.co/elasticsearch/elasticsearch-oss:6.0.0
# 提交配置&emsp;包括新的elasticsearch.yml 和&emsp;keystore.jks文件
COPY --chown=elasticsearch:elasticsearch conf/ /usr/share/elasticsearch/config/
# 安装ik
RUN ./bin/elasticsearch-plugin install https://github.com/medcl/elasticsearch-analysis-ik/releases/download/v6.0.0/elasticsearch-analysis-ik-6.0.0.zip
# 安装readonlyrest
RUN ./bin/elasticsearch-plugin install https://github.com/HYY-yu/BezierCurveDemo/raw/master/readonlyrest-1.16.14_es6.0.0.zip

USER elasticsearch
CMD ./bin/elasticsearch
复制代码
```

这里对上面的操作做一下说明：

1. 首先在Dockerfile下的同级目录中需要建立一个conf文件夹，保存elasticsearch.yml文件（稍后给出）和keystore.jks。（jks是自签名文件，用于https，如何生成请自行搜索）
2. ik是一款很流行的中文分词库，使用它来支持中文搜索。
3. readonlyrest是一款开源的ES插件，用于用户管理、安全验证，土豪可以使用ES自带的X-pack包，有更完善的安全功能。

------

elactic配置 elasticsearch.yml

```less
cluster.name: "docker-cluster"
network.host: 0.0.0.0

# minimum_master_nodes need to be explicitly set when bound on a public IP
# set to 1 to allow single node clusters
# Details: https://github.com/elastic/elasticsearch/pull/17288
discovery.zen.minimum_master_nodes: 1

# 禁止系统对ES交换内存
bootstrap.memory_lock: true

http.type: ssl_netty4

readonlyrest:
  enable: true
  ssl:
    enable: true
    keystore_file: "server.jks"
    keystore_pass: server
    key_pass: server

  access_control_rules:

    - name: "Block 1 - ROOT"
      type: allow
      groups: ["admin"]

    - name: "User read only - paper"
      groups: ["user"]
      indices: ["paper*"]
      actions: ["indices:data/read/*"]

  users:

    - username: root
      auth_key_sha256: cb7c98bae153065db931980a13bd45ee3a77cb8f27a7dfee68f686377acc33f1
      groups: ["admin"]

    - username: xiaoming
      auth_key: xiaoming:xiaoming
      groups: ["user"]
复制代码
```

> 这里`bootstrap.memory_lock: true`是个坑，[禁止交换内存](https://link.juejin.cn?target=https%3A%2F%2Fwww.elastic.co%2Fguide%2Fen%2Felasticsearch%2Freference%2Fcurrent%2Fsetup-configuration-memory.html%23mlockall)这里文档已经说明了，有的os会在运行时把暂时不用的内存交换到硬盘的一块区域，然而这种行为会让ES的资源占用率飙升，甚至让系统无法响应。

配置文件里已经很明显了，一个root用户属于admin组，而admin有所有权限，xiaoming同学因为在user组，只能访问paper索引，并且只能读取，不能操作。更详细的配置请见：[readonlyrest文档](https://link.juejin.cn?target=https%3A%2F%2Fgithub.com%2Fbeshu-tech%2Freadonlyrest-docs%2Fblob%2Fmaster%2Felasticsearch.md)

至此，ES的准备工作算是做完了，`docker build -t ESImage:tag` 一下，`docker run -p 9200:9200 ESImage:Tag`跑起来。

如果https://127.0.0.1:9200/返回

```json
{
    "name": "VaKwrIR",
    "cluster_name": "docker-cluster",
    "cluster_uuid": "YsYdOWKvRh2swz907s2m_w",
    "version": {
        "number": "6.0.0",
        "build_hash": "8f0685b",
        "build_date": "2017-11-10T18:41:22.859Z",
        "build_snapshot": false,
        "lucene_version": "7.0.1",
        "minimum_wire_compatibility_version": "5.6.0",
        "minimum_index_compatibility_version": "5.0.0"
    },
    "tagline": "You Know, for Search"
}
复制代码
```

我们本次教程的主角算是出场了，分享几个常用的API调戏调试ES用：

> {{url}}替换成你本地的ES地址。

- 查看所有插件：{{url}}/_cat/plugins?v
- 查看所有索引：{{url}}/_cat/indices?v
- 对ES进行健康检查：{{url}}/_cat/health?v
- 查看当前的磁盘占用率：{{url}}/_cat/allocation?v

## 导入MYSQL数据

这里我使用的是MYSQL数据，其实其它的数据库也是一样，关键在于如何导入，网上教程会推荐Logstash、Beat、ES的mysql插件进行导入，我也都实验过，配置繁琐，文档稀少，要是数据库结构复杂一点，导入是个劳心劳神的活计，所以并不推荐。其实ES在各个语言都有对应的API库，你在语言层面把数据组装成json，通过API库发送到ES即可。流程大致如下：



![image](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2018/2/8/1617352fa33b70fa~tplv-t2oaga2asx-zoom-in-crop-mark:4536:0:0:0.awebp)



我使用的是Golang的ES库elastic，其它语言可以去github上自行搜索，操作的方式都是一样的。

接下来使用一个简单的数据库做介绍：

#### Paper表

| id   | name                 |
| ---- | -------------------- |
| 1    | 北京第一小学模拟卷   |
| 2    | 江西北京通用高考真题 |

#### Province表

| id   | name |
| ---- | ---- |
| 1    | 北京 |
| 2    | 江西 |

#### Paper_Province表

| paper_id | province_id |
| -------- | ----------- |
| 1        | 1           |
| 2        | 1           |
| 2        | 2           |

如上，Paper和Province是多对多关系，现在把Paper数据打入ES，，可以按Paper名称模糊搜索，也可通过Province进行筛选。json数据格式如下：

```json
{
    "id":1,
    "name": "北京第一小学模拟卷",
    "provinces":[
        {
            "id":1,
            "name":"北京"
        }
    ]
}
复制代码
```

首先准备一份mapping.json文件，这是在ES中数据的存储结构定义，

```json
{
    "mappings":{
        "docs":{
			"include_in_all": false, 
            "properties":{
                "id":{
                    "type":"long"
                },
                "name":{
                    "type":"text",
                    "analyzer":"ik_max_word" // 使用最大词分词器
                },
                "provinces":{
                    "type":"nested",
                    "properties":{
                        "id":{
                            "type":"integer"
                        },
                        "name":{
                            "type":"text",
                            "index":"false" // 不索引
                        }
                    }
                }
            }
        }
    },
    "settings":{
        "number_of_shards":1,
        "number_of_replicas":0
    }
}
复制代码
```

> 需要注意的是取消_all字段，这个默认的_all会收集所有的存储字段，实现无条件限制的搜索，缺点是空间占用大。

> shard（分片）数我设置为了1，没有设置replicas(副本)，毕竟这不是一个集群，处理的数据也不是很多，如果有大量数据需要处理可以自行设置分片和副本的数量。

首先与ES建立连接，ca.crt与jks自签名有关。当然，在这里我使用InsecureSkipVerify忽略了证书文件的验证。

```scss
func InitElasticSearch() {
	pool := x509.NewCertPool()
	crt, err0 := ioutil.ReadFile("conf/ca.crt")
	if err0 != nil {
		cannotOpenES(err0, "read crt file err")
		return
	}

	pool.AppendCertsFromPEM(crt)
	tr := &http.Transport{
		TLSClientConfig: &tls.Config{RootCAs: pool, InsecureSkipVerify: true},
	}
	httpClient := &http.Client{Transport: tr}

	//后台构造elasticClient
	var err error
	elasticClient, err = elastic.NewClient(elastic.SetURL(MyConfig.ElasticUrl),
		elastic.SetErrorLog(GetLogger()),
		elastic.SetGzip(true),
		elastic.SetHttpClient(httpClient),
		elastic.SetSniff(false), // 集群嗅探，单节点记得关闭。
		elastic.SetScheme("https"),
		elastic.SetBasicAuth(MyConfig.ElasticUsername, MyConfig.ElasticPassword))
	if err != nil {
		cannotOpenES(err, "search_client_error")
		return
	}
	//elasticClient构造完成

	//查询是否有paper索引
	exist, err := elasticClient.IndexExists(MyConfig.ElasticIndexName).Do(context.Background())
	if err != nil {
		cannotOpenES(err, "exist_paper_index_check")
		return
	}

	//索引存在且通过完整性检查则不发送任何数据
	if exist {
		if !isIndexIntegrity(elasticClient) {
			//删除当前索引&emsp;&emsp;准备重建
			deleteResponse, err := elasticClient.DeleteIndex(MyConfig.ElasticIndexName).Do(context.Background())
			if err != nil || !deleteResponse.Acknowledged {
				cannotOpenES(err, "delete_index_error")
				return
			}
		} else {
			return
		}
	}

	//后台查询数据库,发送数据到elasticsearch中
	go fetchDBGetAllPaperAndSendToES()
}
复制代码
type PaperSearch struct {
	PaperId    int64     `gorm:"primary_key;column:F_paper_id;type:BIGINT(20)" json:"id"`
	Name       string    `gorm:"column:F_name;size:80" json:"name"`
	Provinces  []Province `gorm:"many2many:t_paper_province;" json:"provinces"`        // 试卷适用的省份
}

func fetchDBGetAllPaperAndSendToES() {
	//fetch paper
	var allPaper []PaperSearch

	GetDb().Table("t_papers").Find(&allPaper)

	//province
	for i := range allPaper {
		var allPro []Province
		GetDb().Table("t_provinces").Joins("INNER JOIN `t_paper_province` ON `t_paper_province`.`province_F_province_id` = `t_provinces`.`F_province_id`").
			Where("t_paper_province.paper_F_paper_id = ?", allPaper[i].PaperId).Find(&allPro)
		allPaper[i].Provinces = allPro
	}

	if len(allPaper) > 0 {
		//send to es - create index
		createService := GetElasticSearch().CreateIndex(MyConfig.ElasticIndexName)
		// 此处的index_default_setting就是上面mapping.json中的内容。
		createService.Body(index_default_setting)
		createResult, err := createService.Do(context.Background())
		if err != nil {
			cannotOpenES(err, "create_paper_index")
			return
		}

		if !createResult.Acknowledged || !createResult.ShardsAcknowledged {
			cannotOpenES(err, "create_paper_index_fail")
		}

		// - send all paper
		bulkRequest := GetElasticSearch().Bulk()

		for i := range allPaper {
			indexReq := elastic.NewBulkIndexRequest().OpType("create").Index(MyConfig.ElasticIndexName).Type("docs").
				Id(helper.Int64ToString(allPaper[i].PaperId)).
				Doc(allPaper[i])

			bulkRequest.Add(indexReq)
		}

		// Do sends the bulk requests to Elasticsearch
		bulkResponse, err := bulkRequest.Do(context.Background())
		if err != nil {
			cannotOpenES(err, "insert_docs_error")
			return
		}

		// Bulk request actions get cleared
		if len(bulkResponse.Created()) != len(allPaper) {
			cannotOpenES(err, "insert_docs_nums_error")
			return
		}
		//send success
	}
}
复制代码
```

跑通上面的代码后，使用`{{url}}/_cat/indices?v`看看ES中是否出现了新创建的索引，使用`{{url}}/papers/_search`看看命中了多少文档，如果文档数等于你发送过去的数据量，搜索服务就算跑起来了。

## 搜索

现在就可以通过ProvinceID和q来搜索试卷，默认按照相关度评分排序。

```go
//q 搜索字符串 provinceID 限定省份id limit page 分页参数
func SearchPaper(q string, provinceId uint, limit int, page int) (list []PaperSearch, totalPage int, currentPage int, pageIsEnd int, returnErr error) {
	//不满足条件，使用数据库搜索
	if !CanUseElasticSearch && !MyConfig.UseElasticSearch {
		return SearchPaperLocal(q, courseId, gradeId, provinceId, paperTypeId, limit, page)
	}

	list = make([]PaperSimple, 0)
	totalPage = 0
	currentPage = page
	pageIsEnd = 0
	returnErr = nil

	client := GetElasticSearch()
	if client == nil {
		return SearchPaperLocal(q, courseId, gradeId, provinceId, paperTypeId, limit, page)
	}

	//ElasticSearch有问题，使用数据库搜索
	if !isIndexIntegrity(client) {
		return SearchPaperLocal(q, courseId, gradeId, provinceId, paperTypeId, limit, page)
	}

	if !client.IsRunning() {
		client.Start()
	}
	defer client.Stop()

	q = html.EscapeString(q)
	boolQuery := elastic.NewBoolQuery()
	// Paper.name
	matchQuery := elastic.NewMatchQuery("name", q)

	//省份
	if provinceId > 0 && provinceId != DEFAULT_PROVINCE_ALL {
		proBool := elastic.NewBoolQuery()
		tpro := elastic.NewTermQuery("provinces.id", provinceId)
		proNest := elastic.NewNestedQuery("provinces", proBool.Must(tpro))
		boolQuery.Must(proNest)
	}

	boolQuery.Must(matchQuery)

	for _, e := range termQuerys {
		boolQuery.Must(e)
	}

	highligt := elastic.NewHighlight()
	highligt.Field(ELASTIC_SEARCH_SEARCH_FIELD_NAME)
	highligt.PreTags(ELASTIC_SEARCH_SEARCH_FIELD_TAG_START)
	highligt.PostTags(ELASTIC_SEARCH_SEARCH_FIELD_TAG_END)
	searchResult, err2 := client.Search(MyConfig.ElasticIndexName).
		Highlight(highligt).
		Query(boolQuery).
		From((page - 1) * limit).
		Size(limit).
		Do(context.Background())

	if err2 != nil {
		// Handle error
		GetLogger().LogErr("搜索时出错 "+err2.Error(), "search_error")
		// Handle error
		returnErr = errors.New("搜索时出错")
	} else {
		if searchResult.Hits.TotalHits > 0 {
			// Iterate through results
			for _, hit := range searchResult.Hits.Hits {
				var p PaperSearch
				err := json.Unmarshal(*hit.Source, &p)
				if err != nil {
					// Deserialization failed
					GetLogger().LogErr("搜索时出错 "+err.Error(), "search_deserialization_error")
					returnErr = errors.New("搜索时出错")
					return
				}

				if len(hit.Highlight[ELASTIC_SEARCH_SEARCH_FIELD_NAME]) > 0 {
					p.Name = hit.Highlight[ELASTIC_SEARCH_SEARCH_FIELD_NAME][0]
				}

				list = append(list, p)
			}

			count := searchResult.TotalHits()

			currentPage = page
			if count > 0 {
				totalPage = int(math.Ceil(float64(count) / float64(limit)))
			}
			if currentPage >= totalPage {
				pageIsEnd = 1
			}
		} else {
			// No hits
		}
	}
	return
}
复制代码
```