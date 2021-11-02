---
title: Ajax 实现文件上传（解决upload.parseRequest为空问题）
tags: [Ajax, Javascript]
categories: 
  - Web
  - Web Develop
abbrlink: 25168d50
date: 2018-03-15 21:52:04
---

## 一、fileupload

这种方式也是目前网上主要绝大部分 Ajax 文件上传的方法，前台代码如下：

```html
<!-- 定义上传按钮 -->
<input type="file" id="uploadFile">
<button onclick="uploadFile()" >上传文件</button>
```

```javascript
<script>
    var fileSize;
    var fileName;

    // 动态获取文件名和文件大小，如果要做判断的话
    $("#uploadFile").change(function(){
        var file = this.files[0];
        fileName = file.name;
        fileSize = file.size;
    });

    function uploadFile() {
        var formData = new FormData();
        var file = document.getElementById("uploadFile").files[0];
        formData.append("file",file);
        formData.append("fileName",fileName);
        formData.append("fileSize",fileSize);

    // 省略掉了success和error的处理
        sendFile("/uploadFile",formData,true,function (res) {
        },function (XMLHttpRequest, textStatus, errorThrown) {
        });
    }
    
    function sendFile(_url, _data, _async, _successCallback, _errorCallback) {
    $.ajax({
        type: "post",
        async: _async,
            url: _url,
            dataType: 'json',
            // 告诉jQuery不要去处理发送的数据
            processData : false,
            // 告诉jQuery不要去设置Content-Type请求头
            contentType: false,
            data: _data,
            success: function (msg) {
                _successCallback(msg);
            },
            error: function (error) {
                _errorCallback(error);
            }
    });
</script>
```

在后台，首先导入 `commons-fileupload` 包，我还用到了 `connons-io` 包，一并导入。

```xml
<dependency>
    <groupId>commons-fileupload</groupId>
    <artifactId>commons-fileupload</artifactId>
    <version>1.3.1</version>
</dependency>
<dependency>
    <groupId>commons-io</groupId>
    <artifactId>commons-io</artifactId>
    <version>2.5</version>
</dependency>
```

在 controller 层编写接口：

```java
package jit.wxs.web;

import org.apache.commons.io.FileUtils;
import org.apache.commons.fileupload.FileItem;
import org.apache.commons.fileupload.disk.DiskFileItemFactory;
import org.apache.commons.fileupload.servlet.ServletFileUpload;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RestController;

import javax.servlet.http.HttpServletRequest;
import java.io.File;
import java.util.List;

/**
 * @author jitwxs
 * @date 2018/3/15 18:56
 */
@Controller
public class FileController {
    @PostMapping("/uploadFile")
    public String uploadFile(HttpServletRequest request) throws Exception {
        int sizeThreshold = 1024 * 1024;
        File repository = new File("/temp");

        // 创建磁盘文件项工厂，参数为缓存文件大小和临时文件位置
        DiskFileItemFactory factory = new DiskFileItemFactory(sizeThreshold, repository);

        // 创建文件上传核心类
        ServletFileUpload upload = new ServletFileUpload(factory);
        upload.setHeaderEncoding("UTF-8");

        // 判断是否是Multipart类型的
        if(ServletFileUpload.isMultipartContent(request)) {
            // 解析request并获得文件项集合
            List<FileItem> list = upload.parseRequest(request);
            if(list != null) {
                for(FileItem item : list) {
                    // 判断参数是普通参数还是文件参数
                    if(item.isFormField()) {
                        // 获得普通参数的key、value（即formData的fileName和fileSize）
                        String fieldName = item.getFieldName();
                        String fieldValue = item.getString("UTF-8");
                        System.out.println("FormField：k=" + fieldName + ",v=" + fieldValue);
                    } else {
                        //获得文件参数（即formData的file）
                        String fileName = item.getName();
                        String path = "/" + fileName;
                        // 上传文件
                        FileUtils.uploadFile(item,path);
                    }
                }
            }
        }
        
        return "返回给前台的消息";
    }
}

public static void uploadFile(FileItem item, String targetFilePath) throws IOException {
    InputStream in = item.getInputStream();
    OutputStream out = new FileOutputStream(targetFilePath);
    IOUtils.copy(in, out);
    in.close();
    out.close();

    // 删除临时文件
    item.delete();
}
```

当我项目环境是 `Spring MVC` 时这样用没有问题，当我更换为 `Spring Boot` 后在 `List<FileItem> list = upload.parseRequest(request);` 这个地方的值始终为 null。最后我参考 http://blog.csdn.net/u013248535/article/details/55823364 这篇文章找到了解决方法。

## 二、StandardMultipartHttpServletRequest

直接将 HttpServletRequest 强转为 `StandardMultipartHttpServletRequest`，然后通过 `getFileNames()` 方法进行迭代，用这种方法不用使用 `common-fileupload`包了，代码如下：

```java
package jit.wxs.web;

import org.apache.commons.io.FileUtils;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RestController;
import org.springframework.web.multipart.MultipartFile;
import org.springframework.web.multipart.support.StandardMultipartHttpServletRequest;

import javax.servlet.http.HttpServletRequest;
import java.io.File;
import java.util.Iterator;

/**
 * @author jitwxs
 * @date 2018/3/15 18:56
 */
@RestController
public class FileController {
    @PostMapping("/uploadFile")
    public String uploadFile(HttpServletRequest request) throws Exception {
        StandardMultipartHttpServletRequest req = (StandardMultipartHttpServletRequest) request;

        // 遍历普通参数（即formData的fileName和fileSize）
        Enumeration<String> names = req.getParameterNames();
        while (names.hasMoreElements()) {
            String key = names.nextElement();
            String val = req.getParameter(key);
            System.out.println("FormField：k=" + key + "v=" + val);
        }

        // 遍历文件参数（即formData的file）
        Iterator<String> iterator = req.getFileNames();
        while (iterator.hasNext()) {
            MultipartFile file = req.getFile(iterator.next());
            String fileNames = file.getOriginalFilename();
            // 文件名
            fileNames = new String(fileNames.getBytes("UTF-8"));
            //int split = fileNames.lastIndexOf(".");
            // 文件前缀
            //String fileName = fileNames.substring(0, split);
            // 文件后缀
            //String fileType = fileNames.substring(split + 1, fileNames.length());
            // 文件大小
            //Long fileSize = file.getSize();
            // 文件内容
            byte[] content = file.getBytes();

            FileUtils.writeByteArrayToFile(new File(fileNames), content);
        }

        return "返回给前台的消息";
    }
}
```

这样就能够实现文件上传，如果出现大小超出限制的错误，记得在 `springboot` 的配置文件中配置下：

```ini
# 上传单个文件最大允许
spring.servlet.multipart.max-file-size=10MB
# 每次请求最大允许
spring.servlet.multipart.max-request-size=100MB
```

## 三、文件上传控件

在条件允许的情况下，可以使用一些第三方的文件上传控件。推荐下我使用过的控件：

- BootstrapFileInput
- WebUploader
