---

layout: post
title: openstack horizon不支持根据实例IP地址进行搜索的问题

---

**问题：**

我用的是openstack juno版本，以admin用户登陆dashboard，点击管理员panel group,再点击实例panel，里面有一些搜索filter可选，其中根据ipv4,ipv6,项目名这三个filter进行搜索只会从当前显示的页面中搜索，实际使用过程中非常难用。
![](http://ww3.sinaimg.cn/mw690/5d326ffdgw1f0e5i1wesdj21kw0mbdmx.jpg)

**问题分析：**
