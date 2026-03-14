
#### 核心结论：**root 和 alias，是 Nginx 中唯二专门用于静态资源路径映射、指定静态资源根目录的核心指令**，没有其他同等级的核心指令可以替代，这道题考的是静态资源服务的核心配置，也是面试高频手写配置题的必考点。

我给你讲透两个指令的核心定位和区别，帮你彻底记牢：

1. **root 指令**：Nginx 静态资源服务的默认核心指令，作用是**路径拼接**，会把 location 匹配的 URL 路径，拼接到 root 指定的根目录后面，形成最终的文件查找路径。
    
    举个例子：
    
```nginx
    location /static/ {
        root /data/www; # 根目录指定
    }
```

用户访问`http://test.com/static/1.jpg`时，Nginx 会去服务器上找 `/data/www/static/1.jpg` 这个文件，也就是root路径 + location匹配的路径。
  
2. **alias 指令**：静态资源路径映射的补充核心指令，作用是**路径替换**，会把 location 匹配的 URL 路径，直接替换成 alias 指定的目录，不会拼接路径。
    
    举个例子：

    ```nginx
    location /static/ {
        alias /data/www/; # 路径替换
    }
    ```
    
    用户访问`http://test.com/static/1.jpg`时，Nginx 会直接去找`/data/www/1.jpg`这个文件，直接替换掉了匹配到的`/static/`路径。

补充误区：很多人会误以为 index 是根目录指令，其实不是 ——index 只是用来指定默认首页文件，不负责定义静态资源的根目录 / 路径映射，所以不能填这个。