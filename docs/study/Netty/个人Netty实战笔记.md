# 个人Netty实战笔记

> 这个是自己使用Netty开发的时候，踩坑等的笔记。
>
> 有些东西暂时不方便公开，所以不会太详细地介绍

## 1. 后端的protobuf的java类转换JSON传输到前端Dart后转protobuf的dart类读取

> 这个是我自己踩过的坑之一。因为某些场景中需要直接把.proto生成的java类包装成JSON通过HTTP请求返回给Dart。但是我试了很多种方式，发现都会出现奇怪的错误，但是网上没找到比较好的解决方法。
>
> 最后还是自己各种尝试后，试出来了一种可以在Dart识别出JSON中的proto类的数据的方式。
>
> 不过，这种场景终究是少数情况。我这里protobuf除这个情况以外都是用UDP数据报传输的，然后再字节数组转proto类。

​	我自己试过用protobuf转JSON的`protobuf-java-format`和`protobuf-java-util`来将protobuf转成JSON然后传输到Dart。而且其实protobuf生成的Java类的toString()，我自己打印到控制台，看着也是JSON格式的。这几种JSON的方式我都试过了，但是直接这么JSON后传到Dart都会出现一些奇怪的错误。再者也试过一些其他的操作，这里不一一列举，最后还是转成UTF-8编码的String传输得以成功。

​	出现的错误比如有：`Protocol message end-group tag did not match expected tag`、`Protocol message tag had invalid wire type.`等。



1. Java后端(这里不方便公开代码，只大致描述)

   1. testController.java

      ​	这里假设你有一个Java类叫作ProtobufJavaPo，里面存了protobuf序列化数据的String形式数据的List

      ```java
      // Controller
      @RestController
      public class testController{
          
          @GetMapping("/getTest")
          public ProtobufJavaPo getTest() {
              ProtobufJavaPo protobufJavaPo = new ProtobufJavaPo();
              //... 省略给protobufJavaPo装填数据的过程
              return protobufJavaPo; // 这些都是我自己捏造的，就大概这个意思
          }
      
      }
      ```

   2. ProtobufJavaPo.java

      ​	这里假设你的.proto生成的java类为TestProto.java，然后你用其Builder生成了对应的对象叫作testProto

      ```java
      // 这里用到了IDEA的Lombok插件
      @Data
      @EqualsAndHashCode(callSuper = false)
      @Accessors(chain = true)
      @JsonInclude(JsonInclude.Include.NON_NULL)
      public class ProtobufJavaPo implements Serializable {
          private static final long serialVersionUID = 1L;
      
          // protobuf序列化数据的 byte数组 使用UTF-8编码 转String
          // 这里假设testProto就是.proto文件生成的java类的对象
          // udpProtoList.add(new String(testProto.toByteArray(), StandardCharsets.UTF_8));
          List<String> udpProtoList;
      }
      ```

2. Dart前端/移动端(同样都是模拟的，讲个大概)
   1. HttpDart.dart

      ​	这里假设dart中，dart的.proto文件生成dart类为TestProtoDart.dart。Dart识别HTTP中JSON数据的String形式的proto数据，并重新转化成dart生成的TestProtoDart.dart。

      ```dart
      import 'package:dio/dio.dart';// 社区的http请求包。可以去(dev.pub)这个网站查找
      // 其余依赖我就不说了
      // 这里DartMess是假装有这么一个dart类作为返回值，其含有List<TestProtoDart> protoList;
      class HttpDart {
      
          // HTTP获取JSON数据，解析里面的String字符串形式的protobuf数据
          // 然后转成dart中.proto生成的TestProtoDart.dart类。(这些名称什么的全都假设的)
          Future<DartMess> getProtoFromJSON() async {
      
              var response = await dio.get('XXXXXXXX(host:port)/getTest');
              List dynamicList = response.data['udpProtoList'];
              if(dynamicList!=null){
                  var dartMess = DartMess(); 
                  List<TestProtoDart> protoDartList = List();
                  dynamicList.forEach((f)=>{protoDartList.add(TestProtoDart.fromBuffer(utf8.encode(f)))});
                  dartMess.protoList = protoDartList;
                  return dartMess;
              }
              return null;
          }
      }
      ```

   2. DartMess.dart

      ```dart
      class DartMess {
          List<TestProtoDart> protoList; // TestProtoDart假设是.proto文件生成的dart类
      }
      ```

      