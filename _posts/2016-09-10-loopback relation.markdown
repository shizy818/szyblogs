---
title:  “loopback & Hibernate relation”
mathjax: true
layout: post
date:   2016-09-10 08:00:12 +0800
categories: nodejs express
---

在loopback的框架里自带了ORM机制，即对Model和Database做了一一对应，不再需要开发人员自己动手写SQL语句。
当然我认为这极大的提高了编程效率，但是涉及到Model之间的关系（OneToMany，ManyToOne，ManyToMany）时，
我们需要考虑一下其底层效率。下面将其和著名的老牌ORM Hibernate做一个简单对比。

举个例子，假如我们有2个客户，每个客户可能有多个联系住址:  
    Customer  <---  1 : N  ---> Address


A. 使用loopback的Include方式返回Customer
{% highlight javascript %}
Customer.find({
  include: 'addresses'
})
{% endhighlight %}

后台的SQL语句如下:
{% highlight sql %}
Query     SELECT name,id FROM Customer ORDER BY id
Query     SELECT city,customerId,id FROM Address WHERE customerId IN (1,2) ORDER BY id
{% endhighlight %}

B. 使用Hibernate的注解方式
{% highlight javascript %}
// Customer.java
@OneToMany(mappedBy = "customer", fetch = FetchType.EAGER)
@Cache(usage = CacheConcurrencyStrategy.NONSTRICT_READ_WRITE)
private Set<Address> addresses = new HashSet<>();

// Address.java
@ManyToOne
@JoinColumn(name="customer_id")
@JsonIgnore
private Customer customer;
{% endhighlight %}

后台的SQL语句如下
{% highlight sql %}
Hibernate: select customer0_.id as id1_0_, customer0_.name as name2_0_ from Customer customer0_
Hibernate: select addresses0_.customer_id as customer3_0_0_, addresses0_.id as id1_1_0_, addresses0_.id as id1_1_1_, addresses0_.customer_id as customer3_1_1_, addresses0_.city as city2_1_1_ from Address addresses0_ where addresses0_.customer_id=?
Hibernate: select addresses0_.customer_id as customer3_0_0_, addresses0_.id as id1_1_0_, addresses0_.id as id1_1_1_, addresses0_.customer_id as customer3_1_1_, addresses0_.city as city2_1_1_ from Address addresses0_ where addresses0_.customer_id=?
{% endhighlight %}

这是很有名的Hibernate 1+N问题， 即Hibernate先用一条SQL语句取回所有Customer，然后再发出N条SQL语句取回子表Address的数据。
理论上可以用FetchMode来解决这个问题:
1. FetchMode.SELECT : 默认，1+N
2. FetchMode.JOIN : 用一条JOIN语句取出Customer和Address
3. FetchMode.SUBSELECT : 2条SQL语句，第二条用IN查询关联Address

后台SQL语句如下
{% highlight sql %}
Hibernate: select customer0_.id as id1_0_, customer0_.name as name2_0_ from Customer customer0_
Hibernate: select addresses0_.customer_id as customer3_0_1_, addresses0_.id as id1_1_1_, addresses0_.id as id1_1_0_, addresses0_.customer_id as customer3_1_0_, addresses0_.city as city2_1_0_ from Address addresses0_ where addresses0_.customer_id in (select customer0_.id from Customer customer0_)
{% endhighlight %}

坑爹的是在我的测试环境中，FetchMode.JOIN不起效，有人说FetchMode.JOIN和FetchType.LAZY冲突，也有人说跟Query interface和
Criteria interface有关，我还没有搞清楚为啥我的环境失败。当然Hibernate很强大的是，我们还有其他方法强制使用JOIN，这就是HQL:
{% highlight javascript %}
public interface CustomerRepository extends JpaRepository<Customer,Long> {
	@Query(value = "select customer from Customer customer join fetch customer.addresses")
	List<Customer> findAll();
}
{% endhighlight %}

后台SQL语句如下
{% highlight sql %}
Hibernate: select customer0_.id as id1_0_0_, addresses1_.id as id1_1_1_, customer0_.name as name2_0_0_, addresses1_.customer_id as customer3_1_1_, addresses1_.city as city2_1_1_, addresses1_.customer_id as customer3_0_0__, addresses1_.id as id1_1_0__ from Customer customer0_ inner join Address addresses1_ on customer0_.id=addresses1_.customer_id
{% endhighlight %}

# 总结

个人的感觉是Hibernate很强大，但是太复杂，作为一个凡人，我需要一个通俗易懂，简洁明了的方案。与其这样，
还不如就像loopback一样默认给我SUBSELECT方案，然后不用管其他配置了（大家可以发现loopback的机制和
Hibernate FetchMode.SUBSELECT基本是一致的）。

loopback是个比较新的玩意，还需要不停的进步发展。比如如下的问题，我觉得应该是一个defect；
通过customer.addresses.create插入时间比Address.create多出一倍，感觉是有问题的。
{% highlight javascript %}
let addresses = [];
let start = process.hrtime();
Customer.create({name: 'shi'}, (err, customer) => {
  if (err) throw err;

  for(let i =0; i< 2000; i++) {
    addresses.push({city: 'BJ'});
  }

  customer.addresses.create(addresses, (err, results) => {
    if (err) throw err;

    let diff = process.hrtime(start);
    let ms = diff[0] * 1e3 + diff[1] * 1e-6;
    console.log('The request processing time is %d ms.', ms);
  });
});
{% endhighlight %}
The request processing time is 1591.5240469999999 ms.

{% highlight javascript %}
let addresses = [];
let start = process.hrtime();
Customer.create({name: 'shi'}, (err, customer) => {
  if (err) throw err;

  for(let i =0; i< 2000; i++) {
    addresses.push({city: 'BJ', customerId: customer.id});
  }

  Address.create(addresses, (err, results) => {
    if (err) throw err;

    let diff = process.hrtime(start);
    let ms = diff[0] * 1e3 + diff[1] * 1e-6;
    console.log('The request processing time is %d ms.', ms);
  });
});
{% endhighlight %}
The request processing time is 898.9572549999999 ms.

推荐链接：
[https://docs.strongloop.com/display/public/LB/LoopBack][loopback-link]

[loopback-link]:https://docs.strongloop.com/display/public/LB/LoopBack
