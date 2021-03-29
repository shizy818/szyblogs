---
title:  "supertest expect"
mathjax: true
layout: post
date:   2016-07-14 11:00:12 +0800
categories: javascript
---

之前将原Spring Boot项目搬到Nodejs运行环境，框架选择了IBM收购的Strongloop/loopback。
样例程序中选择了Mocha + supertest进行测试，所以我也就在这个套路上走下去。很快写了大约20+测试用例，
一切跑起来都很正常，于是乎自信满满将后台程序推送到Bluemix云端供前端开发人员调用测试。

很快同事就跑来跟我说，第一个Register API就不对，POST请求返回的是Status 200，
而不是预想的Status 201！瀑布汗。于是找到测试注册的代码：

{% highlight javascript %}
it('should register OK with correct request', function(done) {
    json('post', '/api/v1/accounts/register')
      .send({
        email: 'xxx@xxx.com',
        password: 'passwd'
      })
      .expect(201)
      .end(function(err, res) {
        assert(typeof res.body === 'object');
        assert.equal(res.body.email, 'xxx@xxx.com');
        assert(typeof res.body.id === 'number');
        done();
      });
  });
{% endhighlight %}


一试，确实返回Status 200的时候测试用例也通过了。对比`loopback-example-app-logic`里的测试代码`rest_api_test.js`，
看了半天没找出来问题在哪儿。纠结了好一阵子，怀疑到了`supertest`的身上，于是乎跑到[supertest github][supertest-gh]上
去看官方例子，发现这么一段话：

> If you are using the .end() method .expect() assertions that fail will not throw - they will return the
> assertion as an error to the .end() callback. In order to fail the test case, you will need to rethrow
> or pass err to done(), as follows:
{% highlight javascript %}
describe('GET /users', function() {
  it('respond with json', function(done) {
    request(app)
      .get('/users')
      .set('Accept', 'application/json')
      .expect(200)
      .end(function(err, res) {
        if (err) return done(err);
        done();
      });
  });
});
{% endhighlight %}

很显然这就是问题所在，加上一行抛出`err`的代码即搞定。但是expect()的语法很容易让人误认为不匹配时测试用例失败。

{% highlight javascript %}
it('should register OK with correct request', function(done) {
    json('post', '/api/v1/accounts/register')
      .send({
        email: 'xxx@xxx.com',
        password: 'passwd'
      })
      .expect(201)
      .end(function(err, res) {
        if (err) throw err;
        assert(typeof res.body === 'object');
        assert.equal(res.body.email, 'xxx@xxx.com');
        assert(typeof res.body.id === 'number');
        done();
      });
  });
{% endhighlight %}

[supertest-gh]:https://github.com/visionmedia/supertest
