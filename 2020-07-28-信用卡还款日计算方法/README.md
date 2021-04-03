信用卡涉及三个日期<sup>[1][2]</sup>：记账日期，账单日期，还款日期。为了简化处理，忽略交易日期<sup>[2]</sup>和记账日期的区别，只使用记账日期。对于这三个日期的翻译参考了中国银行对账单<sup>[3]</sup>：

* 记账日期 - post date
* 账单日期 - statement date
* 还款日期 - due date

在申请信用卡时会确定两个日期：账单日和还款日<sup>[1]</sup>。一般有类似这样的描述：账单日为每月多少日，还款日为账单日后多少天。根据这个描述可以推断出具体的还款日（每月按 30 天计算）。

举个例子：账单日为每月 3 日，还款日为账单日后 10 天，那么还款日为每月 13 日。

记账日期 | 账单日期 | 还款日期 | 描述
---|---|---|---
2020-07-02 | 2020-07-03 | 2020-07-13 | 2020-06-03 00:00:00至2020-07-02 23:59:59的账单
2020-07-03 | 2020-08-03 | 2020-08-13 | 2020-07-03 00:00:00至2020-08-02 23:59:59的账单
2020-07-04 | 2020-08-03 | 2020-08-13 | 2020-07-03 00:00:00至2020-08-02 23:59:59的账单
2020-08-02 | 2020-08-03 | 2020-08-13 | 2020-07-03 00:00:00至2020-08-02 23:59:59的账单
2020-08-03 | 2020-09-03 | 2020-09-13 | 2020-08-03 00:00:00至2020-09-02 23:59:59的账单
2020-08-04 | 2020-09-03 | 2020-09-13 | 2020-08-03 00:00:00至2020-09-02 23:59:59的账单

再举个例子：账单日为每月 26 日，还款日为账单日后 10 天，那么还款日为次月 6 日（每月按 30 天计算）。

记账日期 | 账单日期 | 还款日期 | 描述
---|---|---|---
2020-07-02 | 2020-07-26 | 2020-08-06 | 2020-06-26 00:00:00至2020-07-25 23:59:59的账单
2020-07-03 | 2020-07-26 | 2020-08-06 | 2020-06-26 00:00:00至2020-07-25 23:59:59的账单
2020-07-04 | 2020-07-26 | 2020-08-06 | 2020-06-26 00:00:00至2020-07-25 23:59:59的账单
2020-08-02 | 2020-08-26 | 2020-09-06 | 2020-07-26 00:00:00至2020-08-25 23:59:59的账单
2020-08-03 | 2020-08-26 | 2020-09-06 | 2020-07-26 00:00:00至2020-08-25 23:59:59的账单
2020-08-04 | 2020-08-26 | 2020-09-06 | 2020-07-26 00:00:00至2020-08-25 23:59:59的账单

关于记账日、账单日和还款日的一时没有想到更合适的翻译，因此选取了 day 这个单词，用来指代某一天。

* 记账日 - post day
* 账单日 - statement day
* 还款日 - due day

为了消除大小月份和润年导致每个月份天数的差异，一般账单日和还款日不会选择 29、30、31 这个三个日期。

在前面描述的基础上，给定任意记账日期可以写出计算该笔消费还款日期的程序。

```java
public Date calculateDueDate(Date postDate, int statementDay, int dueDay) {
    int postDay = getDayOfMonth(postDate);
    
    Date date1 = DateUtils.setDays(postDate, 1);
    Date date2 = DateUtils.addMonths(date1, postDay < statementDay ? 0 : 1);
    Date date3 = DateUtils.addDays(date2, dueDay - 1);
    
    return date3;
}
```

其中 `DateUtils` 是 `commons-lang3` 库提供的工具方法，`getDayOfMonth` 方法的实现为

```java
public int getDayOfMonth(Date date) {
    Calendar calendar = Calendar.getInstance();
    calendar.setTime(date);
    return calendar.get(Calendar.DAY_OF_MONTH);
}
```

在信用卡的场景中，还款日一般与结账日在同一月份或者次月，几乎没有间隔 2 个月或以上的情况存在。然而在我们自己的业务中遇到与信用卡类似，但又有一些差异的场景。还款日可以在结账日之后几天、一个月、两个月或更多，这时候就不能简单的记某一天了，需要记录完整的日期，即包含年月日。

举个例子：账单日为 2020-07-16，还款日可以为 2020-07-23、2020-08-05、2020-08-26、2020-11-21、2021-03-03 等等选择。

带上了年月需要用户在选择是改变已有的习惯的，同时方便我们计算还款日和结账日之间的相差的月份数。

```java
public int getMonthDifference(Date date1, Date date2) {
    Calendar calendar1 = Calendar.getInstance();
    Calendar calendar2 = Calendar.getInstance();

    calendar1.setTime(date1);
    calendar2.setTime(date2);

    int year1 = calendar1.get(Calendar.YEAR);
    int year2 = calendar2.get(Calendar.YEAR);

    int month1 = calendar1.get(Calendar.MONTH);
    int month2 = calendar2.get(Calendar.MONTH);

    return (year2 * 12 + month2) - (year1 * 12 + month1);
}
```

有了前面的基础就可以计算还款日期：

1. 计算账单日和还款日相差的的月份数；
2. 在记账日期的基础上加第 1 步的月份数得到一个新的日期；
3. 重置第 2 步日期的日为还款日就得到还款日期。

将以上步骤翻译成代码就是

```java
public Date calculateDueDate(Date postDate, Date statementDate, Date dueDate) {
    int postDay = getDayOfMonth(postDate);
    int statementDay = getDayOfMonth(statementDate);
    int dueDay = getDayOfMonth(dueDate);
    int monthDifference = Math.abs(getMonthDifference(statementDate, dueDate));

    Date date1 = DateUtils.addMonths(postDate, postDay < statementDay ? monthDifference : monthDifference + 1);
    Date date2 = DateUtils.setDays(date1, dueDay);

    return date2;
}
```

完~

参考资料：

1. [信用卡详细消费时间](http://www.jiduu.com/news/478282.html)
2. [对账单内容和名词解释](https://www.boci.com.hk/GBC/chs/definition.htm)
3. [中国银行对账单英汉对照](https://wenku.baidu.com/view/9a412e03f01dc281e53af0ee.html)