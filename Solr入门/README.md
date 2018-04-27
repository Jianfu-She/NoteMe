# Solr入门

本文基于Solr6.6，结合IK-Analyzer中文分词器，介绍了如何利用Solr进行数据库导入、数据查询。最后调用Solrj接口展示了Solr在实际项目中的使用。

本文并不涉及Solr相关的理论知识，目的在于帮助没有接触过Solr的同学快速上手，在使用中发现问题，最后带着问题去有针对性的学习Solr。

## 1 基础概念
Solr是一个高性能，采用Java5开发，基于Lucene的全文搜索服务器。提供了比Lucene更为丰富的查询语言，同时实现了可配置、可扩展并对查询性能进行了优化，并且提供了一个完善的功能管理界面，是一款非常优秀的全文搜索引擎。

IK-Analyzer是一个开源的，基于java语言开发的轻量级的中文分词工具包，于2012年停止更新，在Solr6.x中使用时需要自己构造相应的工厂方法。IK-Analyzer采用了特有的正向迭代最细粒度切分算法，支持细粒度和智能分词两种切分模式。

## 2 相关配置
下载好Solr后，通过`solr create_collection -c mars`(或 `solr create_core -c mars`)创建一个名字为mars的核，mars文件位于“D:\solr-6.6.0\server\solr\mars”路径下。该路径下包含conf和data两个文件夹。配置相关的工作主要是针对于conf中的managed-schema与solrconfig.xml。

### 2.1 IK-Analyzer相关配置
Solr默认是不支持中文分词的，因此需要安装第三方工具包，我们选择IK-Analyzer。

在managed-schema中我们需要添加以下内容，注意“*_txt_ik”，很关键：

```
<!-- Chinese -->
<dynamicField name="*_txt_ik" type="text_ik"  indexed="true"  stored="true"/>
<fieldType name="text_ik" class="solr.TextField">
  <!-- 索引时候的分词器 -->
  <analyzer type="index">
    <tokenizer class="org.wltea.analyzer.util.IKTokenizerFactory" useSmart="false"/>
    <filter class="solr.LowerCaseFilterFactory"/>
  </analyzer>
  <!-- 查询时候的分词器 -->
  <analyzer type="query">
    <tokenizer class="org.wltea.analyzer.util.IKTokenizerFactory" useSmart="true"/>
  </analyzer>
</fieldType>
```

text_ik就是我们新给出的中文分词方式。由于IK-Analyzer在2012年就停止更新了，所以IKTokenizerFactory这个工厂方法需要我们自己去写，同时将新的IK-Analyzer打成Jar包，导入Solr。

原IK-Analyzer代码中需要作出的更改如下：

IKAnalyzer.java
```java
public final class IKAnalyzer extends Analyzer {
  @Override
  protected TokenStreamComponents createComponents(String fieldName) {
    Tokenizer _IKTokenizer = new IKTokenizer(this.useSmart);
    return new TokenStreamComponents(_IKTokenizer);
  }
}
```

IKTokenizerFactory.java
```java
public class IKTokenizerFactory extends TokenizerFactory {
  private boolean useSmart;  

  public IKTokenizerFactory(Map<String, String> args) {  
        super(args);  
        useSmart = getBoolean(args, "useSmart", false);  
    }  
  
    @Override  
    public Tokenizer create(AttributeFactory attributeFactory) {  
        Tokenizer tokenizer = new IKTokenizer(useSmart);
        return tokenizer;  
    }
}
```

同时，pom.xml需要配置如下项来导入原代码放在外面的dic文件：
```
<build>
    <resources>
        <resource>
            <directory>src/main/java</directory>
            <includes>
                <include>**/*.dic</include>
            </includes>
        </resource>
        <resource>
            <directory>src/main/resources</directory>
            <includes>
                <include>**/*.dic</include>
                <include>**/*.xml</include>
            </includes>
        </resource>
    </resources>
</build>
```

### 2.2 导入数据库
首先，为了在Solr中导入数据库的数据，我们需要先导入一些Jar包，在solrconfig.xml中做如下配置：
```
<lib dir="${solr.install.dir:../../../..}/ext/ikanalyzer" regex=".*\.jar" />
<lib dir="${solr.install.dir:../../../..}/ext/mysql" regex=".*\.jar" />
<lib dir="${solr.install.dir:../../../..}/dist/" regex="solr-dataimporthandler-\d.*\.jar" />

<!-- 自己导入的-全量索引 -->
<requestHandler name="/dataimport" class="solr.DataImportHandler">
  <lst name="defaults">
    <str name="config">solr-data-config.xml</str>
  </lst>
</requestHandler>

<!-- 自己导入的-增量索引 -->
<requestHandler name="/deltaimport" class="solr.DataImportHandler">
  <lst name="defaults">
    <str name="config">delta-data-config.xml</str>
  </lst>
</requestHandler>
```

ext是自建的文件夹，第一行导入的Jar包就是我们之前自己打包的ik-analyzer-6.6.0.jar。第二行导入的Jar包是mysql-connector-java-5.1.39.jar。

在Conf文件夹下新建solr-data-config.xml与delta-data-config.xml，分别代表全量索引与增量索引的配置项。

本例中，solr-data-config.xml如下：
```
<dataConfig>
    <dataSource type="JdbcDataSource" 
              driver="com.mysql.jdbc.Driver"
              url="jdbc:mysql://127.0.0.1:3306/moontest?useSSL=false" 
              user="root" 
              password="password"/>
    <document>
        <entity name="t_user" pk="id" query="SELECT id, nickname, tel, shop_name FROM t_user WHERE user_type = 1 ORDER BY id DESC ">
            <field column="nickname" name="nickname_txt_ik"/>
            <field column="tel" name="tel_txt_ik"/>
            <field column="shop_name" name="shopName_txt_ik"/>
        </entity>
    </document>
</dataConfig>
```
本例中，delta-data-config.xml如下：
```
<dataConfig>
  <dataSource type="JdbcDataSource" 
              driver="com.mysql.jdbc.Driver"
              url="jdbc:mysql://127.0.0.1:3306/moontest?useSSL=false" 
              user="root" 
              password="password"/>
    <document>
        <entity name="t_user" pk="id" query="SELECT id, nickname, tel, shop_name FROM t_user WHERE user_type = 1 ORDER BY id DESC "
          deltaImportQuery="SELECT id, nickname, tel, shop_name FROM t_user WHERE id = '${dih.delta.id}' AND user_type = 1 ORDER BY id DESC "
          deltaQuery="SELECT id FROM t_user WHERE create_time > '${dih.last_index_time}'"
          transformer="RegexTransformer">
            <field column="nickname" name="nickname_txt_ik"/>
            <field column="tel" name="tel_txt_ik"/>
            <field column="shop_name" name="shopName_txt_ik"/>
        </entity>
    </document>
</dataConfig>
```
至此，Solr单机版的配置就基本完成了。

在浏览器打开http://localhost:8983，进行数据导入、IK分词效果、数据查询的操作图像如下：

![](https://github.com/Jianfu-She/NoteMe/blob/master/pic/Solr入门-1.png)

![](https://github.com/Jianfu-She/NoteMe/blob/master/pic/Solr入门-2.png)

![](https://github.com/Jianfu-She/NoteMe/blob/master/pic/Solr入门-3.png)

## 3 实例
我把Solr用到了自己的一个项目中，这个项目中有一个对分销商的搜索，可以通过账号、分销商名、联系电话找到分销商。以前是用MySQL的LIKE语句实现的，很low，改用Solr后能够应对以后更大的数据量、提供更快捷准确的搜索服务。

下面给出service层的核心代码，结合上面的solr-data-config.xml就很容易看懂了：
```java
@Service
public class SearchServiceImpl implements SearchService {

    private static final String SOLR_URL = "http://127.0.0.1:8983/solr/mars";
    private HttpSolrClient client = new HttpSolrClient.Builder(SOLR_URL).build();
    private static final String NICKNAME_FIELD = "nickname_txt_ik";
    private static final String TEL_FIELD = "tel_txt_ik";
    private static final String SHOPNAME_FIELD = "shopName_txt_ik";

    @Resource
    private UserMapper userMapper;

    @Override
    // 搜索
    public PageData searchDistributorData(String key, int curPage, int pageSize, String hlPre, String hlPos) throws Exception {

        if (key == null || ("").equals(key))
            key = "*:*";

        List<DistributorData> distributorDataList = new ArrayList<>();

        // 按照key进行搜索
        SolrQuery solrQuery = new SolrQuery(key);
        // 搜索的数量
        solrQuery.setRows(pageSize);
        // 搜索的起始位置
        solrQuery.setStart((curPage - 1) * pageSize);
        // 搜索结果高亮相关
        solrQuery.setHighlight(true);
        solrQuery.setHighlightSimplePre(hlPre);
        solrQuery.setHighlightSimplePost(hlPos);
        // solr默认分词为text_general，可以改为text_ik，这样就不要写下面这行了。如果没改就需要写下面的Field List(fl)
        solrQuery.set("hl.fl", NICKNAME_FIELD + "," + TEL_FIELD + "," + SHOPNAME_FIELD);

        // 过滤器，-表示不要
        solrQuery.setFilterQueries("-(id = 1)");

        // 搜索
        QueryResponse response = client.query(solrQuery);

        // 本工程中要求搜索结果按id逆序给出，故采用TreeMap，与Solrj无关
        TreeMap<String, Map<String, List<String>>> treeMap = new TreeMap<>((String o1, String o2) -> Integer.parseInt(o2) - Integer.parseInt(o1));
        for (Map.Entry<String, Map<String, List<String>>> entry : response.getHighlighting().entrySet()) {
            treeMap.put(entry.getKey(), entry.getValue());
        }

        for (Map.Entry<String, Map<String, List<String>>> entry : treeMap.entrySet()) {

            DistributorData distributorData = new DistributorData();
            DistributorData user = userMapper.selectDistributorDataById(Long.parseLong(entry.getKey()));

            distributorData.setOpenid(user.getOpenid());
            distributorData.setMoneySum(user.getMoneySum());
            distributorData.setRank(user.getRank());
            distributorData.setStatus(user.getStatus());

            // 如果账户包含搜索词，则高亮账户中的搜索词
            if (entry.getValue().containsKey(NICKNAME_FIELD)) {
                List<String> nicknameList = entry.getValue().get(NICKNAME_FIELD);
                if (nicknameList.size() > 0)
                    distributorData.setNickName(nicknameList.get(0));
            } else {
                distributorData.setNickName(user.getNickName());
            }

            // 如果手机号包含搜索词，则高亮手机号中的搜索词
            if (entry.getValue().containsKey(TEL_FIELD)) {
                List<String> telList = entry.getValue().get(TEL_FIELD);
                if (telList.size() > 0)
                    distributorData.setTel(telList.get(0));
            } else {
                distributorData.setTel(user.getTel());
            }

            // 如果账户包含搜索词，则高亮账户中的搜索词
            if (entry.getValue().containsKey(SHOPNAME_FIELD)) {
                List<String> shopNameList = entry.getValue().get(SHOPNAME_FIELD);
                if (shopNameList.size() > 0)
                    distributorData.setShopName(shopNameList.get(0));
            } else {
                distributorData.setShopName(user.getShopName());
            }

            distributorDataList.add(distributorData);
        }

        int total = (int) response.getResults().getNumFound();
        return new PageData<>(total, curPage, (int) Math.ceil((double) total / pageSize), pageSize, distributorDataList);
    }

    @Override
    // 建索引
    public boolean indexDistributorData(long id, String nickname, String tel, String shopName) throws Exception {
        SolrInputDocument doc =  new SolrInputDocument();
        doc.setField("id", id);
        doc.setField(NICKNAME_FIELD, nickname);
        doc.setField(TEL_FIELD, tel);
        doc.setField(SHOPNAME_FIELD, shopName);
        UpdateResponse response = client.add(doc, 1000);
        return response != null && response.getStatus() == 0;
    }
}
```

实际效果如下图：

!()[https://github.com/Jianfu-She/NoteMe/blob/master/pic/Solr入门-4.png]

!()[https://github.com/Jianfu-She/NoteMe/blob/master/pic/Solr入门-5.png]

!()[https://github.com/Jianfu-She/NoteMe/blob/master/pic/Solr入门-6.png]

!()[https://github.com/Jianfu-She/NoteMe/blob/master/pic/Solr入门-7.png]