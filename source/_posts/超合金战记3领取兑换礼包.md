---
title: 超合金战记3领取兑换礼包
date: 2022-05-10 18:42:58
---

一次抓包以及http请求拦截方式的案例



### <br>过程中使用到的工具： [Fiddler](http://xz.w10a.com/Small/fiddlerhhb.rar "HTTP协议调试代理工具")，[Postman](https://dl.pstmn.io/download/latest/win64 "API调试、HTTP请求的工具")，[Jpexs-decompiler](https://github.com/jindrapetrik/jpexs-decompiler/releases/download/version15.1.0/ffdec_15.1.0_setup.exe "SWF反编译调试工具")，[NodeJS](https://cdn.npmmirror.com/binaries/node/v16.15.0/node-v16.15.0-x64.msi "JavaScript 运行环境")</br>

![完成效果](https://s1.ax1x.com/2022/05/10/ONYOde.jpg)
<!-- more -->


### <br>HTTP请求分析</br>
打开fiddler，然后我们对游戏进行一次兑换操作。这样即可在fiddler左侧列表得到游戏对HTTP的请求操作
![](https://s1.ax1x.com/2022/05/10/ONdEn0.png)
**而我们所需要的便是对最后一行HTTP请求进行分析:**
```text
POST http://my.4399.com/jifen/activation HTTP/1.1
Host: my.4399.com
Connection: keep-alive
Content-Length: 76
User-Agent: Mozilla/5.0 (Windows NT 10.0; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/86.0.4240.198 Safari/537.36
Content-Type: application/x-www-form-urlencoded
Accept: */*
Origin: http://sbai.4399.com
X-Requested-With: ShockwaveFlash/32.0.0.465
Referer: http://sbai.4399.com/4399swf/upload_swf/ftp14/cwb/20140214/chj/etjv1130.swf/[[DYNAMIC]]/4
Accept-Encoding: gzip, deflate
Accept-Language: zh-CN,zh;q=0.9
Cookie: ***
```
从抓到的数据中，我们可以轻松的得出这是一个使用HTTP/1.1协议的 **POST** 请求。
确定了请求方式以后，我们再打开PostMan模拟一次POST请求，来观察得到的数据。
![](https://s1.ax1x.com/2022/05/10/ONwvFS.png)
**对数据进行转码后我们即可得到一串JSON：**

```json
{"code":101,"result":[],"msg":"参数错误"}
```
### <br>SWF逆向分析</br>
在得到这串数据后，我们的逆向分析操作便可以开始了。对游戏进行抓包得到SWF的方法就不过多赘述，我们直接进行逆向分析。

**回想第一步操作，我们在对游戏进行兑换时，游戏进行了对话框提示，内容为 <font color=Red>参数错误！无法兑换!</font> <br>既然有了字符串，那先不管这串字符在SWF中是否被加密，先对其进行搜索。** 
果不其然，在搜索到的唯一一项结果中，可以很明显的判断出这就是兑换礼包的AS代码
![](https://s1.ax1x.com/2022/05/10/ONs0gg.png)
```ActionScript
   var code0:int = JSON2.decode(data0).code;
         var result0:int = int(JSON2.decode(data0).result);
         var tipstr:String = "";
         Game.testText.addTestText("当前结果：" + result0);
         switch(code0)
         {
            case 99:
               tipstr = "未知错误！无法兑换！";
               break;
            case 100:
               if(result0 == 132)
               {
                  tipstr = "兑换成功！你获得了新手礼包！";
                  this.getGift();
               }
               else if(result0 == 330)
               {
                  tipstr = "兑换成功！你获得了新春礼包！";
                  this.getGift2();
               }
               else if(result0 == 164)
               {
                  tipstr = "兑换成功！你获得了周年礼包！";
                  this.getGift3();
               }
               else if(result0 == 209)
               {
                  tipstr = "兑换成功！你获得了双节礼包！";
                  this.getGift4();
               }
               break;
            case 101:
               tipstr = "参数错误！无法兑换！";
               break;
            case 102:
               tipstr = "兑换码不存在！无法兑换！";
               break;
            case 103:
               tipstr = "兑换码还没被兑换！";
               break;
            case 104:
               tipstr = "兑换码被使用过了！";
               break;
            case 105:
               tipstr = "兑换码只能被领取者使用！";
               break;
            case 106:
               tipstr = "该礼包已经兑换过了！";
               break;
            case 107:
               tipstr = "验证码失效！无法兑换！";
               break;
            case 108:
               tipstr = "兑换码失效！无法兑换！";
               break;
            case 109:
               tipstr = "激活失败！无法兑换！";
               break;
            case 110:
               tipstr = "您的账号今天已经使用过兑换码了，不能再使用了！";
         }
```
**结合之前我们POST请求得到的数据，我们不难看出，JSON数据中<font color=Red>code</font>的值对应着上文代码的<font color=Red>code0</font>变量，也就是说我们可以通过某种方法将JSON数据中的<font color=Red>code</font>项返回值修改为上文代码中礼包兑换成功的<font color=Red>100</font>这一返回值**
要想实现我们这一设想，那就需要使用Fiddler配合NodeJS来操作:
* NodeJS负责搭建一个可以进行POST请求的本地服务器，通过它，我们可以模拟出兑换成功的JSON数据。
* Fiddler中的自动响应这一功能，便可以通过拦截HTTP请求，将其转发到指定的链接，从而实现对客户端的欺骗。

### <br>编写本地服务器</br>

![](https://s1.ax1x.com/2022/05/10/ONcNU1.png)
<center><font color=Gray size= 3px>代码仅供参考，请勿用于非法用途</font></center>

**<br>code值对应上文代码中的code0变量
result值则对应着上文代码中的result0变量</br>**

### <br>启动NodeJS服务</br>
```bash
npm install express #安装express模块
node test.js #启动服务
```
![](https://s1.ax1x.com/2022/05/10/ON2ytI.png)

服务启动完成后，我们便可以通过PostMan测试本地服务器效果了
![](https://s1.ax1x.com/2022/05/10/ON2LcT.png)
JSON完美返回，但由于Fiddler无法识别127.0.0.1、localhost这类地址，所以我们需要修改hosts，填入任意域名，使它指向127.0.0.1或localhost
我将127.0.0.1指向了www.flowerzzz.top，那么在我这台计算机上，访问www.flowerzzz.top将会被转发到127.0.0.1
![](https://s1.ax1x.com/2022/05/10/ONWQo9.png)
PostMan测试成功
![](https://s1.ax1x.com/2022/05/10/ONWySP.png)

### <br>拦截HTTP请求</br>
在自动响应栏添加规则，写入规则，最后记得勾选顶部**启动规则**，自动响应图标将呈现**绿色**，若是灰色便是没有启动规则
![](https://s1.ax1x.com/2022/05/10/ONf9l6.png)

最后登录游戏，兑换就可以看到效果了。

本地服务器填写的JSON数据中，result的数字便代表了将会领取的礼包 
**132代表新手礼包
330代表新春礼包
164代表周年礼包
209代表双节礼包**


## 仅供学习交流，严禁用于商业用途，请于24小时内删除
## 仅供学习交流，严禁用于商业用途，请于24小时内删除
## 仅供学习交流，严禁用于商业用途，请于24小时内删除