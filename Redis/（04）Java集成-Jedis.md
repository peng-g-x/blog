## Jedis

```xml
<dependency>
    <groupId>redis.clients</groupId>
    <artifactId>jedis</artifactId>
    <version>3.6.0</version>
</dependency>
```

```java
//实体类序列化，需要实现Serializable接口
@Data
@AllArgsConstructor
@NoArgsConstructor
public class User implements Serializable {
    private String name;
    private int age;
}
```

```java
@Test
void contextLoads() {
    Jedis jedis = new Jedis("127.0.0.1",6379);
    System.out.println(jedis.ping());
    jedis.set("name","123");
    System.out.println(jedis.get("name"));
    jedis.close();
}

@Test
void test01() {
    Jedis jedis = new Jedis("127.0.0.1", 6379);
    JSONObject jsonObject = new JSONObject();
    jsonObject.put("name","dasda");
    jsonObject.put("pass","fds");

    //开启事务
    Transaction multi = jedis.multi();
    String s = jsonObject.toJSONString();

    try {
        multi.set("user",s);
        multi.set("user2",s);
        int i=1/0;
        multi.exec();
    }catch (Exception e){
        multi.discard();
        e.printStackTrace();
    }finally {
        System.out.println(jedis.get("user"));
        System.out.println(jedis.get("user2"));
        jedis.close();
    }
}
```