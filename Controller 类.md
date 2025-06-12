### `Controller` 类

```
@RestController
@RequestMapping("/book")
public class Controller {
```

- `@RestController`: 标明该类是一个控制器类，并且每个方法都会返回一个对象，该对象会自动转换为 JSON 格式并作为 HTTP 响应体返回。
- `@RequestMapping("/book")`: 将该类中的所有方法的 URL 路径都与 `/book` 路径相关联，意味着所有的请求路径都会以 `/book` 开头。

### `listStatus` 方法

```
private List<String> listStatus(Path path) {
    List<String> filePaths = new ArrayList<>();
    //获取文件配置信息
    Configuration conf = new Configuration();
    conf.set("fs.defaultFS", "hdfs://192.168.20.102:8020");
```

- 该方法用于获取指定路径下所有文件的路径，并将文件路径存储在 `filePaths` 列表中。
- `Configuration conf = new Configuration();`：创建一个新的 HDFS 配置信息对象。
- `conf.set("fs.defaultFS", "hdfs://192.168.20.102:8020");`：设置 HDFS 文件系统的 URI，指向 `192.168.20.102` 服务器上的 HDFS 系统，并使用 `8020` 端口。

```
    try {
        // 获取 HDFS 文件系统
        FileSystem fs = FileSystem.get(conf);
```

- `FileSystem fs = FileSystem.get(conf);`：通过配置创建一个 HDFS 文件系统对象，用于访问 Hadoop 文件系统。

```
       // 获取该路径下的文件和目录
        FileStatus[] fileStatuses = fs.listStatus(path);
```

- `fs.listStatus(path);`：列出指定路径下的所有文件和目录（`path` 是方法的参数，表示你想查询的路径）。`fileStatuses` 是包含文件和目录信息的数组。

```
        // 遍历文件和目录
        for (FileStatus status : fileStatuses) {
            if (status.isFile()) {
                // 获取文件的相对路径，并添加到列表中
                String relativePath =
                        status.getPath().toString().replace
                                ("hdfs://192.168.20.102:8020", "");
                filePaths.add(relativePath);
            } else if (status.isDirectory()) {
                // 如果是目录，递归调用 listStatus 进行子目录的搜索
                filePaths.addAll(listStatus(status.getPath()));
            }
        }
```

- 这个循环遍历所有的文件和目录：
  - `status.isFile()` 判断是否是文件。如果是文件，则提取文件的路径。
  - `status.getPath().toString()` 获取文件的完整路径。
  - 使用 `replace("hdfs://192.168.20.102:8020", "")` 去掉 HDFS 的 URI 部分，保留文件的相对路径。
  - 将每个文件的相对路径添加到 `filePaths` 列表中。
  - 如果是目录，则递归调用 `listStatus` 方法来进一步获取该目录下的文件和子目录。

```
        fs.close();  // 关闭文件系统连接
    } catch (IOException e) {
        e.printStackTrace();
    }
```

- `fs.close();` 关闭 HDFS 文件系统连接，释放资源。
- 捕获 `IOException` 异常，如果发生错误，将其打印出来。

```
   return filePaths;
}
```

- 返回存储了文件路径的 `filePaths` 列表。

### `search` 方法

```
@GetMapping("/search")
public String search(@RequestParam("title") String title)  {
```

- `@GetMapping("/search")`：定义了一个 GET 请求的端点，路径为 `/book/search`，并且接收一个名为 `title` 的查询参数。
- `@RequestParam("title") String title`：通过请求参数 `title` 获取用户输入的搜索关键字。

```

    List<String> files = listStatus(new Path("/"));  // 获取 HDFS 的文件信息
```

- 调用 `listStatus` 方法，传入根路径（`/`），获取 HDFS 上根目录及其子目录下的所有文件路径。

```
    List<String> targetFileList = new ArrayList<>();
    for (String file : files) {
        if (file.contains(title)) {
            targetFileList.add(file);
        }
    }
```

- `targetFileList` 用于存储匹配的文件路径。
- 遍历 `files` 中的每个文件路径，判断文件路径是否包含用户传入的 `title` 字符串。如果匹配，就将该文件路径添加到 `targetFileList` 中。

```

    R r = new R(200, "success", targetFileList);
```

- 创建一个 `R` 对象，并设置响应状态码为 `200`，消息为 `"success"`，返回的结果是匹配的文件路径列表 `targetFileList`。

```
   ObjectMapper mapper = new ObjectMapper();
    String json = null;
    try {
        json = mapper.writeValueAsString(r);
    } catch (IOException e) {
        throw new RuntimeException(e);
    }
```

- 创建一个 `ObjectMapper` 对象，它可以将 Java 对象转换为 JSON 格式的字符串。
- 使用 `mapper.writeValueAsString(r)` 将 `R` 对象转换为 JSON 字符串。如果转换过程中发生错误，将抛出 `RuntimeException`。

```
   return json;
}
```

- 返回 JSON 格式的响应，这个响应包含了状态码、消息和匹配的文件路径列表。

### 总结：

- 该类主要功能是实现一个基于 Spring Boot 的 RESTful API，允许用户通过传递标题（`title`）参数来搜索 HDFS 中的文件。
- `listStatus` 方法递归遍历 HDFS 文件系统，获取文件路径，并去掉 HDFS URI 部分。
- `search` 方法通过查询参数 `title` 来匹配文件路径，返回匹配的文件路径列表，并将结果以 JSON 格式返回给客户端。