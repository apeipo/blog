---
title: nginx中的last和break
tags: [Nginx]
category: [Nginx]
date: 2017-03-17 17:02:22
---

## last后的操作
之前看大部分文档说的是：
**last**是从server域开始重新请求。
nginx官方解释：
>**last：**
    stops processing the current set of ngx_http_rewrite_module directives followed by a search for a new location matching the changed URI;
**break：**
    stops processing the current set of ngx_http_rewrite_module directives;

```sh
	#location中的last
	rewrite /test2 /tt break;
   location /test {
       rewrite /test2 /test3 break;
       rewrite /test /test2 last;
       rewrite /test2 /test3 break;
   }
   location /test2 {
       return 508;
   }
   location /test3 {
       return 503;
   }
```
请求/test的结果:508,说明：
1. location域里的last终止当前location下的所有rewrite(因为没有跳到test3),重启location匹配
2. 重新发起请求后是从匹配location开始,不是从server域的rewrite开始(如果这样的话，应该跳转/tt,返回404)

## server域里的last和break

```sh
			#server中的last
        rewrite /tt /index.html break;
        rewrite /test2 /tt last;
        //rewrite /tt /index.html break;
        location /test {
            rewrite /test2 /test3 break;
            rewrite /test /test2 last;
            rewrite /test2 /test3 break;
        }
        location /test2 {
            return 508;
        }
        location /test3 {
            return 503;
        }

        location / {
            root   html;
            index  index.html index.htm;
        }
```
请求/test2,返回404,将rewrite /tt的位置调整后,请求test2的返回结果还是404，说明：
server域里的last，会终止server的rewrite，进入location匹配（而不是从server头发起请求）。
所以，**server域里的break和last的作用没有区别**，都是终止rewrite进入location匹配.
总结：
1.server中的break终止rewrite进入location匹配
2.location中的break，终止当前请求的匹配工作，进入执行阶段
## location中rewrite的作用
补充一个nginx配置的问题，之前在nginx中加了如下配置，配置一些url的日志不打印

```nginx
    location ~ ^/+receiver.php$ {
        access_log off;
    }   
    location ~ \.php(.*)$ {
      fastcgi_pass    unix:/home/app/php/var/php-cgi.sock;
      fastcgi_split_path_info            ^(.+\.php)(.*)$;
      fastcgi_param   SCRIPT_FILENAME    $document_root$fastcgi_script_name;
      fastcgi_param   PATH_INFO    $fastcgi_path_info;
      include         fastcgi_params;
      break;
    }   
```
配置完成后，发现所有的`/receiver.php`请求都直接返回了，没有经过php处理
location阶段如果没有进行break的话，不是应该进入到下一个location么？如下的配置,当请求/test时，返回的是509。

```
location /test2 {
    return 509;
}
location /test {
    rewrite /test /test1;
    rewrite /test1 /test2;
}
```
由此看来，rewrite指令也会影响location阶段的后续处理，整理了下location中rewrite指令最后一个参数几种情况分别进行实验。

### 有rewrite
#### 1.break
终止rewrite，进入请求处理阶段
#### 2.last
终止rewrite，重新开始匹配location
#### 3.redirect , perminate
301和302
#### 4.default(rewrite后不带指令)
继续执行下一条rewrite指令，如果该条指令为最后一条，则执行处理请求的指令（如fastcgi_pass，proxy_pass），没有则继续匹配其他location。
### 无rewrite 
直接进入请求处理阶段

