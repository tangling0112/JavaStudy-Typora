# 使用原生`Servlet API`实现

```java
@RequestMapping("/download")
public void downloadFile(HttpServletRequest request, HttpServletResponse response) throws IOException {
    response.setHeader("Content-Type","application/octet-stream;charset=utf-8");
    response.setHeader("Content-disposition","attachment;filename=PythonCode.py");
    File file = new File("C:\\Users\\Administrator\\Desktop\\uploader\\CommonFiledemo.py");
    FileInputStream fileInputStream = new FileInputStream(file);
    OutputStream webOutputStream = response.getOutputStream();
    int len = 0;
    while((len = fileInputStream.read(buffer))>0){
        webOutputStream.write(buffer);
    }
}
```



# 使用`ResponseEntity`实现

```java
```

