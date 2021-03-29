---
title:  “SQL EXISTS & IN 语法”
mathjax: true
layout: post
date:   2017-08-25 08:00:12 +0800
categories: sql
---

# EXISTS

表A:

 ID   | Name
:----:|:----:
 1 | A1
 2 | A2
 3 | A3

表B:

ID |  AID  |   Name
:----:|:----:|:----:
1 | 1 | A1
2 | 2 | A2
3 | 2 | A3

EXISTS用于检查子查询是否至少会返回一行数据，该子查询实际上并不返回任何数据，而是返回值True或False
{% highlight sql %}
SELECT ID,NAME FROM A WHERE EXISTS (SELECT * FROM B WHERE A.ID=B.AID)
{% endhighlight %}


执行结果为：

1 | A1
2 | A2

可以按照如下分析:

SELECT ID,NAME FROM A WHERE EXISTS (SELECT * FROM B WHERE B.AID=１)  
---> SELECT * FROM B WHERE B.AID=１有值返回真所以有数据

SELECT ID,NAME FROM A WHERE EXISTS (SELECT * FROM B WHERE B.AID=2)  
---> SELECT * FROM B WHERE B.AID=２有值返回真所以有数据

SELECT ID,NAME FROM A WHERE EXISTS (SELECT * FROM B WHERE B.AID=3)  
---> SELECT * FROM B WHERE B.AID=３无值返回假所以没有数据

# IN

EXISTS与IN最大的区别在于IN引导的子句只能返回一个字段。
{% highlight sql %}
SELECT ID,NAME FROM A WHERE ID IN (SELECT AID FROM B WHERE A.ID=B.AID)
{% endhighlight %}

# 练习

关系表
BOOKS: bno, bname,...

USER: uid, uname, ....

BORROW: borrowid, uid, bno,...

- Borrow约束性
{% highlight sql %}
CREATE TABLE BORROW (
    BORROWID INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
    BNO INT FOREIGN KEY REFERENCES BOOKS(BNO),
    UID INT FOREIGN KEY REFERENCES USER(UID),
  ...
);
{% endhighlight %}

- 找出借书超过5本的读者和所借图书册数
{% highlight sql %}
SELECT UID, COUNT(*) FROM BORROW GROUP BY UID HAVING COUNT(*) > 5;
{% endhighlight %}

- 查询借阅了“水浒”一书的读者
{% highlight sql %}
SELECT * FROM USER u WHERE EXISTS (
    SELECT  * FROM BORROW a, BOOKS b
      WHERE a.BNO = b.BNO AND b.BNAME = '水浒' AND a.UID = u.UID
  )
{% endhighlight %}

- 查询现有图书中价格最高的图书
{% highlight sql %}
SELECT * FROM BOOKS WHERE PRICE = (
    SELECT MAX(PRICE) FROM BOOKS
  )
{% endhighlight %}

- 查询当前借了“计算方法”但是没有借"计算方法习题"的读者，并降序排列
{% highlight sql %}
SELECT a1.UID FROM BORROW a1, BOOKS b1
    WHERE a1.BNO = b1.BNO AND b1.BNAME = ’计算方法‘
EXCEPT
SELECT a2.UID FROM BORROW a2, BOOKS b2
    WHERE a2.BNO = b2.BNO AND b2.BNAME = '计算方法习题'
ORDER BY a1.UID DESC
{% endhighlight %}
或者
{% highlight sql %}
SELECT a.UID FROM BORROW a, BOOKS b
    WHERE a.BNO = b.BNO AND b.BNAME = '计算方法'
    AND NOT EXISTS (
      SELECT * FROM BORROW aa, BOOKS bb
        WHERE aa.BNO = bb.BNO AND bb.BNAME = '计算方法习题' AND aa.UID = a.UID
    )
  ORDER BY a.UID DESC
{% endhighlight %}

- 从BOOKS表中删除无人借阅的图书
{% highlight sql %}
DELETE FROM BOOKS WHERE NOT EXISTS (
    SELECT * FROM BORROWS WHERE BNO = BOOKS.BNO
  )
{% endhighlight %}

- 建立一个视图，显示"力01"班学生的借书信息（只要求显示姓名和书名）
{% highlight sql %}
CREATE VIEW V_VIEW AS
SELECT a.NAME,b.BNAME FROM BORROW ab, USER a,BOOKS b
    WHERE ab.UID = a.UID AND ab.BNO = b.BNO AND a.CLASS = '力01'
{% endhighlight %}

- 查询当前同时借有"计算方法"和"组合数学"两本书的读者，输出其借书卡号，并按卡号升序排序输出
{% highlight sql %}
SELECT a.UID FROM BORROW a,BOOKS b
WHERE a.BNO=b.BNO AND b.BNAME IN('计算方法','组合数学')
GROUP BY a.UID HAVING COUNT(*)=2
ORDER BY a.UID DESC
{% endhighlight %}
