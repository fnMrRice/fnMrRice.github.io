+++
date = '2026-02-10T15:58:32+08:00'
title = 'Hugo部署记录'
+++

手头上有个已备案域名，得利用起来，于是网站诞生了  

虽然我有一个小鸡，但是想了想还是用 Github Pages 来做比较方便，万一小鸡到期忘记续费这个就好弄多了  

国内 Github Pages 访问速度还是挺慢的，正好 CF 那边有免费 CDN，直接白嫖  

## 1. 准备域名

先上大善人 CloudFlare 网站注册一个账号，登录后进入主页，在左侧栏中选择域，然后点击加入域，添加自己的域名

![/images/hello-world/cf-domain-1.png](/images/hello-world/cf-domain-1.png)

此时，CF会检测该域是否已经被其他用户占用，如果有的话，那就只能申诉了  

如果没有，CF会要求修改DNS记录的NS为他们的服务器  

我的域名是在腾讯云买的，所以就去DNSPod修改NS服务  

进入 `腾讯云` - `控制台` - `域名注册` - `我的域名`，选择要套CF的域名，点进去就能看到基本信息  

![alt text](/images/hello-world/dnspod-domain-1.png)

基本信息下方有一个DNS解析，在这里修改NS服务器  

改完之后，腾讯云的控制台会各种报警告，直接不管就行  

再次回到CF网站，等几分钟后这里应该就能正常获取到NS服务变更，此时就成功套上CF盾了  

### 1.1 已有服务HTTPS管理

如果域名是个之前没有解析到某个服务，那可以不用看这一节  

#### 1.1.1 生成证书

CF默认配置的SSL/TLS加密模式是完全，当原有服务没有配置HTTP时，访问网站会报错。此时需要给原网站配置HTTPS证书。 

先进入CF管理后台，然后选择 `SSL/TLS` - `源服务器`，点击`创建证书`。  

![alt text](/images/hello-world/cf-ssl.png)

这里推荐自行生成密钥对，然后创建CSR交给CF，拿到证书后丢给nginx来做https转发。不过为了方便，我选择使用CF生成的私钥，虽然不能完全保证CF不记录私钥，不过想想也没什么太隐私的数据，也无所谓了。  

#### 1.1.2 导入证书

我用的是nginx，服务器上部署了一个用来打洞的 zerotier planet 节点和HTTP服务，所以下面以nginx为例  

下载好证书后，把私钥和证书丢到服务器上的 `/etc/nginx/certs/cloudflare` 中  
然后修改 `/etc/nginx/sites-available/zerotier.example.com.conf`  
这里需要把 `example.com` 改为自己的域名  

```nginx
server {
    listen 443 ssl;
    server_name zerotier.example.com;
    ssl_certificate /etc/nginx/certs/cloudflare/example.com.pem;
    ssl_certificate_key /etc/nginx/certs/cloudflare/example.com.key;
    ssl_session_timeout 5m;
    ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:ECDHE:ECDH:AES:HIGH:!NULL:!aNULL:!MD5:!ADH:!RC4;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers on;

    location / {
        proxy_pass http://127.0.0.1:4000;
    }
}

server {
    if ($host = zerotier.example.com) {
        return 301 https://$host$request_uri;
    }


    listen 80;
    server_name zerotier.example.com;
    return 301 https://$host$request_uri;
}
```

这个配置还顺便做了把80端口的http请求转到https上  

```bash
# 启用配置
cd /etc/nginx/sites-enabled
sudo ln -s /etc/nginx/sites-available/zerotier.example.com.conf
# 测试配置
sudo nginx -t
# 重启nginx
sudo systemctl reload nginx
```

完事之后，cf那边就不会报错了，甚至还能把加密模式改为严格

## 2. 创建Github Pages

按文档来 [](https://docs.github.com/zh/pages/quickstart)  
一步一步按照文档走就行了  

## 3. 部署Hugo  

### 3.1 克隆仓库  

如果上一步一切正常，这里应该就能拿到一个包含了 README 的仓库。把仓库clone下来

```bash
git clone https://github.com/fnMrRice/fnMrRice.github.io.git
```

### 3.2 安装hugo

hugo安装貌似需要nodejs，不过我这里正好有msys环境，直接装一个hugo 

```bash
cd fnMrRice.github.io.git
pacman -Ss hugo # 搜到hugo的包名
pacman -S mingw-w64-ucrt-x86_64-hugo # 安装hugo
# 初始化hugo
hugo new site demo-site
# 把网站弄到根目录
mv demo-site/* ./
rm -rf ./demo-site
```

### 3.3 安装主题

这个网站是类似于博客的样式，我在[hugo网站](https://themes.gohugo.io/)上面挑了一会，选了 `diary` [这个主题](https://github.com/AmazingRise/hugo-theme-diary.git)  

```bash
git submodule add https://github.com/AmazingRise/hugo-theme-diary.git themes/diary
echo "theme = 'diary'" >> hugo.toml
```

### 3.4 本地测试

```bash
hugo new content content/posts/my-first-post.md
hugo server -D # 编译草稿
```

### 3.5 Github Pages部署

1. 进入Github的仓库界面
2. 选择上方的Actions
3. 创建一个新的Action
4. 搜索Hugo并配置
5. 进入 设置 - Code and automation - Pages - Build and deployment
6. Source选择Github Actions
7. Custom domain中输入自己的域名
8. 完事！接下来当提交文章的时候，hugo就会自己部署了
