---
layout post
title 关于Spring Boot项目打包及部署
categories maven idea
keywords jar maven
---

### maven打包的方式和部署到服务器的方式

1. 确保项目可以正常运行

2. 在idea的右侧有一栏maven栏，点击打开

   ![](\images\posts\idea\maven打包jar-1.jpg)

3. 点开之后，在lifecycle一栏中先清理一下

   ![](\images\posts\idea\maven打包jar-2.jpg)

4. 注意，清理时一定要看到build success才算清理成功

   ![](\images\posts\idea\maven打包jar-3.jpg)

5. 清理成功，开始install，注意，install完毕后一定也要看到build success才算成功

   ![](\images\posts\idea\maven打包jar-4.jpg)

   ![](\images\posts\idea\maven打包jar-5.jpg)

6. 接下来就前往src/target目录下面找到刚才打包的jar包，传到云服务器上，部署就成功了

   ![](\images\posts\idea\maven打包jar-6.jpg)

   ![](\博客记录\images\posts\idea\maven打包jar-7.jpg)

### 总结

1. 这个部署其实难度不高，但是我本人因为部署次数少，经常忘记一些内容
2. 我自己以前采用idea自带的artifacts打包方式来打包，但是这样打的包经常出现莫名其妙的问题
