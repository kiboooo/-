Gson解析Json流程
---
> Gson 是一个强大的序列化和反序列化的一个json库

解析Json流程：
1. 通过抓包分析和API文档，了解获取到的Json结构类型；
2. 利用结构类型，结合相应的 Json 中的变量名，构建相应的 JavaBean 类；
3. 通过Gson的实例变量，调用 fromJson 方法，把需要解析的Json内容和储存结果的JavaBean类传递进去；
```java
public <T> T fromJson(JsonReader reader, Type typeOfT) throws JsonIOException, JsonSyntaxException {
    boolean isEmpty = true;
    boolean oldLenient = reader.isLenient();
    reader.setLenient(true);
    try {
      reader.peek();
      isEmpty = false;
      TypeToken<T> typeToken = (TypeToken<T>) TypeToken.get(typeOfT);
      TypeAdapter<T> typeAdapter = getAdapter(typeToken);
      T object = typeAdapter.read(reader);
      return object;
```
4. 在fromJson 中 根据 相应的JavaBean类 通过TypeAdapterFactory工厂类生成相应的Adapter；
5. 由Adapter调用反序列化的方法 read 通过反射加递归，一层层的解析Json数据，最后返回JavaBean类；
6. 在解析的过程中，获取Json中的键值对信息，根据键名寻找对象中对应的字段 ；
```java
@Override public T read(JsonReader in) throws IOException {
     // 校验，数据为空的时候返回null
      if (in.peek() == JsonToken.NULL) {
        in.nextNull();
        return null;
      }
      // 创建实例，constructor由ConstructorConstructor创建，封装了实例创建的过程
      T instance = constructor.construct();

      try {
        in.beginObject();
        //　处理json对象中的所有属性
        while (in.hasNext()) {
          // 获得属性名
          String name = in.nextName();
          // 根据属性名获得BoundField对象
          BoundField field = boundFields.get(name);
          if (field == null || !field.deserialized) {
            in.skipValue();
          } else {
            // 处理当前属性
            field.read(in, instance);
          }
        }
      } catch (IllegalStateException e) {
        throw new JsonSyntaxException(e);
      } catch (IllegalAccessException e) {
        throw new AssertionError(e);
      }
      in.endObject();
      return instance;
    }
```
7. 如果字段是非基本类型，则回到上一步继续解析；
8. 字段是基本类型，则把值信息转化为基本类型返回 ，并通过反射赋值给结果JavaBean；



Gson的解析过程就是对Json结构的字符串，通过序列化和反序列的解析；
在Gson对象中：
+ 内建了解析不同 Java 类型的 TypeAdpater 分别处理不同类型的 Json 结构；
+ 在 fromJson, toJson 方法中调用 Adapter 的 read（反序列化）, write （序列化）方法递归的解析出不通层次 json 值；

> 总结：
> Gson反序列化的过程本质上是一个递归过程。当对其中一个字段进行解析时，其值如果是花括号保存的对象，则递归解析该对象；其值如果是数组，则处理数组后递归解析数组中的各个值。递归的终止条件是反序列的字段类型是java的基本类型信息。
