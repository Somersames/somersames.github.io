---
title: 利用Jquery和Spring异步上传文件
date: 2018-03-29 11:28:04
tags: [spring,jquery]
categories: Java
---
## 异步上传文件：
用Jquery的异步上传文件的时候需要引入一个js文件`jquery.form.min.js`，用这个文件里面的$.ajaxSubmit()方法来实现一个异步的文件上传功能。
```
 $(this).ajaxSubmit({
            type:'POST',
            url: "/uploadfile",
            dataType: 'json',
            data: serializeData,
            contentType: false,
            cache: false,
            processData:false,
            beforeSubmit: function() {
        },
        uploadProgress: function (event, position, total, percentComplete){
        },
        success:function(){

        },
        error:function(data){
            alert('上传图片出错');
        }
    });
```
在这里的话`$("")`函数需要是form的id，而且`beforeSubmit`可以在上传文件之前可以做一些检查，例如文件后缀或者文件大小之类的检查。

在后端的话接受上传的文件其实跟Servlet差不多，主要是从request中获取请求流，参数的话需要标记为这个` @RequestParam("file") MultipartFile file`,最后在SpringMvc中有一个方法可以将上传的文件通过移动或者复制然后转移到我们指定的文件夹中：
```
void transferTo(java.io.File dest)
         throws java.io.IOException,
                java.lang.IllegalStateException
Transfer the received file to the given destination file.
This may either move the file in the filesystem, copy the file in the filesystem, or save memory-held contents to the destination file. If the destination file already exists, it will be deleted first.

If the target file has been moved in the filesystem, this operation cannot be invoked again afterwards. Therefore, call this method just once in order to work with any storage mechanism.

NOTE: Depending on the underlying provider, temporary storage may be container-dependent, including the base directory for relative destinations specified here (e.g. with Servlet 3.0 multipart handling). For absolute destinations, the target file may get renamed/moved from its temporary location or newly copied, even if a temporary copy already exists.

Parameters:
dest - the destination file (typically absolute)
Throws:
java.io.IOException - in case of reading or writing errors
java.lang.IllegalStateException - if the file has already been moved in the filesystem and is not available anymore for another transfer
See Also:
FileItem.write(File), Part.write(String)
```
也就是说这个方法`transferTo`会将接收到的文件移动或者copy到指定的位置。
那么在Controller中的主要方法体是：
```
@RequestMapping(value = "/uploadfile" ,method = RequestMethod.POST)
        @ResponseBody
        public String uploadfile( @RequestParam("file") MultipartFile file) throws IOException {
        Map<String ,Object> map =new HashMap<>();
        /*  上传的文件为空 */
        if(file ==  null || file.isEmpty()){
            map.put("code","0");
            map.put("msg","fail");
            return JSON.toJSONString(map);
        }else{
            String rootPath = "D:\\disk";
            File dir = new File(rootPath + File.separator + "tmpFiles");
            File serverFile = new File(dir.getAbsolutePath() + File.separator + file.getOriginalFilename());
            file.transferTo(serverFile);
            map.put("code","1");
            map.put("msg","success");
            return JSON.toJSONString(map);
        }
        }
```
这个方法的话如果文件上传成功会返回一个json格式的信息，从而提示前端文件已经上传成功了。

## 生成验证码：
在这里的话顺带在说下关于验证码的事情，在Spring里面使用验证码的话可以使用google的`kaptcha`这个jar包，因为在maven仓库中的话是没有这个版本的，所以这个需要自己下载到本地然后通过命令行导入，最后才可以在pom文件中引用这个包；导入方法如下：
`mvn install:install-file  -Dfile=jar包位置 -DgroupId=com.google.code -DartifactId=kaptcha -Dversion=2.3.2 -Dpackaging=jar`，这样在pom文件中便可以引用了。
如下：
```
 <dependency>
      <groupId>com.google.code</groupId>
      <artifactId>kaptcha</artifactId>
      <version>2.3.2</version>
    </dependency>
```
同时在ApplicationContext.xml中添加一个bean.
```
<bean id="captchaProducer" class="com.google.code.kaptcha.impl.DefaultKaptcha">
        <property name="config">
            <bean class="com.google.code.kaptcha.util.Config">
                <constructor-arg>
                    <props>
                        <!-- 图片边框 -->
                        <prop key="kaptcha.border">no</prop>
                        <!-- 图片宽度 -->
                        <prop key="kaptcha.image.width">95</prop>
                        <!-- 图片高度 -->
                        <prop key="kaptcha.image.height">45</prop>
                        <!-- 验证码背景颜色渐变，开始颜色 -->
                        <prop key="kaptcha.background.clear.from">248,248,248</prop>
                        <!-- 验证码背景颜色渐变，结束颜色 -->
                        <prop key="kaptcha.background.clear.to">248,248,248</prop>
                        <!-- 验证码的字符 -->
                        <prop key="kaptcha.textproducer.char.string">0123456789abcdefghijklmnopqrstuvwxyz这是一个测试验证码的例子</prop>
                        <!-- 验证码字体颜色 -->
                        <prop key="kaptcha.textproducer.font.color">0,0,255</prop>
                        <!-- 验证码的效果，水纹 -->
                        <prop key="kaptcha.obscurificator.impl">com.google.code.kaptcha.impl.WaterRipple</prop>
                        <!-- 验证码字体大小 -->
                        <prop key="kaptcha.textproducer.font.size">35</prop>
                        <!-- 验证码字数 -->
                        <prop key="kaptcha.textproducer.char.length">4</prop>
                        <!-- 验证码文字间距 -->
                        <prop key="kaptcha.textproducer.char.space">2</prop>
                        <!-- 验证码字体 -->
                        <prop key="kaptcha.textproducer.font.names">new Font("Arial", 1, fontSize), new Font("Courier", 1, fontSize)</prop>
                        <!-- 不加噪声 -->
                        <prop key="kaptcha.noise.impl">com.google.code.kaptcha.impl.NoNoise</prop>
                    </props>
                </constructor-arg>
            </bean>
        </property>
    </bean>
```

最后在Controller添加如下方法：
```
@RequestMapping(value = "verificationcode", method = RequestMethod.GET)
    public ModelAndView Verificationcode(ModelAndView modelAndView, HttpServletRequest request, HttpServletResponse response,
                                         @RequestParam(value = "timestamp", required = false) String timestamp) throws IOException {
        /* 添加时间戳，以设置验证码的过期时间*/
        if (timestamp == null || timestamp.length() == 0) {
            modelAndView.addObject("timestamp", System.currentTimeMillis());
        } else {
            modelAndView.addObject("timestamp", timestamp);
        }
        response.setDateHeader("Expires", 0);
        response.setHeader("Cache-Control", "no-store, no-cache, must-revalidate");
        response.addHeader("Cache-Control", "post-check=0, pre-check=0");
        response.setHeader("Pragma", "no-cache");
        response.setContentType("image/jpeg");
        String capText = captchaProducer.createText();
        request.getSession().setAttribute(Constants.KAPTCHA_SESSION_KEY, capText);
        BufferedImage bi = captchaProducer.createImage(capText);
        ServletOutputStream out = response.getOutputStream();
        ImageIO.write(bi, "jpg", out);
        try {
            out.flush();
        } finally {
            out.close();
        }
        return null;
    }
```

最后在前端页面引用
```
<img src="verificationcode">  //在这里随意设置宽度和高度即可。
```