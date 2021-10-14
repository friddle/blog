---
title: Flutter引入C/C++小节(Flutter With C/C++)
date: 2021-10-13 15:27 +0000
categories:
  - Skill
tags:
  - Flutter
---

具体怎么交互参考:https://flutter.dev/docs/development/platform-integration/c-interop

## 基础文档核心描述
```json
native_add.c

#include <stdint.h>

extern "C" __attribute__((visibility("default"))) __attribute__((used))
int32_t native_add(int32_t x, int32_t y) {
    return x + y;
}
//记得C++等要提供链接需要 extern "C"
```

### C++代码丢到ios/Classes/下面。为什么要丢到ios.不是通用目录。因为xCode的编译限制的IDE添加代码地址 。  
### android/ 写一个CMakeList.txt   

```
cmake_minimum_required(VERSION 3.4.1)  # for example
add_library( native_add //名字
             SHARED　//类型。共享链接
             ../ios/Classes/native_add.cpp
　　　　　　　　../ios/Classes/native_add2.cpp //多个文件再加一行
)
```
### 然后再添加
```
android {
  externalNativeBuild {
    cmake {
      path "../CMakeLists.txt"　//自动生成的flutter的android包在app下面。注意相对关系
    }
  }
}
```
### 引入ffi库

```
import 'dart:ffi'; // For FFI
import 'dart:io'; // For Platform.isX

final DynamicLibrary nativeAddLib = Platform.isAndroid
    ? DynamicLibrary.open("libnative_add.so")
    : DynamicLibrary.process();
```
    
### 查找类型
```
final int Function(int x, int y) nativeAdd =
  nativeAddLib
    .lookup<NativeFunction<Int32 Function(Int32, Int32)>>("native_add")
    .asFunction();
```    
## 高阶用法
这文档肯定不够用的。那么提示下常见问题。  
首先推荐下这个库[​pub.dev/packages/ffigen](https://pub.dev/packages/ffigen)  
这里可以自动生成Ｃ的调用Dart类。

然后大部分ffi的高阶调用方法都可以再这里找到：
```
dart-lang/samples
​github.com/dart-lang/samples/tree/master/ffi
```

然后讲下一些常见问题。

## 类型系统。

|C类型|dart|dart_ffi|intro
|---|---|---|--|
|int|int|ffi.Int32||   
|long||int|ffi.Int64||	  
|double|double|ffi.Double||
|unsigned|int|int|ffi.Uint32|unsinged基本就是前面开始为U|  
|char*(string)|ffi.Pointer<Utf8>|ffi.Pointer<Utf8>|转换是String转换到Utf8.toUtf8(string);String转换回来是Utf8.fromUtf8(pointer)|  
|char*(数组)|ffi.Pointer<Uint8>|ffi.Pointer<Uint8>||  



### 推荐查找函数写成这个

```
import 'dart:ffi' as ffi; // For FFI
typedef native_add_c_fun=ffi.Int32 Function(ffi.Int32, ffi.Int32)
typedef native_add_dart_fun=int Function(int, int)

final nativeAdd =
  nativeAddLib
    .lookup<ffi.NativeFunction<native_add_c_fun>>("native_add")
    .asFunction<native_add_dart_fun>();
struct结构体操作

struct.h

struct Coordinate
{
    double latitude;
    double longitude;
};
struct Coordinate *create_coordinate(double latitude, double longitude);
struct.c

struct Coordinate *create_coordinate(double latitude, double longitude)
{
    struct Coordinate *coordinate = (struct Coordinate *)malloc(sizeof(struct Coordinate));
    coordinate->latitude = latitude;
    coordinate->longitude = longitude;
    return coordinate;
}
struct.dart

class Coordinate extends Struct {
  @Double()
  double latitude;

  @Double()
  double longitude;

  factory Coordinate.allocate(double latitude, double longitude) =>
      allocate<Coordinate>().ref
        ..latitude = latitude
        ..longitude = longitude;
}


typedef create_coordinate_func = Pointer<Coordinate> Function(
    Double latitude, Double longitude);
typedef CreateCoordinate = Pointer<Coordinate> Function(
    double latitude, double longitude);

final createCoordinatePointer =
      dylib.lookup<NativeFunction<create_coordinate_func>>('create_coordinate');
final createCoordinate =
      createCoordinatePointer.asFunction<CreateCoordinate>();
final coordinatePointer = createCoordinate(1.0, 2.0);
final coordinate = coordinatePointer.ref;
print('Coordinate: ${coordinate.latitude}, ${coordinate.longitude}');
```
这是基础的Struct对象操作

### 小Tips
#### 添加AndroidLog支持
```
cmake_minimum_required(VERSION 3.4.1)  # for example

find_library(log-lib　log)
add_library( native_add //名字
             SHARED　//类型。共享链接
             ../ios/Classes/native_add.cpp
)
target_link_libraries(motion_detect
                      ${log-lib})
```
在C文件里面使用
```
#ifdef ANDROID
   #include <android/log.h>
   __android_log_print(ANDROID_LOG_ERROR, ANDROID_TAG,"native_add %d %d",x,y);
#endif
```
## 最后
本人的ffi/dart/c都是二流到三流水准。所以极有可能有些纰漏。有错误请指正。   
主要是搜索这个主题的时候发现较少。所以记录下自己问题解决的过程。给自己留一个思路。   