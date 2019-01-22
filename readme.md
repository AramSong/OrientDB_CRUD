## OrientDB로 웹애플리케이션 구현 - 190122

**get('topic/') : view.jade**

**get('topic/:id') : view.jade**

**get('topic/add') : add.jade**

​	post('topic/add')

​	get('topic/:id')

**get('topic/:id/edit') : edit.jade**

​	post('topic/:id/edit')

​	get('topic/:id')

**get('topic/:id/delete')  : delete.jade**

​	post('topic/:id/delete')

​	get('topic/')

-----------------------------------------------------------------------------------------------------------------------------------------------------------

**1. escaping**

아래와 같은 `#` ,`:` 특수기호를 특수기호가 아니게 바꾸어 주는 것을 escaping이라고 함.




![1548130923995](https://user-images.githubusercontent.com/38032500/51531052-c073b880-1e7f-11e9-8817-8b29b2bf68a3.png)

`escape`/`encodedURI`/`encodedURIComponent` 인코딩함수.

**1. escape(), unescape()** **함수**

  아스키문자에 해당하지 않는 문자들은 모두 유니코드 형식으로 변환해 줍니다. 그리니까 16진수 형태로 바꿔주는 것. => escape()

코딩한 문자열을 다시 원래대로 돌리고 싶다면 디코딩 =>unescape()

**2. encodeURI()** **함수**

  encodeURI 는 URL 주소표시를 나타내는 앞자리 특수문자는 인코딩하지 않는다. 인코딩 하지 않는 문자는 ` ; / = ? &`.주소를 통해 넘기는 파라미터를 인코딩할때 사용.

**3. encodeURLComponent()** **함수**

주소를 특수문자도 인코딩 함. 모든 문자를 인코딩하기 때문에 경로를 나타내는 /file/exe/index 값이 있다면 "/"도 인코딩한다. 이렇게 되면 서버에서 인식을 못한다. 이럴 때는 encodeURL를 사용.

![1548147408581](https://user-images.githubusercontent.com/38032500/51531056-c10c4f00-1e7f-11e9-9c76-91a7b37de59a.png)
![1548147506190](https://user-images.githubusercontent.com/38032500/51531060-c4073f80-1e7f-11e9-9239-e9d34757f30f.png)

디코딩은 decodeURIComponent()함수를 사용한다.

출처: <https://mainia.tistory.com/2428> [녹두장군 - 상상을 현실로]  

---------------------------------------------------------------------------------------------------------------------------------------------------------

### 글추가

* `app_orientdb.js`

```javascript
app.get('/topic/add',function(req,res){
  var sql = 'SELECT FROM topic';
  db.query(sql).then(function(topics){

    if(topics.length === 0){
      console.log('There is no topic record.');
      res.status(500).send('Internal Server Error');
    }
      res.render('add',{topics:topics});
  });

})
app.post('/topic/add',function(req,res){
  var title = req.body.title;
  var description = req.body.description;
  var author = req.body.author;
  var sql = 'INSERT INTO topic (title, description, author) VALUES(:title, :desc, :author)';
  db.query(sql, {
    params:{
      title:title,
      desc:description,
      author:author
    }
  }).then(function(results){
    res.redirect('/topic/'+encodeURIComponent(results[0]['@rid']));
  });
});
```

* `add.jade`

```javascript
doctype html
html
  head
    meta(charset='utf-8')
  body
    h1
      a(href='/topic') Server Side JavaScript
    ul
      each topic in topics
       li
        - rid = encodeURIComponent(topic['@rid'])
        a(href='/topic/'+rid)= topic.title
    article
      if topic
        h2= topic.title
        = topic.description
        div= 'by '+topic.author
      else
        h2 Welcome
        | This is server side javascript tutorial.

    div
      a(href='/topic/add') add
```

### 글편집 1

기존의 글(선택한 글에 대한 정보)을 가져와 해당 글을 편집할 수 있는 기능을 추가. 만약 홈일경우엔, EDIT버튼이 뜨지 않도록 해야함.

* `view.jade`

```javascript
      if topic
        li
          - rid = encodeURIComponent(topic['@rid'])
          a(href='/topic/'+rid+'/edit') edit

```

### 글편집 2

* `app_orientdb.js`

```javascript
app.get('/topic/:id/edit',function(req,res){
  var sql = 'SELECT FROM topic';
  var id = req.params.id;
  db.query(sql).then(function(topics){
      var sql = 'SELECT FROM topic WHERE @rid=:rid';
      db.query(sql,{params:{rid:id}}).then(function(topic){
        res.render('edit',{topics:topics,topic:topic[0]});
      });
  });
})
app.post('/topic/:id/edit',function(req,res){
  var sql = 'UPDATE topic SET title=:t, description=:d,author=:a WHERE @rid=:rid';
  var id = req.params.id;
  var title = req.body.title;
  var desc = req.body.description;
  var author = req.body.author;
  db.query(sql,{
    params:{
      t:title,
      d:desc,
      a:author,
      rid:id
    }
  }).then(function(topics){
    res.redirect('/topic/' +encodeURIComponent(id));
  });
})
```

* `edit.jade`

```javascript
  article
      - rid = encodeURIComponent(topic['@rid'])
      form(action='/topic/'+rid+'/edit' method='post')
       p
        input(type='text' name='title' placeholder='title' value=topic.title)
       p
        textarea(name='description' placeholder='description' )
         =topic.description
       p
        input(type='text' name='author' placeholder='author' value=topic.author)
       p
        input(type='submit')
```

### 글삭제

삭제할 경우, SQL문을 클래스명에 따라 주의해서 삭제. 제너릭 클래스 `DELETE`, VERTEX 클래스 `DELETE VERTEX`

를 통해 삭제.

* `app_orientdb.js`

```javascript
app.get('/topic/:id/delete',function(req,res){
  var sql = 'SELECT FROM topic';
  var id = req.params.id;
  db.query(sql).then(function(topics){
      var sql = 'SELECT FROM topic WHERE @rid=:rid';
      db.query(sql,{params:{rid:id}}).then(function(topic){
        res.render('delete',{topics:topics,topic:topic[0]});
      });
  });
})
app.post('/topic/:id/delete',function(req,res){
  var sql = 'DELETE VERTEX FROM topic WHERE @rid=:rid';
  var id = req.params.id;
  db.query(sql, {
    params:{
      rid:id
    }
  }).then(function(topics){
    res.redirect('/topic/'+encodeURIComponent(id));
  });
});
```

* `delete.jade`

```javascript
doctype html
html
  head
    meta(charset='utf-8')
  body
    h1
      a(href='/topic') Server Side JavaScript
    ul
      each topic in topics
         li
          - rid = encodeURIComponent(topic['@rid'])
          a(href='/topic/'+rid)= topic.title
    article
      h1= 'Delete? ' + topic.title
      - rid = encodeURIComponent(topic['@rid'])
      form(action='/topic/'+rid+'/delete' method='post')
       p
        input(type='submit' value='삭제')
      a(href='/topic/'+rid) 이전페이지로
```



