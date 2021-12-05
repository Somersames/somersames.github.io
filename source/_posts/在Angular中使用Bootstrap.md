---
title: 在Angular中使用Bootstrap
date: 2018-05-29 19:28:03
tags: [web前端,Angular]
categories: [web前端,Angular]
---
在去年的时候短暂的接触了大概一个星期的 Angular 之后就再也没碰过了，今天突然想重新捡起 Angular 的相关知识，并且想将 Angular 结合 Bootstrap 一起使用。所以正好记录下一起结合使用的步骤。

初始化一个 Angular 的项目。初始化之后打开命令行，输入：
```shell
npm install jquery --save-dev
npm install bootstrap --save-dev
```
输入以上两条命令之后，在 `package.json` 中可以看到已经多出了 jquery 和 bootstrap 这两个库了。如下：
![](./第一步初始化.png)

当初始化完成之后会在 node_modules 中出现 bootstrap 和 jquery 这两个文件夹。

然后再将需要引入的 css 文件和 js 文件加入到 `.angular-cli.json`中，如下：
![](./第二步初始化.png)

最后再执行如下命令，将 JS 文件转化为 TypeScript 文件。
```xshell
npm install @types/jquery --save -dev

npm install @types/bootstrap  --save -dev
```

## 出错解决
如果安装完成之后提示的错误如下：
```
ERROR in ./node_modules/css-loader?{"sourceMap":false,"importLoaders":1}!./node_modules/postcss-loader/lib?{"ident":"postcss","sourceMap":false}!./node_modules/bootstrap/dist/css/bootstrap.min.css Module build failed: BrowserslistError: Unknown browser major at error 
```
此时需要将 node_modules 里面的 bootstrap 文件夹中的 package.json 中的 `"last 1 major version",">= 1%"` 删除。然后 输入 `npm start`就可以了。
