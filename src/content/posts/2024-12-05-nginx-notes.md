---
published: 2024-12-05
title: NGINX notes
---

NGINX is a configuration language. It runs by the priority of commands, not from top to bottom.

***

We can use the `rewrite` directive with the `last` parameter to tell NGINX to stop processing the current set of rewrite directives and start a new internal request with the rewritten URL.

```nginx
if ($allow != 1) {
    rewrite ^ /__auth_beta$uri last;
}
```

***

In this example, `auth_request` will run before `try_files`. Because `try_files` can accept another named `location`, we can use this trick to make a fallback location if commands prior to `try_files` do not rewrite the URL:

```nginx
location /__auth_beta {
    internal;
    auth_request /api/beta/verify-invitation-code;
    error_page 401 @coming_soon;
    try_files /index.html =404;
}

location @auth_success {
    set $allow 1;
    rewrite ^ $request_uri last;
}
```

***

Another useful trick is using the `map` directive to create variables based on conditions. This can be helpful for setting flags or modifying behavior based on request parameters:

```nginx
map $http_user_agent $is_bot {
    default 0;
    ~*bot 1;
}

server {
    if ($is_bot) {
        return 403;
    }
}
```

***

You can also use the `limit_req` module to rate limit requests. This is useful for preventing abuse and ensuring fair usage of your resources:

```nginx
http {
    limit_req_zone $binary_remote_addr zone=mylimit:10m rate=1r/s;

    server {
        location / {
            limit_req zone=mylimit burst=5 nodelay;
            proxy_pass http://backend;
        }
    }
}
```

***

Using the `geo` directive, you can create variables based on the client's IP address. This is useful for geo-blocking or serving different content based on the user's location:

```nginx
geo $country {
    default ZZ;
    127.0.0.1 US;
    192.168.1.0/24 RU;
}

server {
    if ($country = RU) {
        return 403;
    }
}
```

Another powerful feature is the `upstream` directive, which allows you to define a group of backend servers to distribute requests among them. This is useful for load balancing:

```nginx
upstream backend {
    server backend1.example.com;
    server backend2.example.com;
}

server {
    location / {
        proxy_pass http://backend;
    }
}
```

***

The `log_format` directive allows you to customize the log format. This can be useful for logging specific details about requests:

```nginx
log_format custom '$remote_addr - $remote_user [$time_local] "$request" '
                  '$status $body_bytes_sent "$http_referer" '
                  '"$http_user_agent" "$http_x_forwarded_for"';

server {
    access_log /var/log/nginx/access.log custom;
}
```

***

Using the `resolver` directive, you can specify DNS servers for NGINX to use when resolving domain names. This is useful for dynamic upstreams:

```nginx
http {
    resolver 8.8.8.8 8.8.4.4;
    
    server {
        location / {
            proxy_pass http://example.com;
        }
    }
}
```
