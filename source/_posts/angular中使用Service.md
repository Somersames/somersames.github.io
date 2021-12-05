---
title: angular中使用Service
date: 2018-05-30 22:52:12
tags: [Angular,web前端]
categories: [web前端,Angular]
---
今天使用 Angular 的时候，看到书中封装了自己得一个 LoggerService ，然后自己也想尝试下，顺便也写下这个记录。

## 新建一个Service得ts文件
在 app 目录下建立一个 ts 文件，如下：
```ts
export class LoggerService{
    info(msg : any) {
        console.log(msg);
    }
    warn(msg: any){
        console.warn(msg);
    }
    error(msg: any){
        console.error(msg);
    }
}
```
这个类就是封装了一层日志打印得函数，当然在这里也可以封装一些其他得函数。

## 将改服务添加至module

修改 app.module.ts 文件，将这个日志服务加入到 app.module.ts 这个文件中。
```ts
import { LoggerService } from './loggerService';

providers: [PersonService,LoggerService],

```


## 在组件中使用
如果需要在组件中使用这个服务的话，可以直接在组件得 Component 中 import 即可。
```ts
import { LoggerService } from  '../loggerService'

//然后在构造函数中加入

 constructor(private personService :PersonService,private logger: LoggerService) {
}

// 最后方法中使用：
 printItem(){
    console.log(this.item)
    this.logger.info("Logger服务得日志");
    this.emitor.emit("测试回发数据")
  }
```
## 结束