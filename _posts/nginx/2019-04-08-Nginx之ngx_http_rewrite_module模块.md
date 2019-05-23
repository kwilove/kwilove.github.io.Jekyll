---
layout: post
title:  Nginx之ngx_http_rewrite_module模块
date:   2019-04-08 19:08:00 +0800
categories: 性能调优
tag: Nginx
---

* content
{:toc}


## 1. ngx_http_rewrite_module模块介绍
---
* Nginx的`ngx_http_rewrite_module`模块用于通过PCRE正则表达式修改请求URI，返回重定向，和按条件选择符合的配置信息。
* 本篇文章根据[官方文档](http://nginx.org/en/docs/http/ngx_http_rewrite_module.html)进行翻译，限于个人水平，如有错误，欢迎指正。

#### break、if、return、rewrite和set指令的处理顺序
1. 如果在`server`级别下配置了`rewrite`指令，它会在`location`指令之前执行。
2. 匹配`location`。
3. 执行匹配到的`location`中配置的`rewrite`重写规则，重写URI。
4. 在URI重写后，使用新的URI循环执行第`1、2、3`步骤。
5. 循环最多重复10次，超过后Nginx将返回500 error。


## 2. break指令
---
**语法**：break
**默认值**：none
**作用域**：server、location、if
**作用**：停止执行当前虚拟主机（server{}）的后续rewrite指令。
**示例**：

```nginx
if ($slow) {
    limit_rate  10k;
    break;
}
```


## 3. if指令
---
**语法**：if(condition) {...}
**默认**：none
**作用域**：server、location
**作用**：判断condition条件的真假，如果为true，定义在大括号内的模块指令将被执行，请求将按照`if`指令内的配置处理；`if`指令内的配置会继承以前的配置级别。

#### condition表达式：
* 一个变量名称；如果一个变量的值是空字符串""或"0"，则表示false；在1.0.1版本之前，任何以"0"开头的字符串都被认为是false；
* 使用`=`和`!=`运算符比较变量和字符串；
* 使用`~*`和`~`运算符对变量进行正则表达式匹配：
    * `~`区分大小写；
    * `~*`不区分大小写，例如firefox匹配到FireFox；
    * `!~*`和`!~`表示否定，表示不匹配。
    * 使用括号`()`捕获的正则表达式，后续可以通过`$1`-`$9`使用；
    ```
    正则表达式： /users/(.*)$
    请求uri：   /users/get/100
    $1的值：    get/100
    ```
    * 如果正则表达式包含`}`或`;`字符，则整个表达式应该用单引号或双引号括起来。
* 使用`-f`和`!-f`运算符检测一个文件是否存在；
* 使用`-d`和`!-d`运算符检测一个目录是否存在；
* 使用`-e`和`!-e`运算符检测一个文件/目录/符号链接是否存在；
* 使用`-x`和`!-x`运算符检测一个文件是否可执行。

**示例**：

```nginx
if ($http_user_agent ~ MSIE) {
    rewrite  ^(.*)$  /msie/$1  break;
}
if ($http_cookie ~* "id=([^;] +)(?:;|$)" ) {
    set  $id  $1;
}
if ($request_method = POST ) {
    return 405;
}
if (!-f $request_filename) {
    break;
    proxy_pass  http://127.0.0.1;
}
if ($slow) {
    limit_rate  10k;
}
# 内置变量$invalid_referer的值由valid_referers指令提供。
if ($invalid_referer) {
    return   403;
}
```


## 4. return指令
---
**语法**：return code [text]; return code URL; return URL;
**默认值**：none
**作用域**：server、location、if
**作用**：
* 此指令结束规则的执行，并返回状态码给客户端，在0.7.51的版本之前只能返回以下状态码：204、400、402-406、408、410、411、413、416和500-504；
* 此外，可以通过返回非标准代码444，在不发送相应头的情况下关闭一个连接；
* 从0.8.42版本起，return指令可以指定重定向URL（针对状态码301、302、303、307），或者指定响应体text（针对其他状态码）；
    * 响应体文本和重定向URL可以包含变量；
    * 有一种特殊情况，可以将重定向URL指定为当前服务器的本地URI，在这种情况下，将根据请求协议($scheme)、server_name_in_redirect指令和port_in_redirect指令形成完整的重定向URL。
* 此外，带有302状态码的临时重定向URL可以被指定为唯一参数，这个参数应该以"http://"、"https://"或者"$scheme"字符串开头，可以包含变量；
    ```nginx
    # 监听www.abc.com/abc.com的80端口，将http请求转为到https请求。
    server {
        listen 80;
        server_name www.abc.cn abc.cn;
        return 301 https://$server_name$request_uri;
    }
    ```
* 状态码307直到版本1.1.16和1.0.13才被视为重定向。
* 状态码308直到1.13.0版本才被视为重定向。


## 5. rewrite指令
---
**语法**：rewrite regex replacement [flag];
**默认**：none
**作用域**：server、location、if
**作用**：
* 如果指定的正则表达式能匹配请求URI，URI将被更改为指定的replacement字符串；
* rewrite指令将按照它们在配置文件中出现的位置顺序执行；
* 可以使用标志flag终止对指令的进一步处理，flag可以选择下列值之一；
    * `last` 停止处理当前的ngx_http_rewrite_module指令集，并开始搜索一个能匹配更改后URI的location规则;
    * `break` 与break指令一样，停止处理当前的ngx_http_rewrite_module指令集;
    * `redirect` 返回一个带有302状态码的临时重定向，如果替换字符串不以"http://"、"https://"或"$scheme"开头，则可以使用这个值;
    * `permanent` 返回带有301状态码的永久重定向。
* 如果替换字符串以"http://"、"https://"或"$scheme"开头，处理工作将停止，并将重定向的结果返回给客户端。
* 完整的重定向URL将根据请求协议($scheme)、server_name_in_redirect指令和port_in_redirect指令形成
```nginx
server {
    ...
    rewrite ^(/download/.*)/media/(.*)\..*$ $1/mp3/$2.mp3 last;
    rewrite ^(/download/.*)/audio/(.*)\..*$ $1/mp3/$2.ra  last;
    return  403;
    ...
}

# 如果上面这个例子的指令放在`location /download/ {...}`内部，则需要将`last`标志换成`break`，否则nginx将会进行10次循环，最终返回500错误，修改如下。
server {
    ...
    location /download/ {
        rewrite ^(/download/.*)/media/(.*)\..*$ $1/mp3/$2.mp3 break;
        rewrite ^(/download/.*)/audio/(.*)\..*$ $1/mp3/$2.ra  break;
        return  403;
    }
    ...
}
```
* 注意重写表达式只对相对路径有效，如果你想匹配主机名，你应该使用if条件语句，如下：
```nginx
if ($host ~* www\.(.*)) {
    set $host_without_www $1;
    # $1 contains '/foo', not 'www.mydomain.com/foo'
    rewrite ^(.*)$ http://$host_without_www$1 permanent; 
}
```
* 如果替换字符串包含新的请求参数，则将在新参数之后追加先前的请求参数。如果不希望这样，在替换字符串的末尾加上问号可以避免最新就请求参数，例如:
```nginx
rewrite ^/users/(.*)$ /show?user=$1? last;
```
* 如果正则表达式包含`}`或`;`字符，则整个表达式需要用单引号或双引号括起来。

## 6. rewrite_log指令
---
**语法**：rewrite_log on | off;
**默认**：rewrite_log off;
**作用域**：http, server, location, if
**作用**：开启或关闭将`ngx_http_rewrite_module`模块指令处理结果的`notice`级别日志写入`error.log`文件中。


## 7. set指令
---
**语法**：set $variable value;
**默认**：none
**作用域**：server, location, if
**作用**：为指定的变量设置一个值。该值可以包含文本、变量及其组合。


## 8. uninitialized_variable_warn指令
---
**语法**：uninitialized_variable_warn on | off;
**默认**：uninitialized_variable_warn on;
**作用域**：http, server, location, if
**作用**：控制是否记录未初始化变量的警告信息。