---
layout:     post
title:      macOS 跟 Ubuntu 之间实现便捷的文件传输
subtitle:   百度网盘？还是远程服务器靠谱！
date:       2017-04-05
author:     钟瞻忠
catalog: true
tags:
    - 折腾
---

>  
  记得之前使用 Windows 系统时，只要搭配 SecureCRT 就可以非常方便的跟服务器进行文件传输。这两年换上 Mac 之后，一直希望实现这个功能，期间也试过很多次，但是一直都配置失败。典型的表现就是使用 sz 命令之后，iTerm2 往往会卡住，传输失败。  
  忍受了这么久之后，今天又有传输需求，使用几次 scp 命令之后，实在是受不了那么繁琐的输入，就又来折腾 lrzsz 的相关配置，终于得以成功。于是有了此文。

<h2>前言</h2>
  首先，得先安装好 iTerm2，安装过程不再赘述，有需要的自己搜索相关文章。在安装好 iTerm2 之后，需要下载 lrzsz。需要注意的是，服务器以及 macOS 中都需要安装好 lrzsz ，否则执行命令会失败。
  
<h2>下载 autossh</h2>
  之前一直是在 iTerm2 中使用 expect 的方式进入服务器，而今天我也终于知道，无法使用 lrzsz 的方式跟服务器进行文件传输也是因为 expect 的方式本身就无法实现，至于为什么我也不太了解。  
  所以实现这种方式需要使用另外发现的一个工具去实现，我用的是 autossh ，当然也可以使用其他方式来实现。autossh 是一个 <a href="https://github.com/islenbo/autossh">github</a> 上的开源项目，进入项目后，下载 release 中的 mac 专用压缩文件。  
  下载后压缩到本地，可以看到有两个文件，一个是 autossh shell 文件，一个是 config.json 配置文件。这时打开 config.ssh ，更改里面的文件为如下格式：

```json
{
  "show_detail": true,
  "options": {
    "ServerAliveInterval": 30
  },
  "servers": [
    {
      "name": "vps",
      "ip": "88.88.88.88",  #填入服务器 ip
      "port": 20,  #服务器端口
      "user": "root",  #以什么角色登录
      "password": "lanjun_vps", #该角色对应的密码
      "method": "password",  #不用动
      "key": "",
      "options": {
        "ServerAliveInterval": 30
      }
    } #如果有多个服务器，可以直接在后面接逗号，然后跟上面一样进行配置
  ]
}
```


  此时 autossh 这边的工作已经完成了，接下来进入到 iTerm2 的配置。
  

<h2>iTerm2 配置</h2>
  1，进入 iTerm2 的 Preferences，也就是设置，点击 Profiles，增加一个 Profile ，如下图：
    
<a href="https://i.loli.net/2019/03/01/5c78f934468ae.png"><img src="https://i.loli.net/2019/03/01/5c78f934468ae.png" alt="" /></a>  
  2，去 <a href="https://github.com/mmastrac/iterm2-zmodem">github</a> 下载 zmodem 脚本文件，下载好之后，放置到自己喜欢的位置，如<code>/usr/local/bin</code>；  
  3，进入 Triggers，<code>注意，这里一定要选中 autossh 所在的那个 Profile</code>，如图：  
<a href="https://i.loli.net/2019/03/01/5c78faa5ea3b0.png"><img src="https://i.loli.net/2019/03/01/5c78faa5ea3b0.png" alt="" /></a>  
  4，按照如下配置 Triggers：

配置 sz
```java
Regular expression: rz waiting to receive.\*\*B0100
Action: Run Silent Coprocess
Parameters: /usr/local/bin/iterm2-send-zmodem.sh
```
配置 rz
```
Regular expression: \*\*B00000000000000
Action: Run Silent Coprocess
Parameters: /usr/local/bin/iterm2-recv-zmodem.sh
```



<a href="https://i.loli.net/2019/03/01/5c78fd6c8cb0d.png"><img src="https://i.loli.net/2019/03/01/5c78fd6c8cb0d.png" alt="" /></a>
    

<h2>Over</h2>
  配置之后，在打开 iTerm2 的前提下，即可以使用 command+shift+h 的方式去打开一个新窗口然后连接 ssh 了。重要的是，这样配置后，sz/rz 命令都可以正常使用。



>
  本文作者：ZhanZhong</br>  
  版权声明：本博客所有文章均采用 <a href="https://creativecommons.org/licenses/by-nc-sa/3.0/">CC BY-NC-SA 3.0</a>  许可协议，如需转载，请注明出处！
