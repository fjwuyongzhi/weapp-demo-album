# 微信小程序示例 - 小相册

小相册是结合腾讯云[对象存储服务](https://www.qcloud.com/product/cos.html)（Cloud Object Service，简称COS）制作的一个微信小程序示例。在代码结构上包含如下两部分：

- `app`: 小相册应用包代码，可直接在微信开发者工具中作为项目打开
- `server`: 搭建的Node服务端代码，作为服务器和`app`通信，提供 CGI 接口示例用于拉取 COS 图片资源、上传图片到 COS、删除 COS 图片

小相册主要功能如下：
 * 列出 COS 服务器中的图片列表
 * 点击左上角**上传图片**图标，可以调用相机拍照或从手机相册选择图片，并将选中的图片上传到 COS 服务器中
 * 轻按任意图片，可进入全屏图片预览模式，并可左右滑动切换预览图片
 * 长按任意图片，可将其保存到本地，或从 COS 中删除

![运行截图](screenshot.png)


## 部署和运行

拿到了本小程序源码的朋友可以尝试自己运行起来。

### 整体架构

![整体架构](http://easyimage-10028115.file.myqcloud.com/internal/t3n2rrlm.ulr.jpg)

### 1. 准备域名和证书

在微信小程序中，所有的网路请求受到严格限制，不满足条件的域名和协议无法请求，具体包括：

* 只允许和在 MP 中配置好的域名进行通信，如果还没有域名，需要注册一个。
* 网络请求必须走 HTTPS 协议，所以你还需要为你的域名申请一个 SSL 证书。

> 腾讯云提供[域名注册](https://www.qcloud.com/product/dm.html)和[证书申请](https://console.qcloud.com/ssl)服务，还没有域名或者证书的可以去使用

域名注册好之后，可以登录[微信公众平台](https://mp.weixin.qq.com)配置通信域名了。

![配置通信域名](http://easyimage-10028115.file.myqcloud.com/internal/tjzpgjrz.y5a.jpg)

### 2. Nginx 和 Node 代码部署

小相册服务要运行，需要进行以下几步：

* 部署 Nginx，Nginx 的安装和部署请大家自行搜索（注意需要把 SSL 模块也编译进去）
* 配置 Nginx 反向代理到 `http://127.0.0.1:9993`
* Node 运行环境，可以安装 [Node V6.6.0](https://nodejs.org/)
* 部署 `server` 目录的代码到服务器上，如 `/data/release/qcloud-applet-album`
* 使用 `npm install` 安装依赖模块
* 使用 `npm install pm2 -g` 安装 pm2

> 上述环境配置比较麻烦，剪刀石头布的服务器运行代码和配置已经打包成[腾讯云 CVM 镜像](https://buy.qcloud.com/cvm?marketImgId=370)，推荐大家直接使用。
> * 镜像部署完成之后，云主机上就有运行 WebSocket 服务的基本环境、代码和配置了。
> * 腾讯云用户可以[免费领取礼包](https://www.qcloud.com/act/event/yingyonghao.html#section-voucher)，体验腾讯云小程序解决方案。
> * 镜像已包含「剪刀石头布」和「小相册」两个小程序的服务器环境与代码，需要体验两个小程序的朋友无需重复部署

### 3. 配置 HTTPS

镜像中已经部署了 nginx，需要在 `/etc/nginx/conf.d` 下修改配置中的域名、证书、私钥。

![证书 Nginx 配置](http://easyimage-10028115.file.myqcloud.com/internal/agfty0fn.gfi.jpg)


配置完成后，即可启动 nginx。

```sh
nginx
```

### 4. 域名解析

我们还需要添加域名记录解析到我们的云服务器上，这样才可以使用域名进行 HTTPS 服务。

在腾讯云注册的域名，可以直接使用[云解析控制台](https://console.qcloud.com/cns/domains)来添加主机记录，直接选择上面购买的 CVM。

![添加域名解析](http://easyimage-10028115.file.myqcloud.com/internal/uw25hdj2.k1u.jpg)

解析生效后，我们在浏览器使用域名就可以进行 HTTPS 访问。

![HTTPS 访问效果图](http://easyimage-10028115.file.myqcloud.com/internal/bxfkmjea.g41.jpg)

### 5. 开通和配置 COS

小相册示例的图片资源是存储在 COS 上的，要使用 COS 服务，需要登录 [COS 管理控制台](https://console.qcloud.com/cos/overview)，然后在其中完成以下操作：

- 开通 COS 服务分配得到唯一的`APP ID`
- 使用密钥管理生成一对`SecretID`和`SecretKey`（用于调用 COS API）
- 在 Bucket 列表中创建**公有读私有写**访问权限、**CDN加速**的 bucket（存储图片的目标容器）
- 在创建的 bucket 容器中创建文件夹，命名为`photos`

### 6. 启动小相册示例 Node 服务

在镜像中，小相册示例的 Node 服务代码已部署在目录 `/data/release/qcloud-applet-album` 下：

进入该目录：

```bash
cd /data/release/qcloud-applet-album
```

在该目录下有个名为`config.js`的配置文件（如下所示），按注释修改对应的 COS 配置：

```js
module.exports = {
    // Node 监听的端口号
    port: '9993',
    ROUTE_BASE_PATH: '/applet',

    cosAppId: '填写开通 COS 时分配的 APP ID',
    cosSecretId: '填写密钥 SecretID',
    cosSecretKey: '填写密钥 SecretKey',
    cosFileBucket: '填写创建的公有读私有写的bucket名称',
};
```

代码运行需要临时目录，在部署目录下创建一个临时目录 `tmp`。

```sh
mkdir tmp
```

小相册示例使用`pm2`管理 Node 进程，执行以下命令启动 node 服务：

```bash
pm2 start process.json
```

### 7. 微信小程序服务器配置

进入微信公众平台管理后台设置服务器配置，配置类似如下设置：

![小程序服务器配置](http://easyimage-10028115.file.myqcloud.com/internal/erz2fmyd.bx1.jpg)

注意：需要将 `www.qcloud.la` 设置为上面申请的域名，将 downloadFile 合法域名设置为在 COS 管理控制台中自己创建的 bucket 的相应 **CDN 加速访问地址**，如下图所示：

![CDN加速访问地址](http://easyimage-10028115.file.myqcloud.com/internal/1criixcb.wat.jpg)

### 8. 启动小相册 Demo

在微信开发者工具将小相册应用包源码添加为项目，并把源文件`config.js`中的通讯域名修改成上面申请的域名。

![修改配置文件](http://easyimage-10028115.file.myqcloud.com/internal/nvqrghyi.pl4.jpg)

然后点击调试即可打开小相册Demo开始体验。

![调试](http://easyimage-10028115.file.myqcloud.com/internal/uo4m5gcd.tpr.jpg)

这里有个问题。截止目前为止，微信小程序提供的上传和下载 API 无法在调试工具中正常工作，需要用手机微信扫码预览体验。

## 主要功能实现

### 上传图片

上传图片使用了微信小程序提供的`wx.chooseImage(OBJECT)`获取需要上传的文件路径，然后调用上传文件接口`wx.request(OBJECT)`发送 HTTPS POST 请求到自己指定的后台服务器。和传统表单文件上传一样，请求头`Content-Type`也是`multipart/form-data`。后台服务器收到请求后，使用 npm 模块 multiparty 解析 `multipart/form-data` 请求，将解析后的数据保存为指定目录下的临时文件。拿到临时文件的路径后，就可直接调用 COS SDK 提供的[文件上传 API](https://www.qcloud.com/doc/product/430/5947#.E6.96.87.E4.BB.B6.E4.B8.8A.E4.BC.A0) 进行图片存储，最后得到图片的存储路径及访问地址（存储的图片路径也可以直接在 COS 管理控制台看到）。

### 获取图片列表

调用[列举目录下文件&目录 API](https://www.qcloud.com/doc/product/430/5947#.E5.88.97.E4.B8.BE.E7.9B.AE.E5.BD.95.E4.B8.8B.E6.96.87.E4.BB.B6.26amp.3B.E7.9B.AE.E5.BD.95)可以获取到在 COS 服务端指定 bucket 和该 bucket 指定路径下存储的图片。

### 下载和保存图片

指定图片的访问地址，然后调用微信小程序提供的 `wx.downloadFile(OBJECT)` 和 `wx.saveFile(OBJECT)` 接口可以直接将图片下载和保存本地。这里要注意图片访问地址的域名需要和服务器配置的 dowmloadFile 合法域名一致，否则无法下载。

### 删除图片

删除图片也十分简单，直接调用[文件删除 API](https://www.qcloud.com/doc/product/430/5947#.E6.96.87.E4.BB.B6.E5.88.A0.E9.99.A4) 就可以将存储在 COS 服务端的图片删除。

## LICENSE

[MIT](LICENSE)