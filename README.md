# 安装
1. 安装node 8.0以上版本
2. npm install -d webcontext

# 简介
webcontext是一个轻量级的web开发框架和web容器, 它能提供像php,jsp一样的页面开发体验，让开发者专注于业务开发，而不需要关注node.js底层的技术细节和各种中间件的用法，是目前api最简洁的nodejs web开发框架，能够让你编写最少的代码快速实现业务。特性如下：

* URL自动路由到相应的js文件,类似于php,jsp的页面机制
* 自动解析请求表单、JSON
* 内置静态文件服务器，不再需要nginx
* 内置模板引擎(基于ejs)
* 内置数据库存取(基于mysql)
* 内置Session(支持多进程、分布式)
* all in one，不需要中间件，配置简单一键运行
  
# 快速开始
### app.js

```js
const WebContext = require('webcontext');
const app = new WebContext();
app.listen();
```
1. 在app.js中引入webcontext。
2. 调用listen方法。
3. 运行node app.js启动。

默认监听80端口,如果要改变监听端口，请在根目录的web.config.json文件中修改port属性
#### web.config.json
```js
{
    port:"80"
}
```

### ./service/index.js
```js
module.exports= {  
    onRequest() {       
        var data=[
            {id:1,title:"javascript"},
            {id:2,title:"node.js"},
            {id:3,title:"mysql"}
        ];
        this.render({list:data});
    }
}
```
1. 在项目目录service目录下建立index.js，该文件将自动处理/index路径的请求，代码如上所示
2. 编写onRequest方法
3. 调用this.render方法，传入一个对象做为数据源，将自动加载同名的扩展名为ejs的模板文件，渲染结果会直接输出到reponse.body中。也可以传入一个字符串不使用模板直接输出。
### ./service/index.ejs
```html
<ul>
<% for(var i=0;i<list.length;i++){ %>
<li><%=list[i].title%></li>
<%}%>
</ul>
```
 

# URL映射

URL映射就是一个URL请求由哪块代码（类、函数）来处理,webcontext根据js文件路径自动处理URL映射，类似于jsp和php的页面机制，文件必须存放在/service目录下，支持多级子目录.

例如请求请求http://localhost/todo/list将自动映射到/service/todo/list.js文件，js文件必须使用exports导出一个对象，该对象必须实现onRequest方法, hello,world示例页面代码如下

### /service/index.js
```js
module.exports= {  
    onRequest() {   
        this.response.body="hello,world";
    }
}
```
也可以在application对象的onRequest添加全局的URL映射，支持正则表达式，下面的代码是在每个http请求的响应头中添加server字段:
```js
const WebApp = require('webcontext');
const app = new WebApp();
app.onRequest(/.*/,function (ctx){
    ctx.response.headers["server"]="webcontext";

});
```

# 请求处理
## Post请求
使用request.data可以获取post数据，同时支持json数据和表单数据两种格式。

post 表单数据 (jQuery)：
```js
$.post("/todo/add/",{id:1,title:"hello",status:0})
```
 post json数据 (jQuery)：
```js
$.ajax({
    type : "POST",
    url:"/todo/add/",
    dataType:"json",
    data:{id:1,title:"hello",status:0}
})
```
### /service/todo/add.js 接收post数据代码：
```js
module.exports= {  
    onRequest() {  
        var data=this.request.data;
        this.response.body=JSON.stringify(data);      //{id:1,title:"hello"}
    }
}
```

# 响应处理
## 方式一：this.response.body
```js
this.response.body="hello,world"
```

## 方式二：this.render(string)
```js
this.render("hello,world")
```

## 方式三：this.render(templateString,data)
```js
this.render("hello,<%=message%>",{message:"world"})
```

## 方式四：this.render(data)
调用render方法，传入object对象，将使用与当前文件同名的扩展名为.ejs的文件做为模板进行渲染。

service/hello.js
```js
this.render("hello,<%=message%>",{message:"world"})
```
service/hello.ejs
```html
<html>
hello,<%=message%>
</html>
```
# 数据库操作
webcontext内置支持mysql数据库，数据库连接字符串在web.config.json中配置，配置好后，在程序启动时将自动连接数据库。

使用this.database获取数据库操作对象，该对象提供select,insert,update,delete,query,use几个方法对数据库进行操作。

web.config.json
```js
{
    port:"8080",
    database:{ 
        host:'127.0.0.1',//db server ip
        port:'3306',// db server port
        user:'root', //account
        password:'123456', //password
        database:'todo_db'  //database name or schema's name
    }
}
```
## insert


调用database.insert(tableName,columns)插入数据库，tableName参数为表名称，columns为插入的数据，例如{id:1,title:"hello"}

#### /service/todo/add.js 代码：
```js
module.exports= {  
    onRequest() {  
        var data=this.request.data;     //{id:1,title:"hello"}
        this.database.insert("todo",data);  
    }
}
```
##  replace
与insert用法相同，将生成replace into ...的sql 语句执行。前提条件是数据表相应的字段必须设置unique约束。

##  update
用法：database.update(tableName,columns,where)，如果colums中定义了id参数，则自动使用id字段做where条件

```js
module.exports= {
    onRequest() {
        this.database.update("todo",{
            id:this.request.data["id"],
            title:this.request.data["title"],
            status:0
        }).then((result)=>{
            this.render(JSON.stringify({msg:"update ok!!!"}))
        })
        
        
    }
}
```
##  select 

调用database.select(tableName,options) 查询数据库
options的数据结构：
{
    columns:["id","title"], //可选参数，定义返回的列
    orderBy:"createTime", //排序
    where:{id:1},  //where 条件,and关系
    pageIndex:1,  //分页查询页码
    pageSize:20  //分页查询页大小
}

返回值为promise对象，async/await，直接接收返回的结果集。也可以使用传统的then方法传入回调函数获取结果集。
如果要使用复杂的查询条件，请使用database.query(sql,params) 传入自定义的sql执行



```js
module.exports= {
    async onRequest() {
        var result=await this.database.select("todo",{orderBy:"createTime desc "})
        this.render({list:result});    
    }
}
```

#### /service/todo/list.js 代码：
module.exports= {
    async onRequest() {
        var result=await this.database.select("todo",{
            where:{status:0},
            orderBy:"createTime desc "
        })
        this.render({list:result});    
    }
}

# Session存取
为了支持多进程和分布式，webcontext使用mysql数据库内存表存储Session。因此使用Session之前，必须确保在web.config.json中配置好database数据库连接， 进程首次启动时将自动创建内存表。

### Session 读取
在this.session.load()回调函数中得到sessionData对象

### Session 写入
使用 this.session.set({key:val}) 直接写入session,支持一次写入多个值。返回promise对象，如需监听写入成功，则在then方法中注册回调函数执行后续操作。

以下代码是在session中写入userName字段，写入成功后，再从session中读取userName字段
```js
module.exports= {
    onRequest() {       
        this.session.set({
            userName:"windy"
        }).then(()=>{
            this.session.load( (sessionData)=>{
                this.response.body="hello,"+sessionData["userName"];
            })
        })
    }
}
```
# 其它常用操作

### 重定向
response.redirect("/page1.htm")
### 文件下载
response.writeStrem(fs.createReadStream(fileName))
### 文件上传
request.files
### 请求头读取
request.headers["fieldName"]
### 响应头写入
response.headers["fieldName"]="fieldValue"
### cookie读取
request.cookies["userName"];
### cookie写入
response.cookies["userName"]="windyfany" //会话cookie

response.cookies["userName"]={
    value:"windy",
    domain:'localhost',
    path:'/',
    maxAge:1000*60*60*1,
    expires:new Date(),
    httpOnly:false
}]

# 静态文件服务器
/frontend目录中存储静态文件，如html,css,图片，js等。
例如：
访问http://localhost/css/style.css时对应访问的文件路径是/frontend/css/style.css
访问http://localhost/images/logo.jpg时对应访问的文件路径是/frontend//images/logo.jpg

# 目录结构
service目录存放url映射处理类,该目录存放的js文件实现onRequest方法。
frontend是静态文件服务器的根目录，该目录存放前端的静态资源文件如css,图片，html等。
web.config.json 是配置文件，用于配置web服务端口号，数据库连接字符串，上传文件存放目录等

```
|-- service                          
|   ┠-index.js      //auto handle the http path:/index
|   ┗-index.ejs     //html template file of index.js       
|-- frontend         //static htm,js,css,image files
|   ┠-images
|   ┠-css
|   ┗-js
|-- web.config.json          // config file                   
```

# 配置文件
配置文件存放于项目根目录下web.config.json文件中，首次运行会自动生成，可配置web服务端口号，数据库连接字符串，上传文件存放目录等
```js
{
    port:"3000",
    index:"/index",
    sessionKey:"my_session_id",
    uploadDir:"./upload",
    database:{ 
        host:'127.0.0.1',
        port:'3306',
        user:'root',
        password:'windyfancy123',
        database:'todo_db'
    }
}
```
