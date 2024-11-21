# 原作者:[不良林](https://youtu.be/X7CC5jrgazo?si=Ailu8FUGGkwuAMqO)

# 项目地址:

- https://github.com/yixuan-ovo/CF-Worker

# CloudFlare:

- https://cloudflare.com
---
# CLoudFlare

## 注册并登录
选择免费版即可,够用

## 创建workers
点击

![alt text](./img/1.png)

创建一个新的workers

![alt text](./img/2.png)

![alt text](./img/3.png)

随便定义一个名字即可,随后点击部署

![alt text](./img/4.png)

返回概述点击创建的workers进入

进入设置,添加一个新的变量和机密

![alt text](./img/5.png)

名称BUCKEND,值为任意的订阅转换链接,搭建自己的订阅转换服务参考[这篇](https://github.com/yixuan-ovo/ImmortalWrt-Files/blob/2ba744524f589ff5937f81bdf6a9cac76c1bcbc9/OpenClash/openclash-tutorials/dockerAnalysis.md)

设置完后右上角点击编辑代码,原有示例代码全部删除

![alt text](./img/6.png)

**将[项目地址](#项目地址)下的worker.js下所有代码复制并部署**

此时可以点击访问,但并未配置环境变量,所以会报错

## 创建KV
点击KV,创建一个命名空间

![alt text](./img/7.png)

免费版本即可,一月1000次请求完全够用,随便命名创建完成

随后返回workers的设置下,选择绑定项,点击添加

![alt text](./img/8.png)

添加资源绑定选择KV命名空间

![alt text](./img/9.png)

### **变量名称必须输入SUB_BUCKET**

完成绑定后选择部署,此时点击右上角访问即可正常进入链接

链接使用见[这里](https://github.com/yixuan-ovo/ImmortalWrt-Files/blob/2ba744524f589ff5937f81bdf6a9cac76c1bcbc9/OpenClash/openclash-tutorials/%E4%B8%80%E4%B8%AA%E9%93%BE%E6%8E%A5%E5%90%8C%E6%97%B6%E5%AE%9E%E7%8E%B0%E9%85%8D%E7%BD%AE%E6%A8%A1%E6%9D%BF%E5%92%8C%E5%90%8E%E7%AB%AF%E8%AE%A2%E9%98%85%E8%BD%AC%E6%8D%A2.md)

## *也可选择创建R2对象存储,但需要绑定信用卡*