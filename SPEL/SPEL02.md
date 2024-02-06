## 构造数组

```java
/**
 * 数组生成
 */
@Test
public void test5(){
    int[] numbers1 = (int[]) parser.parseExpression("new int[4]").getValue();
	int[][] numbers2 = (int[][]) parser.parseExpression("new int[4][5]").getValue();
    
    // 一维数组可以初始化
    int[] numbers3 = (int[]) parser.parseExpression("new int[]{1,2,3}").getValue();
    Assert.assertTrue(numbers3.length==3);
    // 二维数组初始化失败
    int[][] numbers4 = (int[][]) parser.parseExpression("new int[][]{{1,2,3},{4,5,6}}").getValue();
}
```

如果需要构造一个空数组，则可以直接new一个空的数组，多维数组也是支持的，但是多维数组只支持一个空的数组，对于需要初始化指定数组元素的定时暂时在SPEL中是不支持的

<a name="E9aba"></a>

## 构造List

在SPEL中可以使用{e1,e2,e3}的形式来构造一个List

```java
@Test
public void test09() {
    ExpressionParser parser = new SpelExpressionParser();
    List<Integer> intList = (List<Integer>)parser.parseExpression("{1,2,3,4,5,6}").getValue();
    int index = 0;
    for (Integer i : intList) {
    	Assert.assertTrue(i == ++index);
    }
}
```

如果希望构造的List的元素还是一个List，则可以将构造的List的元素定义为{e1,e2,e3}这样的形式，如：{{1,2},{3,4,5},{6,7,8,9}}

```java
@Test
public void test09_1() {
    ExpressionParser parser = new SpelExpressionParser();
    List<List<Integer>> list = (List<List<Integer>>)parser.parseExpression("{{1,2},{3,4,5},{6,7,8,9}}").getValue();
    int index = 0;
    for (List<Integer> intList : list) {
        for (Integer i : intList) {
        	Assert.assertTrue(i == ++index);
        }
    }
}
```

<a name="JBOH2"></a>

## 构造Map

在SPEL中如果需要构造一个Map则可以使用{key1:value1,key2:value2}的形式进行定义

```java
@Test
public void test10() {
	 ExpressionParser parser = new SpelExpressionParser();
	 Map<String, Long> map = (Map<String, Long>)parser.parseExpression("{'key1':1L,'key2':2L}").getValue();
	 Assert.assertTrue(map.get("key1").equals(1L));
	 Assert.assertTrue(map.get("key2").equals(2L));
}
```

如果需要构造一个空的Map，则只需指定对应的表达式为{:}即可

```java
@Test
public void test10_1() {
	 ExpressionParser parser = new SpelExpressionParser();
	 Map<String, Long> map = (Map<String, Long>)parser.parseExpression("{:}").getValue();
	 Assert.assertTrue(map.isEmpty());
}
```

<a name="tiBYV"></a>

## 元素访问

1.在SPEL中可以通过索引的形式访问List或Array的某一个元素，对应的索引是从0开始的，以list[index]的形式出现

```java
@Test
public void test08_1() {
    Object user = new Object() {
    	public List<String> getInterests() {
        	List<String> interests = Arrays.asList(new String[] {"BasketBall", "FootBall"});
        	return interests;
        }
    };
    ExpressionParser parser = new SpelExpressionParser();
    Assert.assertTrue(parser.parseExpression("interests[0]").getValue(user, String.class).equals("BasketBall"));
    Assert.assertTrue(parser.parseExpression("interests[1]").getValue(user, String.class).equals("FootBall"));
}
 
@Test
public void test08_2() {
    Object user = new Object() {
        public String[] getInterests() {
        	return new String[] {"BasketBall", "FootBall"};
        }
    };
    ExpressionParser parser = new SpelExpressionParser();
    Assert.assertTrue(parser.parseExpression("interests[0]").getValue(user, String.class).equals("BasketBall"));
    Assert.assertTrue(parser.parseExpression("interests[1]").getValue(user, String.class).equals("FootBall"));
}
```

2.对于Map而言，则是通过类似于map[key]的形式访问对应的元素的

```java
@Test
public void test08_3() {
    Object user = new Object() {
        public Map<String, String> getInterests() {
            Map<String, String> interests = new HashMap<String, String>();
            interests.put("key1", "BasketBall");
            interests.put("key2", "FootBall");
            return interests;
        }
    };
    ExpressionParser parser = new SpelExpressionParser();
    Assert.assertTrue(parser.parseExpression("interests['key1']").getValue(user, String.class).equals("BasketBall"));
    Assert.assertTrue(parser.parseExpression("interests['key2']").getValue(user, String.class).equals("FootBall"));
}
```

<a name="wOKk2"></a>

## 集合选择

SPEL允许将集合中的某些元素选出组成一个新的集合进行返回，语法是collection.[符号\][selectionExpression]，selectionExpression中直接使用的属性、方法等都是针对于集合中的元素来的

1. collection.?[selectionExpression]：从集合按条件筛选生成新集合
2. collection.^[selectionExpression]：从集合按条件筛选取第一个元素
3. collection.$[selectionExpression]：从集合按条件筛选取最后一个元素

```java
@Test
public void test12_1() {
    Object user = new Object() {
        public List<String> getInterests() {
            List<String> interests = new ArrayList<String>();
            interests.add("BasketBall");
            interests.add("FootBall");
            interests.add("Movie");
            return interests;
        }
    };
    ExpressionParser parser = new SpelExpressionParser();
    List<String> interests = (List<String>)parser.parseExpression("interests.?[endsWith('Ball')]").getValue(user);
    Assert.assertTrue(interests.size() == 2);
    Assert.assertTrue(interests.get(0).equals("BasketBall"));
    Assert.assertTrue(interests.get(1).equals("FootBall"));
}
```

```java
@Test
public void test12_2() {
    Object user = new Object() {
        public Map<String, String> getInterests() {
            Map<String, String> interests = new HashMap<String, String>();
            interests.put("key1", "BasketBall");
            interests.put("key2", "FootBall");
            interests.put("key3", "Movie");
            return interests;
        }
    };
    ExpressionParser parser = new SpelExpressionParser();
    Map<String, String> interests = (Map<String, String>)parser.parseExpression("interests.?[value.endsWith('Ball')]").getValue(user);
    Assert.assertTrue(interests.size() == 2);
    Assert.assertTrue(interests.get("key1").equals("BasketBall"));
    Assert.assertTrue(interests.get("key2").equals("FootBall"));
}
```

<a name="IBHct"></a>

## 集合投影

将集合中每个元素的某部分内容组成一个新的集合进行返回，语法是collection.![projectionExpression]，projectionExpression中直接使用的属性和方法都是针对于collection中的每个元素而言的

```java
@Test
public void test13_1() {
    Object user = new Object() {
        public List<String> getInterests() {
            List<String> interests = new ArrayList<String>();
            interests.add("BasketBall");
            interests.add("FootBall");
            interests.add("Movie");
            return interests;
        }
    };
    ExpressionParser parser = new SpelExpressionParser();
    List<Boolean> interests = (List<Boolean>)parser.parseExpression("interests.![endsWith('Ball')]").getValue(user);
    Assert.assertTrue(interests.size() == 3);
    Assert.assertTrue(interests.get(0).equals(true));
    Assert.assertTrue(interests.get(1).equals(true));
    Assert.assertTrue(interests.get(2).equals(false));
}
```

```java
@Test
public void test13_2() {
    Object user = new Object() {
        public Map<String, String> getInterests() {
            Map<String, String> interests = new HashMap<String, String>();
            interests.put("key1", "BasketBall");
            interests.put("key2", "FootBall");
            interests.put("key3", "Movie");
            return interests;
        }
    };
    ExpressionParser parser = new SpelExpressionParser();
    List<String> interests = (List<String>)parser.parseExpression("interests.![value]").getValue(user);
    Assert.assertTrue(interests.size() == 3);
    for (String interest : interests) {
    	Assert.assertTrue(interest.equals("BasketBall") || interest.equals("FootBall") || interest.equals("Movie"));
    }
}
```

<a name="YG2zy"></a>

## 自动扩充

在构建SpelExpressionParser时我们可以将其传递一个SpelParserConfiguration对象以对SpelExpressionParser进行配置

1. 参数1：在遇到List或Array为null时是否自动new一个对应的实例
2. 参数2：在List或Array中对应索引超过了当前索引的最大值时是否自动进行扩充

```java
class User {
    public List<String> interests;
}

@Test
public void test() {
    User user = new User();
    SpelParserConfiguration parserConfig = new SpelParserConfiguration(true, true);
    ExpressionParser parser = new SpelExpressionParser(parserConfig);
    //第一次为null
    Assert.assertNull(parser.parseExpression("interests").getValue(user));
    //自动new一个List的实例，对应ArrayList,并自动new String()添加6次。
    Assert.assertTrue(parser.parseExpression("interests[5]").getValue(user).equals(""));
    //size为6
    Assert.assertTrue(parser.parseExpression("interests.size()").getValue(user).equals(6));
}
```

<a name="LKM4z"></a>