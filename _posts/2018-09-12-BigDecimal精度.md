---
title: BigDecimal精度
date: 2018-09-12 16:41:50
tags: java
categories: java
---


### 1. 构造函数的坑
```java
public class BigDecimalTest {
    public static void main(String[] args){
      double number = 6302079.05;
      BigDecimal bigDecimalFromDouble = new BigDecimal(number);
      LOGGER.info("bigDecimalFromDouble={}", bigDecimalFromDouble);
      BigDecimal bigDecimalFromString = new BigDecimal(String.valueOf(number)) ;
      LOGGER.info("bigDecimalFromString={}", bigDecimalFromString);
    }
}
```
输出结果：
```
bigDecimalFromDouble=6302079.049999999813735485076904296875
bigDecimalFromString=6302079.05
```
可以看到，直接使用double作为参数构造BigDecimal时，会丢失精度。
所以推荐的是先将double转化为字符串，然后赋值给构造函数。

<!--  more -->

### 2. 数值计算丢失精度
```java
public void testDecimalCompute() {
    BigDecimal bigDecimalFromString = new BigDecimal("6302079.05") ;
    LOGGER.info("bigDecimalFromString={}", bigDecimalFromString);
    LOGGER.info("bigDecimalFromString floatValue={}", bigDecimalFromString.floatValue());
    LOGGER.info("bigDecimalFromString floatValue*100={}", (int) (bigDecimalFromString.floatValue() * 100));
    LOGGER.info("bigDecimalFromString doubleValue={}", bigDecimalFromString.doubleValue());
    LOGGER.info("bigDecimalFromString doubleValue*100={}", (int) (bigDecimalFromString.doubleValue() * 100));
    LOGGER.info("bigDecimalFromString multiply(100)={}", bigDecimalFromString.multiply(new BigDecimal("100")).intValue());
}
```
输出结果：
```
bigDecimalFromString=6302079.05
bigDecimalFromString floatValue=6302079.0
bigDecimalFromString floatValue*100=630207872
bigDecimalFromString doubleValue=6302079.05
bigDecimalFromString doubleValue*100=630207905
bigDecimalFromString multiply(100)=630207905
```
尝试调用`floatValue()`方法获取浮点值时，内部其实会将double转化为float，导致精度丢失。
所以要求使用BigDecimal提供的计算方法做四则运算，最后将结果返回指定数值类型。

### 3. 四舍五入

#### 3.1 ROUND_UP

> Rounding mode to round away from zero. Always increments the digit prior to a nonzero discarded fraction.

不关心正负的无脑向上取整，远离0。

```java
public void testBigDecimalRound() {
    BigDecimal bigDecimalPositive = new BigDecimal("6302079.05") ;
    BigDecimal bigDecimalNegative = new BigDecimal("-6302079.05") ;
    LOGGER.info("bigDecimalPositive={}", bigDecimalPositive);
    LOGGER.info("bigDecimalNegative={}", bigDecimalNegative);

    LOGGER.info("bigDecimalPositive ROUND_UP={}", bigDecimalPositive.setScale(1, BigDecimal.ROUND_UP));
    LOGGER.info("bigDecimalNegative ROUND_UP={}", bigDecimalNegative.setScale(1, BigDecimal.ROUND_UP));
}
```
输出结果：
```
bigDecimalPositive ROUND_UP=6302079.1
bigDecimalNegative ROUND_UP=-6302079.1
```

#### 3.2 ROUND_DOWN

> Rounding mode to round towards zero. Never increments the digit prior to a discarded fraction (i.e., truncates).

不关心正负的无脑向下取整，靠近0。

```java
public void testBigDecimalRound() {
    BigDecimal bigDecimalPositive = new BigDecimal("6302079.05") ;
    BigDecimal bigDecimalNegative = new BigDecimal("-6302079.05") ;
    LOGGER.info("bigDecimalPositive={}", bigDecimalPositive);
    LOGGER.info("bigDecimalNegative={}", bigDecimalNegative);

    LOGGER.info("bigDecimalPositive ROUND_DOWN={}", bigDecimalPositive.setScale(1, BigDecimal.ROUND_DOWN));
    LOGGER.info("bigDecimalNegative ROUND_DOWN={}", bigDecimalNegative.setScale(1, BigDecimal.ROUND_DOWN));
}
```
输出结果：
```
bigDecimalPositive ROUND_DOWN=6302079.0
bigDecimalNegative ROUND_DOWN=-6302079.0
```

#### 3.3 ROUND_CEILING

> Rounding mode to round towards positive infinity. If the BigDecimal is positive, behaves as for ROUND_UP; if negative, behaves as for ROUND_DOWN.

向正无穷取整，如果是正值，那么表现跟ROUND_UP一样，如果是负值，那么表现跟ROUND_DOWN一样。


```java
public void testBigDecimalRound() {
    BigDecimal bigDecimalPositive = new BigDecimal("6302079.05") ;
    BigDecimal bigDecimalNegative = new BigDecimal("-6302079.05") ;
    LOGGER.info("bigDecimalPositive={}", bigDecimalPositive);
    LOGGER.info("bigDecimalNegative={}", bigDecimalNegative);

    LOGGER.info("bigDecimalPositive ROUND_CEILING={}", bigDecimalPositive.setScale(1, BigDecimal.ROUND_CEILING));
    LOGGER.info("bigDecimalNegative ROUND_CEILING={}", bigDecimalNegative.setScale(1, BigDecimal.ROUND_CEILING));
}
```
输出结果：
```
bigDecimalPositive ROUND_CEILING=6302079.1
bigDecimalNegative ROUND_CEILING=-6302079.0
```

#### 3.4 ROUND_FLOOR

> Rounding mode to round towards negative infinity. If the BigDecimal is positive, behave as for ROUND_DOWN; if negative, behave as for ROUND_UP.

向负无穷取整，如果是正值，那么表现跟ROUND_DOWN一样，如果是负值，那么表现跟ROUND_UP一样。

```java
public void testBigDecimalRound() {
    BigDecimal bigDecimalPositive = new BigDecimal("6302079.05") ;
    BigDecimal bigDecimalNegative = new BigDecimal("-6302079.05") ;
    LOGGER.info("bigDecimalPositive={}", bigDecimalPositive);
    LOGGER.info("bigDecimalNegative={}", bigDecimalNegative);

    LOGGER.info("bigDecimalPositive ROUND_FLOOR={}", bigDecimalPositive.setScale(1, BigDecimal.ROUND_FLOOR));
    LOGGER.info("bigDecimalNegative ROUND_FLOOR={}", bigDecimalNegative.setScale(1, BigDecimal.ROUND_FLOOR));
}
```
输出结果：
```
bigDecimalPositive ROUND_FLOOR=6302079.0
bigDecimalNegative ROUND_FLOOR=-6302079.1
```

#### 3.5 ROUND_HALF_UP

> Rounding mode to round towards "nearest neighbor" unless both neighbors are equidistant, in which case round up. Behaves as for ROUND_UP if the discarded fraction is ≥ 0.5; otherwise, behaves as for ROUND_DOWN.

不关心正负的无脑四舍五入，如果是5，那么进位。

```java
public void testBigDecimalRound() {
    BigDecimal bigDecimalPositive = new BigDecimal("6302079.05") ;
    BigDecimal bigDecimalNegative = new BigDecimal("-6302079.05") ;
    LOGGER.info("bigDecimalPositive={}", bigDecimalPositive);
    LOGGER.info("bigDecimalNegative={}", bigDecimalNegative);

    LOGGER.info("bigDecimalPositive ROUND_HALF_UP={}", bigDecimalPositive.setScale(1, BigDecimal.ROUND_HALF_UP));
    LOGGER.info("bigDecimalNegative ROUND_HALF_UP={}", bigDecimalNegative.setScale(1, BigDecimal.ROUND_HALF_UP));
}
```
输出结果：
```
bigDecimalPositive ROUND_HALF_UP=6302079.1
bigDecimalNegative ROUND_HALF_UP=-6302079.1
```

#### 3.6 ROUND_HALF_DOWN

> Rounding mode to round towards "nearest neighbor" unless both neighbors are equidistant, in which case round down. Behaves as for ROUND_UP if the discarded fraction is > 0.5; otherwise, behaves as for ROUND_DOWN.

不关心正负的无脑四舍五入，如果是5，那么舍掉。

```java
public void testBigDecimalRound() {
    BigDecimal bigDecimalPositive = new BigDecimal("6302079.05") ;
    BigDecimal bigDecimalNegative = new BigDecimal("-6302079.05") ;
    LOGGER.info("bigDecimalPositive={}", bigDecimalPositive);
    LOGGER.info("bigDecimalNegative={}", bigDecimalNegative);

    LOGGER.info("bigDecimalPositive ROUND_HALF_DOWN={}", bigDecimalPositive.setScale(1, BigDecimal.ROUND_HALF_DOWN));
    LOGGER.info("bigDecimalNegative ROUND_HALF_DOWN={}", bigDecimalNegative.setScale(1, BigDecimal.ROUND_HALF_DOWN));
}
```
输出结果：
```
bigDecimalPositive ROUND_HALF_DOWN=6302079.0
bigDecimalNegative ROUND_HALF_DOWN=-6302079.0
```

#### 3.7 ROUND_HALF_EVEN

> Rounding mode to round towards the "nearest neighbor" unless both neighbors are equidistant, in which case, round towards the even neighbor. Behaves as for ROUND_HALF_UP if the digit to the left of the discarded fraction is odd; behaves as for ROUND_HALF_DOWN if it's even.

不关心正负的无脑四舍五入，如果是5，那么看左边数字奇偶性，如果左边数字是奇数，那么进位，如果是偶数，那么舍掉。

```java
public void testBigDecimalRound() {
    BigDecimal bigDecimalPositive = new BigDecimal("6302079.05") ;
    BigDecimal bigDecimalNegative = new BigDecimal("-6302079.05") ;
    LOGGER.info("bigDecimalPositive={}", bigDecimalPositive);
    LOGGER.info("bigDecimalNegative={}", bigDecimalNegative);

    // 最后一位的前一位是偶数，那么ROUND_HALF_EVEN等同与ROUND_HALF_DOWN
    LOGGER.info("bigDecimalPositive ROUND_HALF_DOWN={}", bigDecimalPositive.setScale(1, BigDecimal.ROUND_HALF_EVEN));
    LOGGER.info("bigDecimalNegative ROUND_HALF_DOWN={}", bigDecimalNegative.setScale(1, BigDecimal.ROUND_HALF_EVEN));

    // 将最后一位的前一位变成奇数，那么ROUND_HALF_EVEN等同与ROUND_HALF_UP
    bigDecimalPositive = new BigDecimal("6302079.15");
    bigDecimalNegative = new BigDecimal("-6302079.15");
    LOGGER.info("bigDecimalPositive ROUND_HALF_DOWN={}", bigDecimalPositive.setScale(1, BigDecimal.ROUND_HALF_EVEN));
    LOGGER.info("bigDecimalNegative ROUND_HALF_DOWN={}", bigDecimalNegative.setScale(1, BigDecimal.ROUND_HALF_EVEN));
}
```
输出结果：
```
bigDecimalPositive ROUND_HALF_DOWN=6302079.0
bigDecimalNegative ROUND_HALF_DOWN=-6302079.0
bigDecimalPositive ROUND_HALF_DOWN=6302079.2
bigDecimalNegative ROUND_HALF_DOWN=-6302079.2
```
