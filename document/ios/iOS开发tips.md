### 1. 忽略警告

```objective-c
#pragma clang diagnostic push
#pragma clang diagnostic ignored "警告名称"
 
// 被夹在这中间的代码针对于此警告都会忽视不显示出来
 
//常见警告的名称
//1.声明变量未使用  "-Wunused-variable"
//2.方法定义未实现  "-Wincomplete-implementation"
//3.未声明的选择器  "-Wundeclared-selector"
//4.参数格式不匹配  "-Wformat"
//5.废弃掉的方法     "-Wdeprecated-declarations"
//6.不会执行的代码  "-Wunreachable-code"
//7.忽略在arc 环境下performSelector产生的 leaks 的警告 "-Warc-performSelector-leaks"
//8.忽略类别方法覆盖的警告 "-Wobjc-protocol-method-implementation"（修复开源库bug，覆盖开源库方法时会用到）
 
#pragma clang diagnostic pop
```



