---
layout: post
title: GPG-QuickStart
date: 2022-09-11 16:45:30.000000000 +09:00
tag: web3
---

## 0x01 什么是 GPG
要了解什么是 GPG，就要先了解 PGP。1991年，程序员 Phil Zimmermann 为了避开政府监视，开发了加密软件 PGP。
这个软件非常好用，迅速流传开来，成了许多程序员的必备工具。但是，它是商业软件，不能自由使用。
所以，自由软件基金会决定，开发一个 PGP 的替代品，取名为 GnuPG。这就是 GPG 的由来。

GPG有许多用途，本文主要介绍文件加密。至于邮件的加密，不同的邮件客户端有不同的设置，请参考 0x05 ubuntu 官网链接。

## 0x02 生成密钥
{% highlight ruby %}
$ gpg --gen-key
按照提示填写就行
{% endhighlight %}

## 0x03 密钥管理
{% highlight ruby %}
1 列出密钥列表
$ gpg --list-keys
输出：
pub   ed25519 2022-10-09 [SC] [有效至：2024-10-08]
      4C530E24AEFCFAFC96BDFFA374FC318908554B6F
uid             [ 绝对 ] alice
sub   cv25519 2022-10-09 [E] [有效至：2024-10-08]

2 删除密钥
$ gpg --delete-keys alice

3 导出公钥
$ gpg --armor --output public-key.txt --export [用户ID]

4 导出私钥 (需要输入密码)
$ gpg --armor --output private-key.txt --export-secret-keys [用户ID]
{% endhighlight %}

## 0x04 加密和解密
{% highlight ruby %}
1 加密文件
$ gpg --recipient [用户ID] --output [输出文件路径] --encrypt [要加密的文件路径]

2 解密
$ gpg --recipient [用户ID] --output [输出文件路径] --decrypt [要解密的文件路径]

{% endhighlight %}

## 0x05 参考链接

{% highlight ruby %}
https://www.ruanyifeng.com/blog/2013/07/gpg.html

https://help.ubuntu.com/community/GnuPrivacyGuardHowto#Reading_OpenPGP_E-mail

https://futureboy.us/pgp.html#gpgorgpg2
https://www.gnupg.org/gph/en/manual.html#AEN26
https://www.madboa.com/geek/gpg-quickstart/
{% endhighlight %}
