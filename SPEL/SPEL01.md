## 算术运算

```java
@Test
public void test01() {
	 // 定义解析器
	 ExpressionParser parser = new SpelExpressionParser();
     // 使用解析器解析表达式
     // 获取解析结果
	 Assert.assertTrue(parser.parseExpression("(1+2)*5 + 8-6/2").getValue().equals(20));//加减乘除
	 Assert.assertTrue(parser.parseExpression("8%3").getValue().equals(2));//求余
	 Assert.assertTrue(parser.parseExpression("2.0e3").getValue().equals(2000.0));//指数
	 Assert.assertTrue(parser.parseExpression("2^3").getValue().equals(8));//指数
}
```

<a name="obV5u"></a>

## 逻辑运算

```java
@Test
public void test02() {
	 ExpressionParser parser = new SpelExpressionParser();
	 Assert.assertTrue(parser.parseExpression("true and true").getValue(Boolean.class));//与
	 Assert.assertTrue(parser.parseExpression("true or false").getValue(Boolean.class));//与
	 Assert.assertTrue(parser.parseExpression("!false").getValue(Boolean.class));//非
}
```

<a name="SoRHg"></a>

## 比较运算

```java
@Test
public void test03() {
	 ExpressionParser parser = new SpelExpressionParser();
	 Assert.assertTrue(parser.parseExpression("5>3").getValue(Boolean.class));
	 Assert.assertTrue(parser.parseExpression("5<=8").getValue(Boolean.class));
	 Assert.assertTrue(parser.parseExpression("5==5").getValue(Boolean.class));
	 Assert.assertTrue(parser.parseExpression("5!=6").getValue(Boolean.class));
}
```

<a name="WcPzH"></a>

***使用字符代替符号***

```java
@Test
public void test19() {
    ExpressionParser parser = new SpelExpressionParser();
    Assert.assertTrue(parser.parseExpression("1 lt 2").getValue(boolean.class));//1<2
    Assert.assertTrue(parser.parseExpression("1 le 2").getValue(boolean.class));//1<=2
    Assert.assertTrue(parser.parseExpression("2 gt 1").getValue(boolean.class));//2>1
    Assert.assertTrue(parser.parseExpression("2 ge 1").getValue(boolean.class));//2>=1
    Assert.assertTrue(parser.parseExpression("1 ne 2").getValue(boolean.class));//1!=2
    Assert.assertTrue(parser.parseExpression("not false").getValue(boolean.class));//!false
}
```

<a name="zuZWM"></a>

## 正则表达式

```java
@Test
public void test20 () {
    ExpressionParser parser = new SpelExpressionParser();
    Assert.assertTrue(parser.parseExpression("123 matches '\\d{3}'").getValue(Boolean.class));//正则匹配三位数字
}
```

<a name="qWBbP"></a>

## instanceof类型判断

```java
@Test
public void test21 () {
    ExpressionParser parser = new SpelExpressionParser();
    //检测字符串是否是String的实例。
    Assert.assertTrue(parser.parseExpression("'123' instanceof T(String)").getValue(Boolean.class));
}
```

<a name="YvNhh"></a>

## 三目运算

```java
@Test
public void test22 () {
    ExpressionParser parser = new SpelExpressionParser();
    Assert.assertTrue(parser.parseExpression("1>2 ? 1 : 2").getValue(int.class) == 2);//1跟2之间的较大者为2。
    Assert.assertTrue(parser.parseExpression("1<2 ? 2 : 1").getValue(int.class) == 2);//1跟2之间的较大者为2。
}
```

<a name="MRjDH"></a>

## 字符串

操作字符串时需要通过单引号'来进行包裹，而当我们的字符串中包含单引号时，那么对应的单引号需要使用一个单引号进行转义，即连续两个单引号

```java
@Test
public void test04() {
	 ExpressionParser parser = new SpelExpressionParser();
	 Assert.assertTrue(parser.parseExpression("'abc'").getValue().equals("abc"));
	 Assert.assertTrue(parser.parseExpression("'''abc'").getValue().equals("'abc"));
}
```

<a name="Rr2t0"></a>

## 方法调用

<a name="svFMB"></a>

### 字符串与对象

```java
/**
 * 方法调用
 */
@Test
public void test6(){
    ExpressionParser parser = new SpelExpressionParser();
    //直接访问String的length()方法。
    Assert.assertTrue(parser.parseExpression("'abc'.length()").getValue().equals(3));
    String bc = parser.parseExpression("'abc'.substring(1, 3)").getValue(String.class);
    System.out.println(bc);// bc

    // 传递参数，调用Society的isMember方法
    Society societyContext = new Society();
    boolean isMember = parser.parseExpression("isMember('Mihajlo Pupin')").getValue(societyContext, Boolean.class);
    System.out.println(isMember);// false
}

public Boolean isNumber(Object object) {
    return object instanceof Integer;
}
```

<a name="ST0Vb"></a>

### 构造方法new

使用 new 运算符调用构造函数。对除原始类型（int、float 等）和 String 之外的所有类型使用完全限定的类名

```java
/**
 * new 调用构造方法
 */
@Test
public void test12(){
    Inventor value = parser.parseExpression("new com.crab.spring.ioc.demo20.Inventor('ooo','xxx')").getValue(Inventor.class);
    System.out.println(value.getName() + " " + value.getNationality()); // ooo xxx

    String value1 = parser.parseExpression("new String('xxxxoo')").getValue(String.class);
    System.out.println(value1); // xxxxoo

    Expression exp = parser.parseExpression("new String('hello world').toUpperCase()");
    String message = exp.getValue(String.class);
}
```

<a name="RzHYl"></a>

### 类类型T

类中的静态变量、静态方法属于Class， 可以通过T(xxx).xxx调用

```java
@Test
public void test11(){
    // 1 获取类的Class java.lang包下的类可以不指定全路径
    Class value = parser.parseExpression("T(String)").getValue(Class.class);
    System.out.println(value);

    // 2 获取类的Class 非java.lang包下的类必须指定全路径
    Class dateValue = parser.parseExpression("T(java.util.Date)").getValue(Class.class);
    System.out.println(dateValue);

    // 3 类中的静态变量 静态方法属于Class 通过T(xxx)调用
    boolean trueValue = parser.parseExpression(
                    "T(java.math.RoundingMode).CEILING < T(java.math.RoundingMode).FLOOR")
            .getValue(Boolean.class); // true
    System.out.println(trueValue);
    Long longValue = parser.parseExpression("T(Long).parseLong('9999')").getValue(Long.class);
    System.out.println(longValue);// 9999
}
```

<a name="YDuJD"></a>

## 表达式模板

表达式模板允许将文字文本与一个或多个评估块混合。每个评估块都由前缀和后缀字符分隔，默认是#{}。支持实现接口ParserContext自定义前后缀。调用parseExpression()时指定 ParserContext参数如：new TemplateParserContext()，#{}包起来的表达式会被计算

```java
@Test
public void test23 () {
    //the year is 2014
    String expressionStr = "the year is #{T(java.util.Calendar).getInstance().get(T(java.util.Calendar).YEAR)}";
    ExpressionParser parser = new SpelExpressionParser();
    Expression expression = parser.parseExpression(expressionStr, new TemplateParserContext());
    System.out.println(expression.getValue());
}
```

<a name="hB7Kg"></a>

## elvis设置默认值

SPEL表达式中支持a?:b这样的语法来设置默认值，表达如果a不为null时其结果为a，否则就为b

```java
@Test
public void test () {
    ExpressionParser parser = new SpelExpressionParser();
    Assert.assertTrue(parser.parseExpression("#abc?:123").getValue().equals(123));//变量abc不存在
    Assert.assertTrue(parser.parseExpression("1?:123").getValue().equals(1));//数字1不为null
}
```

<a name="NSokt"></a>

## ?.嵌套属性安全访问

多级属性访问，如国家城市城镇nation.city.town三级访问，如果中间的 city是null则会抛出 NullPointerException 异常。为了避免这种情况的异常，SpEL借鉴了Groovy的语法?.，如果中间属性为null不会抛出异常而是返回null

```java
/**
 * 多级属性安全访问
 */
@Test
public void test18(){
    Inventor inventor = new Inventor("xx", "oo");
    inventor.setPlaceOfBirth(new PlaceOfBirth("北京", "中国"));

    // 正常访问
    String city = parser.parseExpression("PlaceOfBirth?.city").getValue(context, inventor, String.class);
    System.out.println(city); // 北京

    // placeOfBirth为null
    inventor.setPlaceOfBirth(null);
    String city1 = parser.parseExpression("PlaceOfBirth?.city").getValue(context, inventor, String.class);
    System.out.println(city1); // null

    // 非安全访问 异常
    String city3 = parser.parseExpression("PlaceOfBirth.city").getValue(context, inventor, String.class);
    System.out.println(city3); // 抛出异常
}
```

<a name="jt22J"></a>