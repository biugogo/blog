---
title: Mongo分表动态索引建立
date: 2018-6-24 16:12:06
tags:
 -DB
categories: DB
thumbnail: https://raw.githubusercontent.com/apollochen123/image/master/%E9%BB%98%E8%AE%A42.jpg
---

# Mongo分表动态索引建立
-----------
工作中经常遇到Mongo需要分表的情况。但是分表后，不能再使用spring date的@Document系列注解为mongo建立model和索引了。这时候需要动态建立索引和指定collection。

-------
其实很简单，这里主要是做个笔记，免得忘记。
```java
//PostConstruct注解执行在bean注入之后
@PostConstruct
private void init() {
    //建立索引字段
    Document document = new Document().append(UID, 1).append(ROOM_ID, 1);
    //配置建立索引
    Index idxUidRoomId = new CompoundIndexDefinition(document).background().named("idx_uid_roomId").unique();
    //建立,createAllCollectionName()是需要建立索引的collection name list
    createAllCollectionName().stream().map(mongoTemplate::indexOps)
            .forEach(it -> it.ensureIndex(idxUidRoomId));
}
```
save
```java
public void saveTagsAndScore(Long toUid, Long roomId, Integer score, List<String> tags) {
    Query query = Query.query(Criteria.where(UID).is(toUid).and(ROOM_ID).is(roomId));

    Update update = new Update();
    update.inc(SCORE_TIMES, 1);
    update.inc(SCORE, score);
    tags.stream().forEach(tag -> {
        update.inc(TAGS + "." + tag, 1);
    });

    mongoTemplate.upsert(query, update, TagsAndScoreEntity.class, createTagsAndScoreCollectionName(roomId));
}
```
Query
```java
public List<TagsAndScoreEntity> findTagsAndScore(List<Long> uids, Long roomID) {
    Query query = Query.query(Criteria.where(UID).in(uids).and(ROOM_ID).is(roomID));

    return mongoTemplate.find(query, TagsAndScoreEntity.class, createTagsAndScoreCollectionName(roomID));
}
```

常用的两种分表方式，按照时间和按照uid取余.
```java
private static final int TAG_SAND_SCORE_SHARDING_NUM = 10;

private static final String SCORE_DETAILS_COLLECTION_NAME_PREFIX = "score_details_";
private static final String TAGS_AND_SCORE_COL_NAME_PREFIX = "tags_and_score_";

public static String createScoreDetailCollectionName() {
    return SCORE_DETAILS_COLLECTION_NAME_PREFIX + LocalDate.now().format(DateHelper.yyyyMM);
}

public static String createTagsAndScoreCollectionName(long roomId) {
    return TAGS_AND_SCORE_COL_NAME_PREFIX + (roomId % TAG_SAND_SCORE_SHARDING_NUM);
}
```
LocalDate这个类挺好用的。很简便。
