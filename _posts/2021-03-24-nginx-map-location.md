---
layout: post 
title: 巧用map解决nginx的Location里if失效问题 
date:   2021-03-24 13:33 
description: 在Nginx的Locaiton里借助map替换if 
categories: nginx 
comments: true 
tags:
- nginx, location, map,add_header

---

## 需求: Nginx根据参数来输出不同的header

我们想用Nginx来判断一些通用的参数, 根据参数情况在输出中不同的header, 或者cookie, 那么根据正常思路, 有如下配置: 

```nginx
      location  ^~ /test1/ {
        root /home/app/services/test_html;

        if ($arg_ss != ''){
             add_header Set-Cookie "ss=$arg_ss;Domain=.yourdomain.com;Path=/;Max-Age=604800" always;
        }

        try_files $uri $uri/ /test1/index.html$query_string;
      }

```

这会运行正常吗? 它会跑到404页面, 为什么哪? 反正去掉if这几行代码它就ok了.



```nginx
      location  ^~ /test1/ {

        root /home/app/services/test_html;

        if ($arg_ss != ''){
             add_header Set-Cookie "ss=$arg_ss;Domain=.yourdomain.com;Path=/;Max-Age=604800" always;
        }

        ## Check for file existing and if there, stop ##
        if (-f $request_filename) {
            break;
        }

        # Check for file existing and if there, stop ##
        if (-d $request_filename) {
             break;
        }
        
        rewrite (.*) /test1/index.html?$query_string;
      }
```



这会运行正常吗? 页面是正常的, 但是add_header无法生效, 猜测是因为rewrite, 所以自然不会保留header  (待深入研究)


研究了一番, 发现网上有几篇文章相关:

* https://www.nginx.com/resources/wiki/start/topics/depth/ifisevil/  万恶的IF, 不知道什么时候会失效...
* https://stackoverflow.com/questions/50268485/nginx-try-files-with-add-header 遇到问题的人
* https://trac.nginx.org/nginx/ticket/97  try_files and alias problems

## 巧用map替换if
最终我们发现了 https://stackoverflow.com/questions/39548301/if-conditions-break-try-files-in-nginx-configuration

总之就是if不靠谱, 那么map可以用来替换if, 最终结果:

```nginx

         map $ssKey $ssResult {
          "" "";
          default   $ssValue;
        }

        server {
         listen 80;
         server_name  www.yourdomain.com;
         root /home/app/services/test_html;


          location  ^~ /test1/ {
            root /home/app/services/test_html;
    
            # 根据参数设置
            set $ssKey "$arg_ss";
            set $ssValue "ss=$arg_ss;Domain=.yourdomain.com;Path=/;Max-Age=604800";
    
            # 设置header, 如果为空nginx会忽略
            add_header  Set-Cookie $ssResult;
    
            try_files $uri $uri/ /test1/index.html?$query_string;
          }

      }

```

这里使用了二个技巧:
* map的key, value都可以是变量
* add_header如果值为空, nginx会忽略


运行正常!

