### 来自MonkeyDev的官方介绍

这是一个为越狱和非越狱开发人员准备的工具，包含四个模块：

**logos Tweak**

使用[theos](https://github.com/theos/theos/wiki/Installation)提供的`logify.pl`工具将`*.xm`文件转成`*.mm`文件进行编译，集成了`CydiaSubstrate`，可以使用`MSHookMessageEx`和`MSHookFunction`来`Hook` OC函数和指定地址

**CaptainHook Tweak**

使用[CaptainHook](https://github.com/rpetrich/CaptainHook/)提供的头文件进行OC 函数的Hook以及属性的获取

**Command-line Tool**

可以直接创建运行于越狱设备的命令行工具

**MonkeyApp**

这是自动给第三方应用集成Reveal、Cycript和注入dylib的模块，支持调试dylib和第三方应用，支持Pod给第三放应用集成SDK，只需要准备一个砸壳后的ipa或者app文件即可。

### 安装方法

1、通过pp助手找到计划hook的砸壳ipa包

2、使用class-dump获取app的所有类名/属性/方法的头文件集合

3、借助编辑器，如sublime，通过关键字全局搜索找到相关的方法

4、安装monkeydev

5、Xcode创建monkeyApp工程，引入app

6、编写hook代码和调试，找到确定的hook点修改逻辑

7、重签名运行（注意：monkeydev只支持真机运行）

参考

[使用mokeydev 修改iOS微信的运动步数教程](https://juejin.im/post/5ab330e1518825557b4ca5a8)

[Monkeydev官方教程](https://github.com/AloneMonkey/MonkeyDev/wiki)

### 常见的hook代码编写方式

* Logos Tweak

  ​	xxDylib.xm文件使用logos语法和runtime方法编写hook逻辑后，使用[theos](https://github.com/theos/theos/wiki/Installation)提供的`logify.pl`工具将`*.xm`文件转成`*.mm`文件进行编译。

  ​	相对CaptainHook语法，logos简单明了，但缺少关键字高亮和自动补齐，需注意检查拼写错误。

  **1、使用logos语法**

     a、包含了新增属性/方法的举例demo

  ```
  @interface CustomViewController
  
  @property (nonatomic, copy) NSString* newProperty;
  
  + (void)classMethod;
  
  - (NSString*)getMyName;
  
  - (void)newMethod:(NSString*) output;
  
  @end
  
  %hook CustomViewController
  
  + (void)classMethod
  {
  	%log;
  
  	%orig;
  }
  
  %new
  -(void)newMethod:(NSString*) output{
      NSLog(@"This is a new method : %@", output);
  }
  
  %new
  - (id)newProperty {
      return objc_getAssociatedObject(self, @selector(newProperty));
  }
  
  %new
  - (void)setNewProperty:(id)value {
      objc_setAssociatedObject(self, @selector(newProperty), value, OBJC_ASSOCIATION_RETAIN_NONATOMIC);
  }
  
  - (NSString*)getMyName
  {
  	%log;
      
      NSString* password = MSHookIvar<NSString*>(self,"_password");
      
      NSLog(@"password:%@", password);
      
      [%c(CustomViewController) classMethod];
      
      [self newMethod:@"output"];
      
      self.newProperty = @"newProperty";
      
      NSLog(@"newProperty : %@", self.newProperty);
  
  	return %orig();
  }
  
  %end
  
  ```

  b、通过分类增加系统类的方法

  ```
  //通过分类增加方法
  @interface NSString (MAMD5)
  - (NSString *)MAMD5;
  @end
  
  %hook NSString
  %new
  - (NSString *)MAMD5 {
  return @"replace ok!";
  }
  %end
  ```

  c、获取参数对象的变量/属性/方法调用

  ```
  //原方法：- (void)getWheel:(id)arg1 handler:(id)arg2；
  //需求点：希望在特定方法里获取到某参数对象的属性值
  //问题：无法通过argx.yy来获取属性值，会因无法识别yy属性值而报错
  //解决思路：声明需要hook的类和属性值，把参数的id类型替换成具体的类指针，则可以获取到属性值；同样获取成员变量和调用方法也是类似操作
  
  @interface Car: NSObject
  @property(nonatomic) unsigned int wheelNum;
  @end
  
  - (void)getWheel:(id)arg1 handler:(Car *)arg2 {
  NSLog(@"\)endWithResult:,%u",arg2.wheelNum);
  %orig;
  }
  ```

  ​    [logos语法说明](http://chuquan.me/2018/02/22/logos-syntax/)

     [logos语法 wiki中文翻译](https://juejin.im/post/5c00d61b6fb9a049c042c083)

     [logos语法wiki英文版](http://iphonedevwiki.net/index.php/Logos#.25c)

  2、调用runtime方法

  如获取类/调用方法/

  [支持runtime方法调用](https://developer.apple.com/documentation/objectivec/objective-c_runtime?language=objc)

* **CaptainHook Tweak**

  编写参考

  ```
  CHDeclareClass(CustomViewController)
  
  #pragma clang diagnostic push
  #pragma clang diagnostic ignored "-Wstrict-prototypes"
  
  //add new method
  CHDeclareMethod1(void, CustomViewController, newMethod, NSString*, output){
      NSLog(@"This is a new method : %@", output);
  }
  
  #pragma clang diagnostic pop
  
  CHOptimizedMethod0(self, NSString*, CustomViewController,getMyName){
      //get origin value
      NSString* originName = CHSuper(0, CustomViewController, getMyName);
      
      NSLog(@"origin name is:%@",originName);
      
      //get property
      NSString* password = CHIvar(self,_password,__strong NSString*);
      
      NSLog(@"password is %@",password);
      
      [self newMethod:@"output"];
      
      //set new property
      self.newProperty = @"newProperty";
      
      NSLog(@"newProperty : %@", self.newProperty);
      
      //change the value
      return @"AloneMonkey";
      
  }
  
  //add new property
  CHPropertyRetainNonatomic(CustomViewController, NSString*, newProperty, setNewProperty);
  
  CHConstructor{
      CHLoadLateClass(CustomViewController);
      CHClassHook0(CustomViewController, getMyName);
      
      CHHook0(CustomViewController, newProperty);
      CHHook1(CustomViewController, setNewProperty);
  }
  ```

  

  [CaptainHook可用方法定义](https://github.com/rpetrich/CaptainHook/blob/master/CaptainHook.h)

  