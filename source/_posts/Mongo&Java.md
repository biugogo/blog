---
title: Mongo with Spring
date: 2018-04-5 10:47:51
tags:
 -DB
categories: DB
thumbnail: https://raw.githubusercontent.com/apollochen123/image/master/%E9%BB%98%E8%AE%A42.jpg
---
# Mongo
---------
最近因为项目里用到了Mongo，这里使用Java对Mongo做一下简单的操作。

---------------
## Simple demo connect to mongo
直接上代码：
```java
package com.yy.wolfkill.Apollo;

import java.util.ArrayList;
import java.util.List;

import org.bson.Document;

import com.mongodb.MongoClient;
import com.mongodb.client.FindIterable;
import com.mongodb.client.MongoCollection;
import com.mongodb.client.MongoCursor;
import com.mongodb.client.MongoDatabase;
import com.mongodb.client.model.Filters;

public class MongoJDBC {
    public static void main(String[] args) {
        try {
            //建立连接
            MongoClient mongo = new MongoClient("localhost",27017 );
            System.out.println("Connect to database successfully");
            //选择DB
            MongoDatabase database = mongo.getDatabase("chechaodb");

            //创建一个collection
            database.createCollection("javaTest");
            System.out.println("创建集合成功");
            //选择connection
            MongoCollection<Document> collection= database.getCollection("javaTest");
            System.out.println("选择集合成功");
            //新建一个Document
            Document document = new Document().append("name", "ApolloJava5").append("address", "YY5");
            List<String> phoneList = new ArrayList<>();
            phoneList.add("123");
            phoneList.add("222");
            document.append("phones", phoneList);

            //插入Document
            collection.insertOne(document);
            System.out.println("文档插入成功");
            findAll(collection);
            update(collection);
            findAll(collection);

            collection.deleteMany(Filters.gt("name", "ApolloJava3"));
            findAll(collection);
        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    public static void findAll(MongoCollection<Document> collection) {
        FindIterable<Document> findIterable = collection.find().sort(new Document("name",-1));
        MongoCursor<Document> mongoCusor = findIterable.iterator();
        while(mongoCusor.hasNext()) {
            System.out.println(mongoCusor.next());
        }
    }
    public static void update(MongoCollection<Document> collection) {
        collection.updateMany(Filters.eq("name","ApolloJava"), new Document("$set",new Document("name","ApolloUpdated")));
    }
}

```

------
## Mongo整合与springboot
1. pom配置
  ```
  <dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-data-mongodb</artifactId>
</dependency>
  ```
2. yml配置mongo连接信息
  ```yml
  server:
    port: 8828
spring:
    name: demointegration
    data:
        mongodb:
            url: localhost:27017
            database: test
## MONGODB (MongoProperties)
#spring.data.mongodb.authentication-database= # Authentication database name.
#spring.data.mongodb.database=test # Database name.
#spring.data.mongodb.field-naming-strategy= # Fully qualified name of the FieldNamingStrategy to use.
#spring.data.mongodb.grid-fs-database= # GridFS database name.
#spring.data.mongodb.host=localhost # Mongo server host.
#spring.data.mongodb.password= # Login password of the mongo server.
#spring.data.mongodb.port=27017 # Mongo server port.
#spring.data.mongodb.repositories.enabled=true # Enable Mongo repositories.
#spring.data.mongodb.uri=mongodb://localhost/test # Mongo database URI. When set, host and port are ignored.
#spring.data.mongodb.username= # Login user of the mongo server.

  ```
3. 使用MongoTemplate
  ```java
  public class UserMongoTemplateImpl implements UserMongoTemplate<User> {

    @Autowired
    private MongoTemplate mongoTemplate;
    public void save(String a){
       mongoTemplate.save(a);
       ....
    }
  ```
---------------
## spring MongoRepository与MongoTemplate整合实现快速开发

spring提供了方便的MongoRepository，可以像JPA repositorie一样提供很多预先写好的方法，很是方便。但是某些地方又想使用MongoTemplate的方式。这里可以通过一种方式把他们整合起来

1. 首先我们写一个接口，这个接口里是你想用MongoTemplate实现的方法
   ```java
   package com.yy.Apollo.demointegration.mongo;

   import java.util.List;
   import org.springframework.data.domain.Pageable;
   import org.springframework.data.domain.Sort;

   public interface UserMongoTemplate<T> {
       public List<T> search(T t, Pageable pageable,Sort sort);
   }

   ```
2. 再实现这个使用MongoTemplate查询接口
   ```java
   package com.yy.Apollo.demointegration.mongo;
   import java.util.ArrayList;
   import java.util.List;
   import org.springframework.beans.factory.annotation.Autowired;
   import org.springframework.data.domain.Pageable;
   import org.springframework.data.domain.Sort;
   import org.springframework.data.mongodb.core.MongoTemplate;
   import org.springframework.data.mongodb.core.query.Criteria;
   import org.springframework.data.mongodb.core.query.Query;

   import com.yy.Apollo.demointegration.mongo.entity.User;

   public class UserMongoTemplateImpl implements UserMongoTemplate<User> {

       @Autowired
       private MongoTemplate mongoTemplate;

       @Override
       public List<User> search(User t, Pageable pageable, Sort sort) {
           List<Criteria> criterias = new ArrayList<>();
           if (null != t.getId()) {
               Criteria criteria = Criteria.where(User.FILED.FILED_ID).regex(t.getId());
               criterias.add(criteria);
           }
           if (null != t.getName()) {
               Criteria criteria = Criteria.where(User.FILED.FILED_NAME).is(t.getName());
               criterias.add(criteria);
           }
           Criteria or = new Criteria();
           or.orOperator(criterias.toArray(new Criteria[criterias.size()]));
           Query query = Query.query(or);
           if (null != pageable) {
               query.with(pageable);
           }
           if (null != sort) {
               query.with(sort);
           }
           return mongoTemplate.find(query, User.class);
       }
   }
   ```
3. 写你真正使用的Repository,同时继承UserMongoTemplate与标准MongoRepository两个接口，这里有一些MongoRepository的规范写法
   ```java
   package com.yy.Apollo.demointegration.mongo;
   import java.util.List;

   import org.springframework.data.domain.Page;
   import org.springframework.data.domain.Pageable;
   import org.springframework.data.domain.Sort;
   import org.springframework.data.mongodb.repository.MongoRepository;
   import com.yy.Apollo.demointegration.mongo.entity.User;

   public interface UserRepository extends MongoRepository<User, String>,UserMongoTemplate<User> {
       void deleteById(String id);

       List<String> deleteByName(String name);

       List<User> findByNameAndPhone(String name, String phone);

       /**
        * Distinct + or
        */
       List<User> findDistinctUserByNameOrPhone(String name, String phone);

       /**
        * IgnoreCase
        *
        * @param name
        * @return
        */
       List<User> findByNameIgnoreCase(String name);

       List<User> findByNameAndPhoneAllIgnoreCase(String lastname, String firstname);

       /**
        * order
        */
       List<User> findByNameOrderByPhoneAsc(String name);

       /**
        * A.B式变量查询
        */
       List<User> findByAddress_Home(String home);

       /**
        * Pageable
        */
       Page<User> findByName(String name, Pageable pageable);

       // List<User> findByName(String name, Pageable pageable);

       List<User> findByName(String name, Sort sort);

    }
   ```
4. 使用查询,我们这里分别调用Repository的自带方法和我们自己实现的MongoTemplate方法，都是成功的。
  ```java
  package com.yy.Apollo.demointegration.controller;
  import java.util.HashMap;
  import java.util.List;
  import java.util.Map;
  import org.springframework.beans.factory.annotation.Autowired;
  import org.springframework.data.domain.Sort;
  import org.springframework.web.bind.annotation.GetMapping;
  import org.springframework.web.bind.annotation.PathVariable;
  import org.springframework.web.bind.annotation.PostMapping;
  import org.springframework.web.bind.annotation.RequestBody;
  import org.springframework.web.bind.annotation.RequestMapping;
  import org.springframework.web.bind.annotation.RequestParam;
  import org.springframework.web.bind.annotation.RestController;

  import com.yy.Apollo.demointegration.mongo.UserRepository;
  import com.yy.Apollo.demointegration.mongo.entity.User;
  import com.yy.Apollo.demointegration.mongo.entity.UserOut;
  import com.yy.Apollo.demointegration.rabbitmq.MessageSend;

  @RestController
  @RequestMapping("/mongo")
  public class TestController {

     @Autowired
     private UserRepository useRepository;

     @GetMapping("/testft")
     public Map<String, List<User>> testFindTogether(@RequestParam("name")String name) {
         Sort sort = Sort.by(new Sort.Order(Sort.Direction.ASC, User.FILED.FILED_NAME));
         List<User> repo = useRepository.findByName(name, sort);
         User user = new User();
         user.setName(name);
         List<User> mine = useRepository.search(user, null, sort);
         Map<String, List<User>> result = new HashMap<>();
         result.put("repo", repo);
         result.put("mine", mine);
         return result;
     }

     @PostMapping("/save")
     public User save(@RequestBody UserOut userOut) {
         User user = new User(userOut);
         return useRepository.save(user);
     }

  }

  ```
