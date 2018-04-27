# Java上传下载文件

本文总结了Java上传下载文件的几种方法，包括使用Servlet实现文件的上传下载、使用FileUpload实现文件的上传下载。并详细介绍了文件存储的组织方式，可以运用到实际的JavaWeb开发中。


## 1 文件上传下载原理
### 1.1 上传原理
在TCP/IP中，最早出现的文件上传机制是FTP。它是将文件由客户端发送到服务器端的标准机制。但是在jsp编程中不能使用FTP来上传文件，这是由jsp运行机制决定的。

在jsp中，可行的方案是通过为表单元素设置`Method="post"`和`enctype="multipart/form-data"`属性，让表单提交的数据以二进制编码的方式提交，在接收此请求的Servlet中用二进制流来获取内容，就可以取得上传文件的内容，从而实现文件的上传。

表单ENCTYPE属性
- `application/x-www-form-urlencoded`
  这是默认编码方式，它只处理表单域里的value属性值，采用这种编码方式的表单会将表单域的值处理成URL编码方式。
- `multipart/form-data`
  这种编码方式的表单会以二进制流的方式来处理表单数据，这种编码方式会把文件域指定文件的内容也封装到请求参数里。
- `text/plain`
  这种方式主要适用于直接通过表单发送邮件的方式。

### 1.2 下载原理
1）需要通过`HttpServletResponse.setContentType`方法设置Content-Type头字段的值，以激活某个程序来处理浏览器无法使用的MIME类型，例如，"application/octet-stream"或"application/x-msdownload"等
2）需要通过`HttpServletResponse.setHeader`方法设置Content-Disposition头的值为"attachment;filename=文件名"
3）读取下载文件，调用`HttpServletResponse.getOutputStram`方法返回的ServletOutputStream对象来向客户端写入附件文件内容

## 2 使用Servlet实现文件的上传下载
### 2.1文件上传步骤
1）获取request当中的流信息，保存到临时文件
2）从临时文件当中得到上传的文件名，以及文件内容起止位置
3）根据文件起止位置，读取上传文件内容，保存到本地
```java 
protected void doPost(HttpServletRequest request, HttpServletResponse response)
throws ServletException, IOException {
    // 1.获取request当中的流信息，保存到临时文件
    InputStream fileSource = request.getInputStream();
    String tempFileName = "E:/tempFile";
    File tempFile = new File(tempFileName);  // tempFile指向临时文件
    // outputStream文件输出流指向这个临时文件
    FileOutputStream outputStream = new FileOutputStream(tempFile); 
    byte bytes[] = new byte[1024];
    int n;
    while((n = fileSource.read(bytes)) != -1) {
        outputStream.write(bytes, 0, n);
    }
    outputStream.close();
    fileSource.close();

    // 2.从临时文件当中得到上传的文件名，以及文件内容起止位置
    RandomAccessFile randomAccessFile= new RandomAccessFile(tempFile, "r");
    randomAccessFile.readLine();
    String sr = randomAccessFile.readLine();
    int beginIndex = sr.lastIndexOf("=") + 2;
    int endIndex = sr.lastIndexOf("\"");
    String filename = sr.substring(beginIndex, endIndex);
    System.out.println(filename);

    // 重新定位文件指针到文件头
    randomAccessFile.seek(0);
    long startPosition = 0;
    int i = 1;
    // 获取文件内容开始位置
    while((n = randomAccessFile.readByte()) != -1 && i <= 4) {
        if (n == '\n') {
            startPosition = randomAccessFile.getFilePointer();
            i++;
        }
    }
    //startPosition = startPosition - 1;
    // 获取文件内容结束位置
    randomAccessFile.seek(randomAccessFile.length());
    long endPosition = randomAccessFile.getFilePointer();
    int j = 1;
    while(endPosition >= 0 && j <= 2) {
        endPosition--;
        randomAccessFile.seek(endPosition);
        if (randomAccessFile.readByte() == '\n') {
            j++;
        }
    }
    endPosition = endPosition - 1;

    // 3.根据文件起止位置，读取上传文件内容，保存到本地
    // 设置保存上传文件的路径
    String realPath = getServletContext().getRealPath("/resources/")+ "images";
    System.out.println(realPath);
    File fileupload = new File(realPath);
    if (!fileupload.exists()) {
        fileupload.mkdir();
    }
    File saveFile = new File(realPath, filename);
    RandomAccessFile randomFile = new RandomAccessFile(saveFile, "rw");
    randomAccessFile.seek(startPosition);
    while(startPosition < endPosition) {
        randomFile.write(randomAccessFile.readByte());
        startPosition = randomAccessFile.getFilePointer();
    }
    randomAccessFile.close();
    randomFile.close();
    tempFile.delete();

    request.setAttribute("result", "上传成功");
    RequestDispatcher dispatcher = request.getRequestDispatcher("/");
    dispatcher.forward(request, response);
}
```

### 2.2 文件下载步骤
1）获取目标文件的地址及文件名
2）设置响应类型及响应头
3）获取目标文件的输出流，输出流写入文件内容
```java 
protected void doGet(HttpServletRequest request, HttpServletResponse response) 
throws ServletException, IOException {
    String path=getServletContext().getRealPath("/resources/") + "images/";
    System.out.println(path);
    String filename = request.getParameter("filename");
    File file = new File(path + filename);
    if (file.exists()) {
        // 设置相应类型 （或application/octet-stream）
        response.setContentType("application/x-msdownload");
        response.setHeader("Content-Disposition", "attachment;filename=\""
         + filename + "\"");
        InputStream inputStream = new FileInputStream(file);
        ServletOutputStream servletOutputStream=response.getOutputStream();
        byte bytes[] = new byte[1024];
        int n;
        while((n = inputStream.read(bytes)) != -1) {
            servletOutputStream.write(bytes, 0, n);
        }
        servletOutputStream.close();
        inputStream.close();
    } else {
        request.setAttribute("errorResult", "文件不存在，下载失败");
        RequestDispatcher dispatcher = request.getRequestDispatcher("/");
        dispatcher.forward(request, response);
    }
}
```

### 2.3 前端
前端主要代码
```html
<form action="upload" method="post" enctype="multipart/form-data">
        请选择图片：<input id="myfile" name="myfile" type="file"/>
        <input type="submit" value="提交"/>${result}
</form>
下载：<a href="download?filename=demo.jpg">demo.jpg</a>${errorResult}
```

### 2.4 问题
以上就是使用Servlet实现的文件上传下载，但是有一个很明显的问题，就是在找文件名的时候，不同的浏览器会返回不同的结果。上面的代码只适用于Chrome，换成Edge就会报“文件名、目录名或卷标语法不正确”的错误。
比如，有一个文本文件，内容为“Hello”。
通过Edge上传，其内容为
```
-----------------------------7e0d49540942
Content-Disposition: form-data; name="myfile"; filename="C:\Users\SJF\Desktop\1.txt"
Content-Type: text/plain

Hello
-----------------------------7e0d49540942--
```
通过Chrome上传，其内容为
```
------WebKitFormBoundaryPGdMM9UCDlcxFOfW
Content-Disposition: form-data; name="myfile"; filename="1.txt"
Content-Type: text/plain

Hello
------WebKitFormBoundaryPGdMM9UCDlcxFOfW--
```
filename的不同导致了上面那段代码无法复用，同时该代码无法支持批量文件上传下载、不能限制上传文件大小及类型。总的来说，上述代码理论意义大于实际意义，通过这些代码我们可以理清文件上传下载的思路。更为常用的方法是通过fileupload插件进行文件的上传下载。

## 3 使用FileUpload实现文件的上传下载 
直接看[这篇博客](http://www.cnblogs.com/xdp-gacl/p/4200090.html)就好。

## 4 文件存储的组织方式  
当上传的文件较多，同时硬件条件不足以支持分布式文件系统的情况下，使用hash算法打散存储是一个可行的方案。

首先定义一个类，用于存储网络文件到本地，建立起合适的文件组织结构，并提供web访问接口。该类命名为FileManager.java,首先定义类的构造方式，与类的私有变量等。
```java 
public class FileManager {
    public static final String ERROR = "ERROR";
    public static final String NONE = "NONE";
    //根目录
    private String basePath;
    
    //默认存储根路径
    public FileManager(){
        this.basePath = 
        this.getClass().getResource("/").getPath().split("webapps")[0] + "webapps/file";
    }

    public String getBasePath(){
        return this.basePath;
    }
}
```
采用固定的文件夹来存储文件，此处是指tomcat/webapps/[war包名]/file。其中file是指我们另外部署在tomcat上的一个项目（一个Hello word就足够了）。程序在本地调试的时候，记得采用war包形式，同时部署file项目。

再就是存储文件的组织方式了，为了不使单个文件夹下的文件数目过多，采用文件名的哈希值的后16位构建深度为4的文件夹结构来存储各类文件。原则上，此处饱和情况下可创建2^16个子文件夹用于存储文件。根据文件名构建文件存储路径的方法代码如下：
```java 
private String getPath(String filename) {
    try{
        //得到文件名的hashCode的值，取二进制后16位，文件夹深度为4
        int hashcode = filename.hashCode();
        String dir_suffix = "/" + (hashcode & 0xf) + "/" + 
            ((hashcode & 0xf0) >> 4) + "/" + ((hashcode & 0xf00) >> 8) + 
            "/" + ((hashcode & 0xf000) >> 12);
        //构造新的保存目录
        String dir = basePath + dir_suffix;
        //File既可以代表文件也可以代表目录
        File file = new File(dir);
        //如果目录不存在
        if (!file.exists()) {
            //创建目录
            file.mkdirs();
        }
        return dir;
    } catch (Exception e) {
        e.printStackTrace();
        return basePath + "/error";
    }
}
```
由于FileUpload插件读取的web文件可获得一个InputStream，此处，传入该InputStream的输入流，将文件存储到上一步构建的子文件夹下：
```java 
public String save(String filename, String ext, InputStream in) 
throws IOException {
    FileOutputStream fos = null;
    try{
        String path = this.getPath(filename);
        fos = new FileOutputStream(path + "/" + filename + "." + ext);
        byte [] buffer = new byte[1024];
        int length = 0;
        while ((length = in.read(buffer)) > 0){
            fos.write(buffer,0,length);
        }
        return path + "/" + filename + "." + ext;
    } catch (Exception e) {
        e.printStackTrace();
        return ERROR;
    } finally {
        if(null != in){
            in.close();
        }
        if(null != fos){
            fos.close();
        }
    }
}
```
同时，我们需要为存储在服务器上的文件提供访问链接。如下方法中，根据存储规则，由文件名计算出文件的访问路径，而不是去遍历文件夹，这种方式便显得尤为高效。
```java 
public String getFileURI(String filename, String ext) {
    int hashcode = filename.hashCode();
    String dir_suffix = "/" + (hashcode & 0xf) + "/" + 
        ((hashcode & 0xf0) >> 4) + "/" + ((hashcode & 0xf00) >> 8) 
        + "/" + ((hashcode & 0xf000) >> 12);
    String realPath =  this.getPath(filename) + "/" + filename + "." + ext;
    File file = new File(realPath);
    if(file.exists()){
        String domain = SystemUtil.getProperty("domain");
        return domain.substring(0,domain.lastIndexOf("/")) + "/file" 
            + dir_suffix + "/" + filename + "." + ext;
    } else {
        return NONE;
    }
}
```
另外，在web Controller层，利用fileupload插件获取前端上传文件form各输入控件信息的代码如下：
```java 
@RequestMapping(value = "/file")
public void file (HttpServletRequest request,HttpServletResponse response) 
throws IOException{
    response.setContentType("application/json;charset=utf-8");
    try{
        //使用Apache文件上传组件处理文件上传步骤：
        //1、创建一个DiskFileItemFactory工厂
        DiskFileItemFactory factory = new DiskFileItemFactory();
        //设置工厂的缓冲区的大小，当上传的文件大小超过缓冲区的大小时，
        //就会生成一个临时文件存放到指定的临时目录当中。
        //设置缓冲区的大小为100KB，如果不指定，那么缓冲区的大小默认是10KB
        factory.setSizeThreshold(1024*100);
        /*//设置上传时生成的临时文件的保存目录
        factory.setRepository(tmpFile);*/
        //2、创建一个文件上传解析器
        ServletFileUpload upload = new ServletFileUpload(factory);
        //监听文件上传进度
        upload.setProgressListener(new ProgressListener(){
            public void update(long pBytesRead, long pContentLength, int arg2) {
                System.out.println("文件大小为：" + pContentLength +
                     ",当前已处理：" + pBytesRead);
            }
        });
        //解决上传文件名的中文乱码
        upload.setHeaderEncoding("UTF-8");
        //3、判断提交上来的数据是否是上传表单的数据
        if(!ServletFileUpload.isMultipartContent(request)){
            //按照传统方式获取数据
            return;
        }

        //设置上传单个文件的大小的最大值，目前是设置为1024*1024字节，也就是1MB
        upload.setFileSizeMax(1024*1024*10);
        //设置上传文件总量的最大值，最大值=同时上传的多个文件的大小的
        //最大值的和，目前设置为10MB
        upload.setSizeMax(1024*1024*30);
        //4、使用ServletFileUpload解析器解析上传数据，解析结果返回的是一个
        //List<FileItem>集合，每一个FileItem对应一个Form表单的输入项
        List<FileItem> list = upload.parseRequest(request);
        for(FileItem item : list){
            //如果fileitem中封装的是普通输入项的数据
            if(item.isFormField()){
                String name = item.getFieldName();
                //解决普通输入项的数据的中文乱码问题
                String value = item.getString("UTF-8");
                //value = new String(value.getBytes("iso8859-1"),"UTF-8");
                System.out.println(name + "=" + value);
            } else {//如果fileitem中封装的是上传文件
                //得到上传的文件名称，
                String filename = item.getName();
                System.out.println(filename);
                if(filename == null || filename.trim().equals("")){
                    continue;
                }
                //注意：不同的浏览器提交的文件名是不一样的，有些浏览器提交
                //上来的文件名是带有路径的，如：  
                //c:\a\b\1.txt，而有些只是单纯的文件名，如：1.txt
                //处理获取到的上传文件的文件名的路径部分，只保留文件名部分
                filename = filename.substring(filename.lastIndexOf("\\")+1);
                //得到上传文件的扩展名
                String fileExtName = 
                    filename.substring(filename.lastIndexOf(".")+1);
                filename = filename.substring(0,filename.lastIndexOf("."));
                //如果需要限制上传的文件类型，那么可以通过文件的扩展名来
                //判断上传的文件类型是否合法
                System.out.println("上传的文件的扩展名是："+fileExtName);
                //保存文件
                FileManager fileManager = new FileManager();
                if(FileManager.ERROR.equals(
                    fileManager.save(filename,fileExtName,
                        item.getInputStream()))) {
                            response.getWriter().write(
                              JsonUtil.statusResponse(1,"上传文件失败",null));
                } else {
                    response.getWriter().write(
                        JsonUtil.statusResponse(0,"上传文件成功",
                            fileManager.getFileURI(filename,fileExtName)));
                }
            }
        }
    } catch (FileUploadBase.FileSizeLimitExceededException e) {
        e.printStackTrace();
        response.getWriter().write(JsonUtil.statusResponse(1,
            "单个文件超出最大值！！！",null));
    } catch (FileUploadBase.SizeLimitExceededException e) {
        e.printStackTrace();
        response.getWriter().write(JsonUtil.statusResponse(1,
            "上传文件的总的大小超出限制的最大值！！！",null));
    } catch (Exception e) {
        e.printStackTrace();
        response.getWriter().write(JsonUtil.statusResponse(1,
            "其他异常，上传失败！！！",null));
    }
    return;
}
```
对应前端测试页面代码如下：
```html 
<body>
    <form action="file" enctype="multipart/form-data"
          method="post">
        上传用户：<input type="text" name="username"><br/>
        上传文件：<input type="file" name="file2"><br/>
        <input type="submit" value="提交">
    </form>
</body>
```

参考：
[JavaWeb学习总结(五十)——文件上传和下载](http://www.cnblogs.com/xdp-gacl/p/4200090.html)
[java web --fileupload插件网页文件管理](http://blog.csdn.net/u013248535/article/details/52326489)