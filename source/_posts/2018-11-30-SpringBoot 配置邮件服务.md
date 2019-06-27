---
title: SpringBoot 配置邮件服务
date: 2018-11-30 15:00:00
updated: 2018-11-30 15:00:00
tags: JavaMail
categories: Java
---
# 1. 有关 SpringBoot 邮件服务
Spring Framework 自己有一套基于 `JavaMail` 的邮件服务包 `org.springframework.mail`，并通过 JavaMailSender 接口提供了一种简易的发送邮件的方式。这样，开发人员就可以不用操心底层的邮件系统，使用 Spring 提供的接口即可方便地使用邮件服务。官方文档：[https://docs.spring.io/spring/docs/5.0.10.RELEASE/spring-framework-reference/integration.html#mail](https://docs.spring.io/spring/docs/5.0.10.RELEASE/spring-framework-reference/integration.html#mail)

而在 SpringBoot 中，提供了更为简便的自动配置，又进一步简化了开发人员工作。官方文档：[https://docs.spring.io/spring-boot/docs/2.0.6.RELEASE/reference/htmlsingle/#boot-features-email](https://docs.spring.io/spring-boot/docs/2.0.6.RELEASE/reference/htmlsingle/#boot-features-email) 。下面介绍一下 SpringBoot 中配置邮件服务的具体方法。

# 2. 具体操作
## 2.1 添加 Maven 依赖
想要使用 SpringBoot 的邮件服务，需要依赖 `spring-boot-starter-mail`。所以在 pom 中添加依赖。
```xml
<dependency>
   <groupId>org.springframework.boot</groupId>
   <artifactId>spring-boot-starter-mail</artifactId>
</dependency>
```
## 2.2 配置 yml
```yml
spring:
  mail:
    # 邮件服务地址
    host: smtp.163.com
    # 端口
    port: 25
    # 编码格式
    default-encoding: utf-8
    # 用户名
    username: xxx@163.com
    # 授权码
    password: xxx
    # 其它参数
    properties:
      mail:
        smtp:
          # 如果是用 SSL 方式，需要配置如下属性
          starttls:
            enable: true
            required: true
```
- host: 为邮件服务的地址。一般在邮箱设置中可以看到。
- port: 端口号。不同的服务商端口号可能不同，这个也需要自行查看。
- username: 为登录邮箱的用户名。
- password: 这个不是邮箱登录密码，而是第三方授权码。在邮箱设置中或者帮助文档中会有介绍，需要开通第三方授权。
- properties: 其它参数。如果需要使用 SSL 方式，需要配置 `mail.smtp.starttls.enable` 为 true。有关详细的参数说明见文档[https://javaee.github.io/javamail/#Development_Releases](https://javaee.github.io/javamail/#Development_Releases)

有关 SpringBoot 中 auto-configuration 的参数见源码：[MailProperties.java](https://github.com/spring-projects/spring-boot/blob/v2.0.6.RELEASE/spring-boot-project/spring-boot-autoconfigure/src/main/java/org/springframework/boot/autoconfigure/mail/MailProperties.java)
## 2.3 编写邮件服务类
MailService.java
```java
@Service
public class MailService {
    private static final Logger logger = LoggerFactory.getLogger(MailServiceImpl.class);

    @Autowired
    private JavaMailSender mailSender;

    private static final String SENDER = "xxx@163.com";

    /**
     * 发送普通邮件
     *
     * @param to      收件人
     * @param subject 主题
     * @param content 内容
     */
    @Override
    public void sendSimpleMailMessge(String to, String subject, String content) {
        SimpleMailMessage message = new SimpleMailMessage();
        message.setFrom(SENDER);
        message.setTo(to);
        message.setSubject(subject);
        message.setText(content);
        try {
            mailSender.send(message);
        } catch (Exception e) {
            logger.error("发送简单邮件时发生异常!", e);
        }
    }

    /**
     * 发送 HTML 邮件
     *
     * @param to      收件人
     * @param subject 主题
     * @param content 内容
     */
    @Override
    public void sendMimeMessge(String to, String subject, String content) {
        MimeMessage message = mailSender.createMimeMessage();
        try {
            //true表示需要创建一个multipart message
            MimeMessageHelper helper = new MimeMessageHelper(message, true);
            helper.setFrom(SENDER);
            helper.setTo(to);
            helper.setSubject(subject);
            helper.setText(content, true);
            mailSender.send(message);
        } catch (MessagingException e) {
            logger.error("发送MimeMessge时发生异常！", e);
        }
    }

    /**
     * 发送带附件的邮件
     *
     * @param to       收件人
     * @param subject  主题
     * @param content  内容
     * @param filePath 附件路径
     */
    @Override
    public void sendMimeMessge(String to, String subject, String content, String filePath) {
        MimeMessage message = mailSender.createMimeMessage();
        try {
            //true表示需要创建一个multipart message
            MimeMessageHelper helper = new MimeMessageHelper(message, true);
            helper.setFrom(SENDER);
            helper.setTo(to);
            helper.setSubject(subject);
            helper.setText(content, true);

            FileSystemResource file = new FileSystemResource(new File(filePath));
            String fileName = file.getFilename();
            helper.addAttachment(fileName, file);

            mailSender.send(message);
        } catch (MessagingException e) {
            logger.error("发送带附件的MimeMessge时发生异常！", e);
        }
    }

    /**
     * 发送带静态文件的邮件
     *
     * @param to       收件人
     * @param subject  主题
     * @param content  内容
     * @param rscIdMap 需要替换的静态文件
     */
    @Override
    public void sendMimeMessge(String to, String subject, String content, Map<String, String> rscIdMap) {
        MimeMessage message = mailSender.createMimeMessage();
        try {
            //true表示需要创建一个multipart message
            MimeMessageHelper helper = new MimeMessageHelper(message, true);
            helper.setFrom(SENDER);
            helper.setTo(to);
            helper.setSubject(subject);
            helper.setText(content, true);

            for (Map.Entry<String, String> entry : rscIdMap.entrySet()) {
                FileSystemResource file = new FileSystemResource(new File(entry.getValue()));
                helper.addInline(entry.getKey(), file);
            }
            mailSender.send(message);
        } catch (MessagingException e) {
            logger.error("发送带静态文件的MimeMessge时发生异常！", e);
        }
    }
}
```

`@Autowired` 自动注入 JavaMailSender 类，这个类已经自动被 Spring 容器管理了。其中的属性配置即为 yml 中的配置，SpringBoot 已经在项目启动的时候帮我们 auto-configuration 了。

这里我封装了4种发送邮件的方法。
- sendSimpleMailMessge: 发送普通邮件。使用 SimpleMailMessage 类封装邮件发送者、接受者、主题和内容。
- sendMimeMessge(发送 html 邮件): MimeMessage 可以用来发送更复杂的邮件。这里可以发送 HTML 格式的邮件。
- sendMimeMessge(发送带有附件的邮件): 传入文件绝对路径，可以在将文件以附件的形式发送。
- sendMimeMessge(发送带有静态文件的邮件): 如果邮件内容中需要插入图片，可以使用 `helper.addInline` 方法将 html 文件中，资源属性的 cid 替换成对应的文件。详见第3章中测试用例。

# 3. 通过测试用例测试效果
SpringbooEmailDemoApplicationTests.java

```java
@RunWith(SpringRunner.class)
@SpringBootTest
public class SpringbooEmailDemoApplicationTests {

    @Autowired
    private MailService mailService;

    private static final String TO = "xxx@qq.com";
    private static final String SUBJECT = "主题 - 测试邮件";
    private static final String CONTENT = "Testing Testing Testing";

    /**
     * 测试发送普通邮件
     */
    @Test
    public void sendSimpleMailMessage() {
        mailService.sendSimpleMailMessge(TO, SUBJECT, CONTENT);
    }

    /**
     * 测试发送html邮件
     */
    @Test
    public void sendHtmlMessage() {
        String htmlStr = "<h1>Test</h1>";
        mailService.sendMimeMessge(TO, SUBJECT, htmlStr);
    }

    /**
     * 测试发送带附件的邮件
     * @throws FileNotFoundException
     */
    @Test
    public void sendAttachmentMessage() throws FileNotFoundException {
        File file = ResourceUtils.getFile("classpath:testFile.txt");
        String filePath = file.getAbsolutePath();
        mailService.sendMimeMessge(TO, SUBJECT, CONTENT, filePath);
    }

    /**
     * 测试发送带附件的邮件
     * @throws FileNotFoundException
     */
    @Test
    public void sendPicMessage() throws FileNotFoundException {
        String htmlStr = "<html><body>测试：图片1 <br> <img src=\'cid:pic1\'/> <br>图片2 <br> <img src=\'cid:pic2\'/></body></html>";
        Map<String, String> rscIdMap = new HashMap<>(2);
        rscIdMap.put("pic1", ResourceUtils.getFile("classpath:pic01.jpg").getAbsolutePath());
        rscIdMap.put("pic2", ResourceUtils.getFile("classpath:pic02.jpg").getAbsolutePath());
        mailService.sendMimeMessge(TO, SUBJECT, htmlStr, rscIdMap);
    }
}
```
## 效果
### 普通邮件
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181130151641568.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvbHRvbl9OdWxs,size_16,color_FFFFFF,t_70)
### html 邮件
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181130151658288.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvbHRvbl9OdWxs,size_16,color_FFFFFF,t_70)
### 带附件的邮件
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181130151720136.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvbHRvbl9OdWxs,size_16,color_FFFFFF,t_70)
### 带静态文件的邮件
![在这里插入图片描述](https://img-blog.csdnimg.cn/20181130151739523.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L0NvbHRvbl9OdWxs,size_16,color_FFFFFF,t_70)
# 4. 总结
SpringBoot 中提供了简易的配置方法和简单的接口，极大方便了开发者在应用中开发有关发送邮件的服务。以上，就是 SpringBoot 中配置邮件服务的操作。

# 5. 附

项目 Demo 地址:[springboot-email-demo](https://github.com/MartinMa94/springboot-email-demo)

----
站在前人的肩膀上前行，感谢以下博客及文献的支持。

- [springboot(十)：邮件服务](https://www.cnblogs.com/ityouknow/p/6823356.html)