---
layout: post
title: "Java中使用spring发送邮件"
date: 2017-08-11
excerpt: "QQ邮箱"
tags: [Java， Spring]
comments: true
---

## 前言

&nbsp; &nbsp; &nbsp;Javamail以及spring中使用JavaMail都会踩到许多坑，记录一下，省得下次再配一晚上。



## Mail

```java
package spittr.util;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.mail.SimpleMailMessage;
import org.springframework.mail.javamail.JavaMailSender;

public class Mail {
    @Autowired
    private JavaMailSender mailSender;
    public void send(SimpleMailMessage mail)
    {
        mailSender.send(mail);
    }
}
```

JavaMailSender是一个接口，定义了send方法，可以发送一条MailMessage。这里采用自动注入的方式为Mail类里注入一个JavaMailsender。



## xml的配置

```xml
 <bean id="mailSender" class="org.springframework.mail.javamail.JavaMailSenderImpl">
        <property name="host" value="${mail.host}"></property>
        <property name="port" value="${mail.port}"></property>
        <property name="javaMailProperties">
            <props>
                <prop key="mail.smtp.auth">true</prop>
                <prop key="mail.smtp.timeout">25000</prop>
                <prop key="mail.smtp.socketFactory.class">javax.net.ssl.SSLSocketFactory</prop>
                <prop key="mail.smtp.starttls.enable">true</prop>
            </props>
        </property>
        <property name="username" value="${mail.username}"></property>
        <property name="password" value="${mail.password}"></property>
    </bean>
```

在applicationContext.xml中（或者可以新建配置文件），加入这段代码。JavaMailSenderImpl是JavaMailSender的实现，可以在xml中为它配置一些属性。包括host、port等。这里要注意的是，如果要使用QQ邮箱，因为它的安全措施，要将mail.smtp.socketFactory.class设置为javax.net.ssl.SSLSocketFactory，mail.smtp.auth设置为true。



## properties的配置

```
mail.host = smtp.qq.com
mail.username = {username}
mail.password = ********
mail.port = 465
```

QQ邮箱的host是smtp.qq.com，smtp协议的默认端口是465，写好username和password即可。



## 测试

```java
public static void main(String[] args)
{
    ApplicationContext ac =
            new FileSystemXmlApplicationContext("src\\main\\webapp\\WEB-INF\\applicationContext.xml");
    Mail mail = (Mail)ac.getBean("mail");
    SimpleMailMessage message = new SimpleMailMessage();
    message.setTo("");
    message.setFrom("");
    message.setSubject("hello spring");
    message.setText("hello world");
    mail.send(message);
}
```

这里我原来用ClassPathXmlApplicationContext来加载配置文件，但一直失败，换成FileSystemXmlApplicationContext就行了不知道为什么orz。填入要发送的对象和自己的邮箱发送即可，可以看见发送成功。



## 注意事项

* xml配置中要开启安全验证，尤其是ssl
* 发送邮件的邮箱要开启smtp服务
* JavaMail默认端口是25，不一样的话要改
* QQ邮箱以及其它一些邮箱的第三方服务要求填写的密码是独立密码而不是原本的密码，独立密码要在邮箱里获取
* 发送者要和配置的邮箱一致