---
title: Protobuf 与 JSON 的互相转换
copyright_author: Jitwxs
categories:
  - Java
  - Protobuf
tags: Protobuf
abbrlink: e5e60178
date: 2021-08-29 19:42:45
related_repos:
  - name: protobuf-sample
    url: https://github.com/jitwxs/blog-sample/tree/master/javase-sample/protobuf-sample
    rel: nofollow noopener noreferrer
    target: _blank
---

## 一、前言

Protobuf 默认并没有提供跟 JSON 的互相转换方法，虽然 Protobuf 对象自身有 `toString()` 方法，但是并非是 JSON 格式，而是形如：

```protobuf
age: 57
name: "urooP"
sex: MALE
grade {
  key: 1
  value {
    score: 2.589357441994722
    rank: 32
  }
}
parent {
  relation: "father"
  tel: "3286647499263"
}
```

本篇文章中，我将使用 `protobuf-java-util` 实现 Protobuf 对象的 JSON 序列化，使用 `fastjson` 实现反序列化，在文章最后，我编写了一个 fastjson 的转换器，来帮助大家更加优雅的实现 Protobuf 的序列化与反序列化。

## 二、序列化与反序列化

### 2.1 基础配置

首先需要引入 `protobuf-java-util` 依赖，其版本号跟 `protobuf-java` 一致即可，例如：

```xml
<dependency>
    <groupId>com.google.protobuf</groupId>
    <artifactId>protobuf-java</artifactId>
    <version>3.17.3</version>
</dependency>
<dependency>
    <groupId>com.google.protobuf</groupId>
    <artifactId>protobuf-java-util</artifactId>
    <version>3.17.3</version>
</dependency>
```

然后新建一个工具类 `ProtobufUtils`，在其中以静态的方式初始化 `JsonFormat.Printer` 和 `JsonFormat.Parser`，前者用于序列化，后者用于反序列化。

```java
public class ProtobufUtils {
    private static final JsonFormat.Printer printer;
    private static final JsonFormat.Parser parser;

    static {
        JsonFormat.TypeRegistry registry = JsonFormat.TypeRegistry.newBuilder()
            .add(StringValue.getDescriptor())
            .build();
        
        printer = JsonFormat
            .printer()
            .usingTypeRegistry(registry)
            .includingDefaultValueFields()
            .omittingInsignificantWhitespace();

        parser = JsonFormat
            .parser()
            .usingTypeRegistry(registry);
    }
}
```

### 2.2 用例准备

为了便于验证，这里先定义好 Proto，后续用于测试：

```protobuf
// enums.proto
syntax = "proto3";

option java_package = "com.github.jitwxs.sample.protobuf";
option java_outer_classname = "EnumMessageProto";

enum SexEnum {
    DEFAULT_SEX = 0;
    MALE = 1;
    FEMALE = 2;
}

enum SubjectEnum {
    DEFAULT_SUBJECT = 0;
    CHINESE = 1;
    MATH = 2;
    ENGLISH = 3;
}
```

```protobuf
// user.proto
syntax = "proto3";

import "enums.proto";

option java_package = "com.github.jitwxs.sample.protobuf";
option java_outer_classname = "MessageProto";

message User {
    int32 age = 1;
    string name = 2;
    SexEnum sex = 3;
    map<int32, GradeInfo> grade = 4;
    repeated ParentUser parent = 5;
}

message GradeInfo {
    double score = 1;
    int32 rank = 2;
}

message ParentUser {
    string relation = 1;
    string tel = 2;
}
```

提供一个随机创建 Protobuf 的方法（使用到了 `commons-lang3` 包）：

```java
protected MessageProto.User randomUser() {
    final Map<Integer, MessageProto.GradeInfo> gradeInfoMap = new HashMap<>();

    for (EnumMessageProto.SubjectEnum subjectEnum : EnumMessageProto.SubjectEnum.values()) {
        if (subjectEnum == EnumMessageProto.SubjectEnum.DEFAULT_SUBJECT || subjectEnum == EnumMessageProto.SubjectEnum.UNRECOGNIZED) {
            continue;
        }

        gradeInfoMap.put(subjectEnum.getNumber(), MessageProto.GradeInfo.newBuilder()
                .setScore(RandomUtils.nextDouble(0, 100))
                .setRank(RandomUtils.nextInt(1, 50))
                .build());
    }

    final List<MessageProto.ParentUser> parentUserList = Arrays.asList(
            MessageProto.ParentUser.newBuilder().setRelation("father").setTel(RandomStringUtils.randomNumeric(13)).build(),
            MessageProto.ParentUser.newBuilder().setRelation("mother").setTel(RandomStringUtils.randomNumeric(13)).build()
    );

    return MessageProto.User.newBuilder()
            .setName(RandomStringUtils.randomAlphabetic(5))
            .setAge(RandomUtils.nextInt(1, 80))
            .setSex(EnumMessageProto.SexEnum.forNumber(RandomUtils.nextInt(1, 2)))
            .putAllGrade(gradeInfoMap)
            .addAllParent(parentUserList)
            .build();
}
```

### 2.3 Protobuf Message

#### 2.3.1 序列化

```java
public static String toJson(Message message) {
    if (message == null) {
        return "";
    }

    try {
        return printer.print(message);
    } catch (InvalidProtocolBufferException e) {
        throw new IllegalStateException(e.getMessage(), e);
    }
}
```

#### 2.3.2 反序列化

```java
public static <T extends Message> T toBean(String json, Class<T> clazz) {
    if (StringUtils.isBlank(json)) {
        return null;
    }

    try {
        final Method method = clazz.getMethod("newBuilder");
        final Message.Builder builder = (Message.Builder) method.invoke(null);

        parser.merge(json, builder);

        return (T) builder.build();
    } catch (Exception e) {
        throw new RuntimeException("ProtobufUtils toMessage happen error, class: " + clazz + ", json: " + json, e);
    }
}
```

#### 2.3.3 测试用例

```java
@Test
public void testBean2Json() {
    // 序列化null
    final MessageProto.User nullUser = null;
    final String nullJson = ProtobufUtils.toJson(nullUser);
    Assert.assertEquals("", nullJson);

    // 反序列化null
    final MessageProto.User deserializeNull = ProtobufUtils.toBean(nullJson, MessageProto.User.class);
    Assert.assertNull(deserializeNull);

    // 序列化
    final MessageProto.User common = randomUser();
    final String commonJson = ProtobufUtils.toJson(common);
    System.out.println(commonJson);

    // 反序列化
    final MessageProto.User deserializeCommon = ProtobufUtils.toBean(commonJson, MessageProto.User.class);
    Assert.assertNotNull(deserializeCommon);
    Assert.assertEquals(common, deserializeCommon);
}
```

### 2.4 Protobuf Message List

#### 2.4.1 序列化

```java
public static String toJson(List<? extends MessageOrBuilder> messageList) {
    if (messageList == null) {
        return "";
    }
    if (messageList.isEmpty()) {
        return "[]";
    }

    try {
        StringBuilder builder = new StringBuilder(1024);
        builder.append("[");
        for (MessageOrBuilder message : messageList) {
            printer.appendTo(message, builder);
            builder.append(",");
        }
        return builder.deleteCharAt(builder.length() - 1).append("]").toString();
    } catch (Exception e) {
        throw new IllegalStateException(e.getMessage(), e);
    }
}
```

#### 2.4.2 反序列化

```java
public static <T extends Message> List<T> toBeanList(String json, Class<T> clazz) {
    if (StringUtils.isBlank(json)) {
        return Collections.emptyList();
    }

    final JSONArray jsonArray = JSON.parseArray(json);

    final List<T> resultList = new ArrayList<>(jsonArray.size());

    for (int i = 0; i < jsonArray.size(); i++) {
        resultList.add(toBean(jsonArray.getString(i), clazz));
    }

    return resultList;
}
```

#### 2.4.3 测试用例

```java
@Test
public void testList2Json() {
    // 序列化null集合
    List<MessageProto.User> nullList = null;
    final String nullListJson = ProtobufUtils.toJson(nullList);
    Assert.assertEquals("", nullListJson);

    // 反序列化null集合
    final List<MessageProto.User> deserializeNull = ProtobufUtils.toBeanList(nullListJson, MessageProto.User.class);
    Assert.assertEquals(0, deserializeNull.size());

    // 序列化空集合
    final String emptyListJson = ProtobufUtils.toJson(Collections.emptyList());
    Assert.assertEquals("[]", emptyListJson);

    // 反序列化空集合
    final List<MessageProto.User> deserializeEmpty = ProtobufUtils.toBeanList(emptyListJson, MessageProto.User.class);
    Assert.assertEquals(0, deserializeEmpty.size());

    // 序列化集合
    final List<MessageProto.User> commonList = IntStream.range(0, 3).boxed().map(e -> randomUser()).collect(Collectors.toList());
    final String commonListJson = ProtobufUtils.toJson(commonList);
    Assert.assertNotEquals("[]", commonListJson);
    System.out.println(commonListJson);

    // 反序列化
    final List<MessageProto.User> deserializeCommon = ProtobufUtils.toBeanList(commonListJson, MessageProto.User.class);
    Assert.assertEquals(commonList.size(), deserializeCommon.size());
    for (int i = 0; i < commonList.size(); i++) {
        Assert.assertEquals(commonList.get(i), deserializeCommon.get(i));
    }
}
```

### 2.5 Protobuf Message Map

这里我只实现了 Key 是普通类型，Value 是 Message 类型。如果有其他需求可以二次开发。

#### 2.5.1 序列化

```java
public static String toJson(Map<?, ? extends Message> messageMap) {
    if (messageMap == null) {
        return "";
    }
    if (messageMap.isEmpty()) {
        return "{}";
    }

    final StringBuilder sb = new StringBuilder();
    sb.append("{");
    messageMap.forEach((k, v) -> {
        sb.append("\"").append(JSON.toJSONString(k)).append("\":").append(toJson(v)).append(",");
    });
    sb.deleteCharAt(sb.length() - 1).append("}");
    return sb.toString();
}
```

#### 2.5.2 反序列化

```java
public static <K, V extends Message> Map<K, V> toBeanMap(String json, Class<K> keyClazz, Class<V> valueClazz) {
    if (StringUtils.isBlank(json)) {
        return Collections.emptyMap();
    }

    final JSONObject jsonObject = JSON.parseObject(json);

    final Map<K, V> map = Maps.newHashMapWithExpectedSize(jsonObject.size());
    for (String key : jsonObject.keySet()) {
        final K k = JSONObject.parseObject(key, keyClazz);
        final V v = toBean(jsonObject.getString(key), valueClazz);

        map.put(k, v);
    }

    return map;
}
```

#### 2.5.3 测试用例

```java
@Test
public void testMapToJson() {
    // 序列化nullMap
    final Map<Integer, MessageProto.User> nullMap = null;
    final String nullMapJson = ProtobufUtils.toJson(nullMap);
    Assert.assertEquals("", nullMapJson);
    // 反序列化nullMap
    final Map<Integer, MessageProto.User> deserializeNull = ProtobufUtils
            .toBeanMap(nullMapJson, Integer.class, MessageProto.User.class);
    Assert.assertEquals(0, deserializeNull.size());

    // 序列化空Map
    final String emptyMapJson = ProtobufUtils.toJson(Collections.emptyMap());
    Assert.assertEquals("{}", emptyMapJson);

    // 反序列化空Map
    final Map<Integer, MessageProto.User> deserializeEmpty = ProtobufUtils
            .toBeanMap(emptyMapJson, Integer.class, MessageProto.User.class);
    Assert.assertEquals(0, deserializeEmpty.size());

    // 序列化Map
    final Map<Integer, MessageProto.User> commonMap = new HashMap<Integer, MessageProto.User>() {{
        put(RandomUtils.nextInt(), randomUser());
        put(RandomUtils.nextInt(), randomUser());
        put(RandomUtils.nextInt(), randomUser());
    }};
    final String commonMapJson = ProtobufUtils.toJson(commonMap);
    Assert.assertNotEquals("[]", commonMapJson);
    System.out.println(commonMapJson);

    // 反序列化Map
    final Map<Integer, MessageProto.User> deserializeCommon = ProtobufUtils
            .toBeanMap(commonMapJson, Integer.class, MessageProto.User.class);
    Assert.assertEquals(commonMap.size(), deserializeCommon.size());
    commonMap.forEach((k, v) -> Assert.assertEquals(v, deserializeCommon.get(k)));
}
```

## 三、彩蛋

### 3.1 ProtobufCodec

最后给大家提供一个 fastjson 的转换器，免去手动序列化反序列化的烦恼。

```java
public class ProtobufCodec implements ObjectSerializer, ObjectDeserializer {
    @Override
    public <T> T deserialze(final DefaultJSONParser parser, final Type fieldType, final Object fieldName) {
        final String value = parser.parseObject().toJSONString();

        if (fieldType instanceof Class && Message.class.isAssignableFrom((Class<?>) fieldType)) {
            return (T) ProtobufUtils.toBean(value, (Class) fieldType);
        }

        if (fieldType instanceof ParameterizedType) {
            final ParameterizedType type = (ParameterizedType) fieldType;

            if (List.class.isAssignableFrom((Class<?>) type.getRawType())) {
                final Type argument = type.getActualTypeArguments()[0];
                if (Message.class.isAssignableFrom((Class<?>) argument)) {
                    return (T) ProtobufUtils.toBeanList(value, (Class) argument);
                }
            }

            if (Map.class.isAssignableFrom((Class<?>) type.getRawType())) {
                final Type[] arguments = type.getActualTypeArguments();
                if (arguments.length == 2) {
                    final Type keyType = arguments[0], valueType = arguments[1];
                    if (Message.class.isAssignableFrom((Class<?>) valueType)) {
                        return (T) ProtobufUtils.toBeanMap(value, (Class) keyType, (Class) valueType);
                    }
                }
            }
        }

        return null;
    }

    @Override
    public int getFastMatchToken() {
        return JSONToken.LITERAL_INT;
    }

    @Override
    public void write(final JSONSerializer serializer, final Object object, final Object fieldName,
                      final Type fieldType, final int features) throws IOException {
        final SerializeWriter out = serializer.out;

        if (object == null) {
            out.writeNull();
            return;
        }

        if (fieldType instanceof Class && Message.class.isAssignableFrom((Class<?>) fieldType)) {
            final Message value = (Message) object;
            out.writeString(ProtobufUtils.toJson(value));
        } else if (fieldType instanceof ParameterizedType) {
            final ParameterizedType type = (ParameterizedType) fieldType;

            if (List.class.isAssignableFrom((Class<?>) type.getRawType())) {
                final Type argument = type.getActualTypeArguments()[0];
                if (Message.class.isAssignableFrom((Class<?>) argument)) {
                    final List<Message> messageList = (List<Message>) object;
                    out.writeString(ProtobufUtils.toJson(messageList));
                } else {
                    out.writeString("[]");
                }
            } else if (Map.class.isAssignableFrom((Class<?>) type.getRawType())) {
                final Type[] arguments = type.getActualTypeArguments();
                if (arguments.length == 2) {
                    final Type keyType = arguments[0], valueType = arguments[1];

                    if (Message.class.isAssignableFrom((Class<?>) valueType)) {
                        Map<?, Message> messageMap = (Map<?, Message>) object;

                        final String toStr = ProtobufUtils.toJson(messageMap);

                        out.write(toStr, 0, toStr.length());
                    }
                } else {
                    out.writeString("{}");
                }
            }
        }
    }
}
```

### 3.2 测试用例

首先写一个对象来测试下：

```java
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class ProtoBean {
    private long id;

    @JSONField(serializeUsing = ProtobufCodec.class, deserializeUsing = ProtobufCodec.class)
    private MessageProto.User user;

    @JSONField(serializeUsing = ProtobufCodec.class, deserializeUsing = ProtobufCodec.class)
    private List<MessageProto.User> userList;

    @JSONField(serializeUsing = ProtobufCodec.class, deserializeUsing = ProtobufCodec.class)
    private Map<Integer, MessageProto.User> userMap;

    private Date createDate;
}
```

最后附上测试用例：

```java
@Test
public void testCodec() {
    final Map<Integer, MessageProto.User> userMap = new HashMap<Integer, MessageProto.User>() {{
        put(RandomUtils.nextInt(), randomUser());
        put(RandomUtils.nextInt(), randomUser());
        put(RandomUtils.nextInt(), randomUser());
        put(RandomUtils.nextInt(), randomUser());
    }};

    final ProtoBean protoBean = ProtoBean.builder()
            .id(RandomUtils.nextLong())
            .user(randomUser())
            .userList(IntStream.range(0, 3).boxed().map(e -> randomUser()).collect(Collectors.toList()))
            .userMap(userMap)
            .createDate(new Date(RandomUtils.nextLong()))
            .build();

    final String json = JSON.toJSONString(protoBean);

    final ProtoBean protoBean1 = JSON.parseObject(json, ProtoBean.class);

    Assert.assertNotNull(protoBean1);
    Assert.assertEquals(protoBean.getId(), protoBean1.getId());
    Assert.assertEquals(protoBean.getUser(), protoBean1.getUser());
    Assert.assertEquals(protoBean.getUserList().size(), protoBean1.getUserList().size());
    for (int i = 0; i < protoBean.getUserList().size(); i++) {
        Assert.assertEquals(protoBean.getUserList().get(i), protoBean1.getUserList().get(i));
    }
    Assert.assertEquals(protoBean.getUserMap().size(), protoBean1.getUserMap().size());
    protoBean.getUserMap().forEach((k, v) -> Assert.assertEquals(v, protoBean1.getUserMap().get(k)));
    Assert.assertEquals(protoBean.getCreateDate(), protoBean1.getCreateDate());
}
```

