# NginxAuth
Nginx auth_request 自定义登录

## 背景
  项目中经常会引用一些开源项目，而大部分开源项目都是无需登录即可操作的，尤其是ElasticSearch，数据可以被轻易修改和删除。所以，使用最简单快捷的方法实现了基本的登录验证。
  
## 实现
### 思路
  使用nginx并安装auth_request模块（1.12以后版本默认已经安装此模块），配合 [anvars](https://github.com/bogies/anvars)，来实现。
### 具体方法
以下为相关的三个服务
192.168.1.132:8080 为rbacview，实现登录验证
192.168.1.132:20000 为nginx代理
192.168.1.132:20001 为要保护的资源

  在 [rbacview](https://github.com/bogies/anvars/blob/develop/rbacview/src/main/java/org/bogies/tommy/controller/ServicesController.java)中的ServicesController.java中添加 /auth.jsd进行登录验证，从session中获取用户信息，成功则返回 HTTP STATUS 200，失败返回 HTTP STATUS 401。
  在 [rbacview](https://github.com/bogies/anvars/blob/develop/rbacview/src/main/webapp/login/login.jsp) 中实现 login.html 页面，用来做登录。
  
  配置 nginx 代理，添加配置如下：
  
  
  `
  
  server {
    listen 20000;
    server_name localhost;

    location = /auth {
        internal;
        proxy_pass http://192.168.1.132:8080/s/auth.jsd;
        proxy_pass_request_body     off;
        proxy_set_header X-Original-URI $request_uri;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    error_page 401 = @error401;
    location @error401 {
        # 设置登录成功后的跳转链接
        add_header Set-Cookie "$scheme://$http_host$request_uri";
        return 302 http://192.168.1.132:8080/login.html
    }
    location / {
        auth_request /auth;
        error_page 401 = @error401;
        proxy_pass http://192.168.1.132:20001;
    }
}
`

  这样，只要访问 `http://192.168.1.132:20000/` 时就会请求 /auth 进行验证，如果登录则代理至 `http://192.168.1.132:20001` ；否则会跳转到 login.html 页面
