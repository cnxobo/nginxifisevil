Nginx的配置是一种`声明式的配置`，虽然有`if`指令，但是跟传统意义上的编程语言是完全不同的。在实际使用的过程中，使用`if`指令会产生有很多很奇怪的事情，由其是在对它没有充分了解之前，经常会误用。Nginx官方也提醒在`location`里使用`if`是灾难，如果一定要使用`if`，需要测试测试再测试.
[If is Evil... when used in location context
](https://www.nginx.com/resources/wiki/start/topics/depth/ifisevil/)

对于`声明式的配置`来说:
1. Nginx的`if`指令条件满足时，会在`if`块里创建一个嵌套的子`location`块，然后再执行`if`块内的声明和内容处理器(content handler)。
2.  在一个配置块里只有一个`if`语句和一些其他的声明，由于`if`的存在，会导致有些声明不生效。为了保证这些声明都生效，最好在`if`块内，重新声明一次。
3. 在一个配置块里，如果有两个`if`指令都满足条件，只有第二个`if`会被执行。

Case 1
```
location /case1 {
    set $a 32;
    if ($a = 32) {
        set $a 56;
    }
    set $a 76;
    proxy_pass http://127.0.0.1:$server_port/$a;
}

location ~ /(\d+) {
    echo $1;
}
```
访问`/case1`时会返回76.
```
~ curl localhost/case1
76
```
Nginx 是这样工作的:
1. Nginx先按照配置顺序执行所有的rewrite指令:
```Shell
set $a 32;
if ($a = 32) {
  set $a 56;
}
set $a 76;
```
$a最后值是76

2. 因为条件`$a = 32`成立，所以Nginx就进入`if`的内部块。
3. 这个内部块没有任何内容处理器(content handler),[ngx_proxy](http://wiki.nginx.org/NginxHttpProxyModule) 就会继承外部块的内容处理器(content handler)。(see [src/http/modules/ngx_http_proxy_module.c:2025](https://github.com/nginx/nginx/blob/branches/stable-0.8/src/http/modules/ngx_http_proxy_module.c)).
4. 同时配置指定的[proxy_pass](http://wiki.nginx.org/NginxHttpProxyModule#proxy_pass)也会被`if`内部块继承。(see [src/http/modules/ngx_http_proxy_module.c:2015](https://github.com/nginx/nginx/blob/branches/stable-0.8/src/http/modules/ngx_http_proxy_module.c))
5. 请求结束。(其实流程进了`if`块再也没出来)

所以,外面的`proxy_pass`指令并没有执行，真正提供服务的是`if`内部块。

如果我们的`if`块声明了内容处理器(content handler)会发生什么
Case2
```
location /case2 {
      set $a 32;
      if ($a = 32) {
          set $a 56;
          echo "a = $a";
      }
      set $a 76;
      proxy_pass http://127.0.0.1:$server_port/$a;
  }

location ~ /(\d+) {
    echo $1;
}
```
当我们访问`/case2`时，我们会得到
```
~ curl localhost/case2
a = 76
```
这中间都发生了什么:
1. Nginx先按照配置顺序执行所有的rewrite指令:
```Shell
set $a 32;
if ($a = 32) {
  set $a 56;
}
set $a 76;
```
$a最后值是76

2. 因为条件`$a = 32`成立，所以Nginx就进入`if`的内部块。
3. 这个内部块有内容处理器(content handler)`echo`,所以$a的值（a = 76）就返回给了客户端。
4. 请求结束。(流程进了`if`块再也没出来，和Case1一样)

我们再调整一下:
Case 3
```
location /case3 {
      set $a 32;
      if ($a = 32) {
          set $a 56;
          break;

          echo "a = $a";
      }
      set $a 76;
      proxy_pass http://127.0.0.1:$server_port/$a;
}

location ~ /(\d+) {
      echo $1;
}
  ```
  这一次我们在`if`块内添加了`break`指令. 它会阻止nginx的`ngx_rewrite`指令的执行。因此我们得到
```
~ curl localhost/case3
a = 56
```
  
这一次Nginx是这样工作的:
1. Nginx先按照配置顺序执行所有的rewrite指令:
```Shell
set $a 32;
if ($a = 32) {
  set $a 56;
  break;
}
```
$a最后值是56

1. 因为条件`$a = 32`成立，所以Nginx就进入`if`的内部块。
2. 这个内部块有内容处理器(content handler)`echo`,所以$a的值（a = 56）就返回给了客户端。
3. 请求结束。(流程进了`if`块再也没出来，和Case1一样)

还有一个`if`块继承配置的场景:
case 4:
```
location /case4 {
      set $a 32;
      if ($a = 32) {
          return 404;
      }
      set $a 76;
      proxy_pass http://127.0.0.1:$server_port/$a;
      more_set_headers "X-Foo: $a";
      add_header X-Bar $a;
}

location ~ /(\d+) {
      echo $1;
}
```
```
~ curl -i localhost/case4
HTTP/1.1 404 Not Found
Server: openresty/1.19.9.1
Date: Thu, 09 Sep 2021 07:28:48 GMT
Content-Type: text/html
Content-Length: 159
Connection: keep-alive
X-Foo: 32

<html>
<head><title>404 Not Found</title></head>
<body>
<center><h1>404 Not Found</h1></center>
<hr><center>openresty/1.19.9.1</center>
</body>
</html>
```
只有`X-Foo` Header被添加了，`X-Bar` Header却没有
这个可能是你想要的或者不是
这里`ngx_header_more`的`more_set_headers`也会被继承
`add_header`指令虽然忽略X-Foo header，但是它依旧被继承了，是因为`add_header`的header filter会忽略4XX响应。  :-p

注: `echo`,`ngx_header_more`模块都是三方模块，如果需要测试以上示例，建议使用openresty.

或者直接使用 docker 测试
```
git clone https://github.com/cnxobo/nginxifisevil.git
docker build -t nginxifisevil .
docker run --rm  -d -p 80:80 --name nginxifisevil nginxifisevil
curl localhost/case1
curl localhost/case2
curl localhost/case3
curl localhost/case4
curl -i localhost/case4
```
相关博客:
[本文地址](https://xobo.org/nginx-if/)
[Nginx proxy_pass 的URL Mapping](https://xobo.org/nginx-proxy_pass-url-mapping/#If_bug)
参考资料:
[How nginx "location if" works | Human & Machine](https://agentzh.blogspot.com/2011/03/how-nginx-location-if-works.html)
[How does if condition work inside location block in nginx conf? - Stack Overflow](https://stackoverflow.com/questions/35257968/how-does-if-condition-work-inside-location-block-in-nginx-conf)
