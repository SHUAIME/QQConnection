> QQ OAuth2.0接入

## 【创建项目】

使用IDEA创建一个Spring Boot项目，依赖下面会给出

### pom.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.2.2.RELEASE</version>
        <relativePath/> <!-- lookup parent from repository -->
    </parent>

    <groupId>com.yan</groupId>
    <artifactId>qqconnection</artifactId>
    <version>1.0-SNAPSHOT</version>

    <properties>
        <java.version>1.8</java.version>
    </properties>

    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-thymeleaf</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>
        <!--        okhttp依赖-->
        <dependency>
            <groupId>com.squareup.okhttp3</groupId>
            <artifactId>okhttp</artifactId>
            <version>3.14.1</version>
        </dependency>
        <dependency>
            <groupId>com.alibaba</groupId>
            <artifactId>fastjson</artifactId>
            <version>1.2.58</version>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
            <exclusions>
                <exclusion>
                    <groupId>org.junit.vintage</groupId>
                    <artifactId>junit-vintage-engine</artifactId>
                </exclusion>
            </exclusions>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>

</project>

```

###  application.properties 配置信息

```
#QQ互联申请的应用id
QQAPPID=101813698
#QQ互联申请的应用key
QQAPPKEY=7265da646c6cf33f6326f6844c1c4ef3
#应用回调地址 ':' 和 '/'等特殊字符需要转义 原本的url是 http://127.0.0.1:8080/login
redirect_uri=http%3A%2F%2F127.0.0.1%3A8080%2Flogin
```

## 【Controller】

```java
package com.yan.controller;

import com.alibaba.fastjson.JSONObject;
import com.yan.entity.User;
import com.yan.util.QQConnection;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestParam;

@Controller
public class IndexController {
    @Autowired
    private QQConnection qqConnection;
    @RequestMapping("/")
    public String index(){
        // 第一步 获取QQ登录按钮url 自己拼接，这是只是演示步骤，无实际意义
        qqConnection.getUrl();
        return "index.html";
    }

    @RequestMapping("/login")
    public Object qqLogin(@RequestParam("code")String code , @RequestParam("state") String state , Model model){
        User user = new User();
        System.out.println("第二步：获取QQ互联返回的code="+code);
//        第三步 获取access token
        String accessToken = qqConnection.getAccessToken(code);
//        第四步 获取登陆后返回的 openid、appid 以JSON对象形式返回
        JSONObject userInfo = qqConnection.getUserOpenID(accessToken);
//        第五步获取用户有效(昵称、头像等）信息  以JSON对象形式返回
        String oauth_consumer_key = userInfo.getString("client_id");
        String openid = userInfo.getString("openid");
        JSONObject userRealInfo = qqConnection.getUserInfo(accessToken,oauth_consumer_key,openid);
        user.setOpenid(openid);
        user.setNickName(userRealInfo.getString("nickname"));
        user.setAvatar(userRealInfo.getString("figureurl_qq"));
        model.addAttribute("User",user);
        return "index.html";
    }
}

```

## 【工具类】

编写工具类QQConnection 配合controller 完成整个登录认证流程，具体我觉得博客上没必要再写了。请配合[QQ互联给的文档](https://wiki.connect.qq.com/使用authorization_code获取access_token) 

```java
package com.yan.util;

import com.alibaba.fastjson.JSONObject;
import okhttp3.OkHttpClient;
import okhttp3.Request;
import okhttp3.Response;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Component;

import java.io.IOException;

@Component
public class QQConnection {
    //    读取appid
    @Value("${QQAPPID}")
    private String QQAppID;
    @Value("${QQAPPKEY}")
//    读取appkey
    private String QQAppKEY;
    //读取redirect_uri
    @Value("${redirect_uri}")
    private String redirect_uri;

//    第一步：获取 QQ登录按钮url 几乎等于手动拼接 无太大意义

    public String getUrl(){
        String url = "https://graph.qq.com/oauth2.0/authorize?display=pc&response_type=code&client_id=" + QQAppID+"&redirect_uri="+redirect_uri+"&state=200";
        System.out.println("第一步：获取QQ登录按钮的url="+url);
        return null;
    }
    /*
    第三步：根据前一步得到的 Authorization Code 获取access token
     */
    public String getAccessToken(String code){
        String accessTaken="";
        String url = "https://graph.qq.com/oauth2.0/token?display=pc&grant_type=authorization_code&client_id="+QQAppID+"&client_secret="+QQAppKEY+"&redirect_uri="+redirect_uri+"&code="+code;
        OkHttpClient client = new OkHttpClient();
        Request request = new Request.Builder()
                .url(url)
                .build();
        try (Response response = client.newCall(request).execute()) {
            assert response.body() != null;
            String responseString = response.body().string();
            accessTaken = responseString.split("=")[1].split("&")[0];
            System.out.println("第三步：获取QQ互联返回的accessTaken="+accessTaken);
        } catch (IOException e) {
            e.printStackTrace();
        }
        return accessTaken;
    }

    /*
    第四步：根据得到的access token 获取用户的 openID 和 oauth_consumer_key：申请QQ登录成功后，分配给应用的openid(对当前网站应用唯一，可用于检测是否为同一用户的凭证）
     */
    public JSONObject getUserOpenID(String accessToken){
        JSONObject userInfo = new JSONObject();
        String urlProvideByQQConnection = "https://graph.qq.com/oauth2.0/me";
        String requestUrl =urlProvideByQQConnection+"?access_token="+accessToken;
        OkHttpClient client = new OkHttpClient();
        Request request = new Request.Builder()
                .url(requestUrl)
                .build();
        try (Response response = client.newCall(request).execute()) {
            assert response.body() != null;
            String UserInfoString = response.body().string().split(" ")[1];
            userInfo = JSONObject.parseObject(UserInfoString);
            System.out.println("第四步：获取QQ互联返回的openid和分配给应用的appid："+userInfo);
        } catch (IOException e) {
            e.printStackTrace();
        }
        return userInfo;
    }

    /*
    第五步：根据 access_token、oauth_consumer_key(上一步返回的client_id）、openid 获取用户有效信息 (昵称、头像等） 返回json对象
     */
    public JSONObject getUserInfo(String access_token , String oauth_consumer_key , String openid ){
        JSONObject userRealInfo = new JSONObject();
        String urlProvideByQQConnection= "https://graph.qq.com/user/get_user_info?";
        String requestUrl = urlProvideByQQConnection+"access_token="+access_token+"&oauth_consumer_key="+oauth_consumer_key+"&openid="+openid;
        OkHttpClient client = new OkHttpClient();
        Request request = new Request.Builder()
                .url(requestUrl)
                .build();
        try (Response response = client.newCall(request).execute()) {
            assert response.body() != null;
            String UserRealInfoString = response.body().string();
            userRealInfo = JSONObject.parseObject(UserRealInfoString);
            System.out.println("第五步：获取QQ互联返回的用户有效信息："+userRealInfo);
        } catch (IOException e) {
            e.printStackTrace();
        }
        return userRealInfo;
    }
}


```

## 【User类】

```java
package com.yan.entity;

import lombok.Data;

@Data
public class User {
    private String nickName;
    private String avatar;
    private String openid;
}
```

## 【前端】

```
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org" >
<head>
    <meta charset="UTF-8">
    <title>Spring Boot接入QQ登录</title>
    <link href="https://cdn.jsdelivr.net/npm/bootstrap@3.3.7/dist/css/bootstrap.min.css" rel="stylesheet">
    <link rel="stylesheet" type="text/css" href="/layui/css/layui.css">
    <style>
        #box-success{
            margin-top: 200px;
            box-shadow: 0px 5px 40px -1px rgba(96, 118, 243, 0.31);
            text-align: center;
            height: 400px;
        }
        #box-fail{
            margin-top: 200px;
            box-shadow: 0px 5px 40px -1px rgba(209, 134, 148, 0.31);
            text-align: center;
            height: 400px;
        }
        #qqicon{
            margin-top: 0px;
            width: 200px;
            height: 200px;
        }
        #login{
            width: 40%;
            text-decoration: none;
            margin-top: 50px;
 
        }
    </style>
    <script src="https://cdn.bootcss.com/jquery/3.4.1/jquery.js"></script>
    <script src="/layer/layer.js"></script>
    <script src="/layui/layui.js"></script>
</head>
<body>
<div class="row">
    <div class="row">
        <div class="col-md-3">
 
        </div>
        <div class="col-md-6" id="container">
            <div id="box-fail" th:if="${User == null}">
                <div class="alert alert-danger" role="alert">当前未登录</div>
                    <img id="qqicon" src="/img/user.png">
                <br>
                <a  id="login" class="layui-btn layui-btn-normal"  href="https://graph.qq.com/oauth2.0/show?which=Login&display=pc&display=pc&response_type=code&client_id=101813698&redirect_uri=http%3A%2F%2F127.0.0.1%3A8080%2Flogin&state=200">
                    QQ登录
                </a>
            </div>
            <div id="box-success" th:if="${User != null}">
                <div class="alert alert-success" role="alert" th:text="'登陆成功，欢迎：'+${User.nickName}+'！'"></div>
                <img id="qqicon" class="img-circle" th:src="${User.avatar}">
                <br>
            </div>
        </div>
        <div class="col-md-3">
 
        </div>
    </div>
</div>
</body>
</html>
```



## 【演示效果】

### 未登录

![img](https://cdn.jsdelivr.net/gh/SHUAIME/imgHome/11/032601.png)

### 跳转登录

![img](https://cdn.jsdelivr.net/gh/SHUAIME/imgHome/11/032602.png)

### 登录之后返回

![img](https://cdn.jsdelivr.net/gh/SHUAIME/imgHome/11/032603.png)登录之后显示头像和昵称

# 【项目地址】

Github地址： https://github.com/SHUAIME/QQConnection
