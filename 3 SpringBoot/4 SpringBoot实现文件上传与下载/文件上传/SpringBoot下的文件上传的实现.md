# 使用原生`Servlet API`实现

# 使用`@RequestPart`注解实现

```java
@RequestMapping("/upload")
public String portal(@RequestPart("file") MultipartFile file,
                     @RequestPart("photos") MultipartFile[] photos) throws IOException {
    System.out.println(photos.length);
    if(!file.isEmpty()){
        System.out.println(file.getOriginalFilename());
        File localFile = new File("C:\\Users\\Administrator\\Desktop\\uploader\\CommonFile\\"+file.getOriginalFilename());
        file.transferTo(localFile);
    }
    if(photos.length>0){
        for(MultipartFile photo : photos){
            if(photo.isEmpty()){
                break;
            }
            File localFile = new File("C:\\Users\\Administrator\\Desktop\\uploader\\img\\"+photo.getOriginalFilename());
            photo.transferTo(localFile);
        }
    }
    return "Success!";
}
```

