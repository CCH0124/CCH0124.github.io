---
title: Spring boot - monogoDB
date: 2021-10-30
description: "Spring boot - monogoDB"
tags: ["Spring boot", "monogoDB"]
draft: true
---

MongoDB 是一個 NoSQL 實現。NoSQL 在具有高吞吐量的應用程序中可以非常高性能。
- 沒有架構和關系
- 和 SQL 都有 Database
- NoSQL 表配稱為集合（collections）
- 集合中沒有 records，而是文檔（Document），像 javascript 的 Object
- 可以在同一個集合中存儲具有不同結構的多個文檔
- 可以存儲通常相同但某些字段可能不同的文檔

如果數據發生變化，我們必須在多個地方更新它。我們檢索數據，不必 Join（Mysql join）多個表，雖然現在的 API 可以使用 Join 但應該盡量避免。

簡單的使用 docker-compose 建立環境
```yaml
version: '3.7'
services:
    mongo:
        image: mongo:4
        restart: always
        environment:
            MONGO_INITDB_ROOT_USERNAME: root
            MONGO_INITDB_ROOT_PASSWORD: '00000000'
        ports:
            - 27017:27017
        volumes:
            - data-volume:/data/db
            - config-volume:/data/configdb

    mongo-express:
        image: mongo-express
        restart: always
        ports:
            - 8081:8081
        environment:
            ME_CONFIG_MONGODB_ADMINUSERNAME: root
            ME_CONFIG_MONGODB_ADMINPASSWORD: '00000000'
            ME_CONFIG_MONGODB_URL: mongodb://root:00000000@mongo:27017/
volumes:
    data-volume:
    config-volume:
```

在 Spring boot 的 pom.xml 加入以下
```xml
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-data-mongodb</artifactId>
		</dependency>
```


配置 Spring boot 連線 MongoDB 的資訊

```java
@Slf4j
@Configuration
public class MongoConfig extends AbstractMongoClientConfiguration {

    @Value("${mongo_db}")
    private String database;

    @Value("${mongo_hosts}")
    private String hosts;

    @Value("${mongo_ports}")
    private String ports;

    @Value("${mongo_user_name}")
    private String user;

    @Value("${mongo_password}")
    private String password;

    @Override
    protected String getDatabaseName() {
        // TODO Auto-generated method stub
        return database;
    }

    @Override
    public MongoClient mongoClient() {
        // TODO Auto-generated method stub
        // 如果有配置使用者以及帳號密碼
        MongoCredential credential = MongoCredential.createScramSha256Credential(user, database, password.toCharArray());
        // createScramSha256Credential(user, database, password.toCharArray());

        /**
         * 當存在 cluster 架構時
         **/
        List<ServerAddress> serverAddresses = new ArrayList<>();
        List<String> hostList = Arrays.asList(hosts.split(","));
        List<String> portList = Arrays.asList(ports.split(","));
        for (String host : hostList) {
            Integer index = hostList.indexOf(host);
            Integer port = Integer.parseInt(portList.get(index));

            ServerAddress serverAddress = new ServerAddress(host, port);
            serverAddresses.add(serverAddress);
        }

        MongoClient mongoClients = MongoClients.create(
            MongoClientSettings.builder()
            .applyToClusterSettings(builder -> builder.hosts(serverAddresses))
            .credential(credential)
            .build());

        log.info("Mongodb Server Addresses: {}", serverAddresses.toString());
        log.info("Mongo Client: {}", mongoClients.getDatabase(database));

        /**
         * 從 database 獲取 gnss collection
         */
        // MongoDatabase database = mongoClients.getDatabase(getDatabaseName());
        // MongoCollection<Document> coll = database.getCollection("gnss");
        // coll.find().forEach(d -> System.out.println(d.toJson()));

        return mongoClients;

    }

    @Override
    protected Collection<String> getMappingBasePackages() {
        // TODO Auto-generated method stub
        // 讓其可以得知存取 mongo 的位置
        return Collections.singleton("com.example.cch.mongo");
    }
    
    @Bean
    public MongoTemplate mongoTemplate() {
        return new MongoTemplate(mongoClient(), getDatabaseName());
    }
}
```

在 `application.properties` 配置環境變數

```
mongo_db=aiot
mongo_hosts=172.17.8.222 # 如果是  cluster 時以 , 分開
# 
mongo_ports=27017
# 
mongo_user_name=root
mongo_password=00000000
# spring.data.mongodb.auto-index-creation=true
```

將以下註解打開即可測試 Spring boot 是否能對 MongoDB 進行交互

```java
        MongoDatabase database = mongoClients.getDatabase(getDatabaseName());
        MongoCollection<Document> coll = database.getCollection("gnss");
        coll.find().forEach(d -> System.out.println(d.toJson()));
```

mongo-express 是一個可以操作 MongoDB 的 WebGUI 介面不論是創建 index 或是 collection 都相當方便


## 宣告 MongoDB 的 document
以下透過 `@Document` 方式建立 `document`，其 entity 會對應要儲存的欄位，`Field` 可以幫助與 mongoDB 中定義的 field 進行映射。
```java
@Data
@AllArgsConstructor
@NoArgsConstructor
@Document(collection = "connectivity")
public class Connectivity {
    private String name;

    private Integer number;
    
    private String status;
    
    private Integer signalStrength;
    
    @Field("IPlist") 
    private List<Map<String, Object>> ipList;
    
    private Integer contentRevision;
    
    private String deviceId;
    
    private Long contentTimestamp;
}
```

##  MongoDB 交互

那接下來我們要怎樣對 MongoDB 進行一些操作呢 ? 在 Spring boot 中也提供 API 給我們使用如下，透過繼層 `MongoRepository` 我們就可以有基本的 CRUD 操作

```java
@Repository
public interface GnssRepository extends MongoRepository<Gnss, String> {
    @Query("{'deviceId' : ?0 }")
    List<GnssMappingVO> getByDeviceId(@Param("id") String id); // 透過裝置 ID 獲取其相關資訊
}
```

在 `Service` 層就可以方便使用

```java
    @Override
    public List<Gnss> getAllGnss() {
        // basic query
        return gnssRepository.findAll();
    }

    @Override
    public Gnss getById(String id) {
        // basic query
        log.debug("Find Gnss by id:{}", id);
        return gnssRepository.findById(id).orElse(null);
    }
```

複雜一點有可以透過 `mongoTemplate` 進行查詢實現

```java
    @Override
    public List<GnssMappingVO> getListDeviceIdCustom(String id) {
        // Use Query method
        Query query = new Query();
        query.fields().include("id").include("sensorName").include("coords").include("deviceId"); // like SQL SELECT
        query.addCriteria(Criteria.where("deviceId").is(id)); // WHERE condition
        query.with(Sort.by(Sort.Direction.ASC, "sensorName", "deviceId")); // like SQL ORDER BY
        
        List<Gnss> gnssList = mongoTemplate.find(query, Gnss.class);
        List<GnssMappingVO> res = new ArrayList<>();
        /**
         * 回傳資料而外處理
         */
        for (int i=0; i<gnssList.size(); i++){
            GnssMappingVO gnssMappingVO = new GnssMappingVO().builder()
                .id(gnssList.get(i).getId())
                .sensorName(gnssList.get(i).getSensorName())
                .coords(gnssList.get(i).getCoords())
                .deviceId(gnssList.get(i).getDeviceId())
                .build();

                res.add(gnssMappingVO);
        }

        return res;
    }
```
另一種範例

```java
    @Override
    public List<Gnss> getListByTimestampRangeBson(GnssListRequestVO queryVO) {
        String deviceId_column = "deviceId";
        String timestamp_column = "timestamp";
        String gps_column = "coords";
        /**
         * User Bson query
         */
        Bson bson = Filters.and(Arrays.asList(
                Filters.eq(deviceId_column, queryVO.getQuery().getDeviceId()),
                Filters.gte(timestamp_column, queryVO.getQuery().getStart().getTime()),
                Filters.lte(timestamp_column, queryVO.getQuery().getEnd().getTime()))); // Like Where condition
        Bson match = Aggregates.match(bson);
        Bson sort = Aggregates.sort(Sorts.ascending(timestamp_column)); // sort
        Bson fields = Aggregates.project(Projections.include(deviceId_column, timestamp_column, gps_column)); // Like SELECT

        AggregateIterable<Document> findIterable = mongoTemplate.getCollection(collection).aggregate(Arrays.asList(match, fields, sort));

        Gson gson = new Gson(); // 轉 JSON
        JsonWriterSettings settings = JsonWriterSettings.builder()
                .dateTimeConverter(new JsonDateTimeCoverter())
                .int64Converter((value, writer) -> writer.writeNumber(value.toString()))
                .build();
        
        List<Gnss> res = StreamSupport.stream(findIterable.spliterator(), false)
                .map(i -> gson.fromJson(i.toJson(settings), Gnss.class))
                .collect(Collectors.toList());

        return res;
    }
```

而 JPA 提供一個強大的使用，以減少語法撰寫，如下透過方法的定義來實現對 MongoDB 的查詢。

```java
@Repository
public interface GnssRepository extends MongoRepository<Gnss, String> {
    List<Gnss> findByDeviceId(@Param("id") String id); // 透過裝置 ID 獲取其相關資訊
    Long countByDeviceId(@Param("id") String id); // 計算 mongo 裡面其傳入的裝置 ID 有幾筆資料
    List<Gnss> findByDeviceIdSortByContentTimestampDesc(String firstname); // 透過裝置 ID 獲取其相關資訊同時使用 ContentTimestamp 進行排序
}
```

官方提供使用方法進行查詢的一些節錄

- findAll: Query for a list of objects of type from the collection.T

- findOne: Map the results of an ad-hoc query on the collection to a single instance of an object of the specified type.

- findById: Return an object of the given ID and target class.

- find: Map the results of an ad-hoc query on the collection to a of the specified type.List

- findAndRemove: Map the results of an ad-hoc query on the collection to a single instance of an object of the specified type. The first document that matches the query is returned and removed from the collection in the database.

更詳細可以參考其官方提供的[資源](https://docs.spring.io/spring-data/mongodb/docs/3.2.6/reference/html/#introduction)
