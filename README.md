# Cache

 Cache 提供了一个良好的公共 API，具有开箱即用的实现和出色的定制可能性。 Cache 利用 Swift 4中的 Codable 来执行序列化。

## 一、功能介绍

* 使用 Swift 4 Codable。符合 Codable 任何内容都将由 Storage 轻松保存和加载。
* 与内存和磁盘存储混合。
* 通过 DiskConfig 和 MemoryConfig 许多选项。
* 支持 expiry 和清理过期的对象。
* 线程安全。 可以从任何队列访问操作。
* 默认情况下同步。 还支持异步 API。
* 广泛的单元测试覆盖率和出色的文档。
* 支持iOS，tvOS和macOS。

##  二、安装

###  CocoaPods

在工程内 Podfile 文件中添加：

```
  pod'Cache ' 
```

### Carthage

在工程内 Cartfile 文件中添加：

```
  github “ hyperoslo / Cache ” 
```

您还需要在 copy-frameworks 脚本中添加 SwiftHash.framework 。


## 三、使用

### 存储

Cache 是基于[责任链模式](https://translate.googleusercontent.com/translate_c?act=url&depth=1&hl=zh-TW&ie=UTF8&prev=_t&rurl=translate.google.com&sl=en&sp=nmt4&tl=zh-CN&u=https://en.wikipedia.org/wiki/Chain-of-responsibility_pattern&xid=17259,15700023,15700124,15700149,15700186,15700190,15700201&usg=ALkJrhgwipa-UMmtMil2WrwPT0byiuk2rg)构建的，其中有许多处理对象，每个处理对象都知道如何执行 1 个任务并委托给下一个任务，因此您可以按照自己喜欢的方式编写存储。

目前支持以下存储

* MemoryStorage ：将对象保存到内存。
* DiskStorage ：将对象保存到磁盘。
* HybridStorage ：将对象保存到内存和磁盘，因此您可以在磁盘上获得持久化对象，同时在内存对象中快速访问。
* SyncStorage ：阻塞API，所有读写操作都在串行队列中调度，所有同步方式。
* AsyncStorage ：非阻塞API，操作在内部队列中进行调度以进行串行处理。 不应该同时进行读写操作。

虽然您可以自行决定使用这些存储，但您不必这样做。 因为我们还提供了一个方便的 Storage ，它使用了 HybridStorage ，同时通过 SyncStorage 和 AsyncStorage 公开同步和异步 API。

您需要做的就是使用 DiskConfig 和 MemoryConfig 指定所需的配置。 默认配置很好，但您可以自定义很多。

```Swift
let diskConfig = DiskConfig(name: "Floppy")
let memoryConfig = MemoryConfig(expiry: .never, countLimit: 10, totalCostLimit: 10)

let storage = try? Storage(
  diskConfig: diskConfig,
  memoryConfig: memoryConfig,
  transformer: TransformerFactory.forCodable(ofType: User.self) // Storage<User>
)
```

###  同步API

Storage 默认是同步的并且是 thread safe ，您可以从任何队列访问它。 所有同步功能都受 StorageAware 协议的约束。

```Swift
// Save to storage
try? storage.setObject(10, forKey: "score")
try? storage.setObject("Oslo", forKey: "my favorite city", expiry: .never)
try? storage.setObject(["alert", "sounds", "badge"], forKey: "notifications")
try? storage.setObject(data, forKey: "a bunch of bytes")
try? storage.setObject(authorizeURL, forKey: "authorization URL")

// Load from storage
let score = try? storage.object(forKey: "score")
let favoriteCharacter = try? storage.object(forKey: "my favorite city")

// Check if an object exists
let hasFavoriteCharacter = try? storage.existsObject(forKey: "my favorite city")

// Remove an object in storage
try? storage.removeObject(forKey: "my favorite city")

// Remove all objects
try? storage.removeAll()

// Remove expired objects
try? storage.removeExpiredObjects()
```

### 异步API

```Swift
storage.async.setObject("Oslo", forKey: "my favorite city") { result in
  switch result {
    case .value:
      print("saved successfully")
    case .error(let error):
      print(error)
  }
}

storage.async.object(forKey: "my favorite city") { result in
  switch result {
    case .value(let city):
      print("my favorite city is \(city)")
    case .error(let error):
      print(error)
  }
}

storage.async.existsObject(forKey: "my favorite city") { result in
  if case .value(let exists) = result, exists {
    print("I have a favorite city")
  }
}

storage.async.removeAll() { result in
  switch result {
    case .value:
      print("removal completes")
    case .error(let error):
      print(error)
  }
}

storage.async.removeExpiredObjects() { result in
  switch result {
    case .value:
      print("removal completes")
    case .error(let error):
      print(error)
  }
}
```

### 处理JSON响应

大多数情况下，我们的用例是从后端获取一些 json，在将 json 保存到存储器以供将来使用时显示它。 如果你使用像 Alamofire 或 Malibu 这样的库，你通常会以字典，字符串或数据的形式获得 json。

Storage 可以持久化 String 或 Data。您甚至可以使用JSONArrayWrapper 和 JSONDictionaryWrapper 将 json 保存到 Storage ，但我们更喜欢保留强类型对象，因为这些是您将用于在 UI 中显示的对象。 此外，如果 json 数据无法转换为强类型对象，那么保存它的重点是什么？ 

您可以在 JSONDecoder 上使用这些扩展来将 json 字典，字符串或数据解码为对象。

```Swift
let user = JSONDecoder.decode(jsonString, to: User.self)
let cities = JSONDecoder.decode(jsonDictionary, to: [City].self)
let dragons = JSONDecoder.decode(jsonData, to: [Dragon].self)
```

使用 Alamofire 执行对象转换和保存的 Alamofire

```Swift
Alamofire.request("https://gameofthrones.org/mostFavoriteCharacter").responseString { response in
  do {
    let user = try JSONDecoder.decode(response.result.value, to: User.self)
    try storage.setObject(user, forKey: "most favorite character")
  } catch {
    print(error)
  }
}
```


