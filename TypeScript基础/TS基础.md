# TypeScript基础

## TypeScript安装与使用
__安装TypeScript:__

```
//全局安装
npm install -g typescript

//项目中安装
npm install -D typescript
```

__使用TypeScript:__
新建文件demo1.ts

```
function test(){
  let str: string = "learn TypeScript"
  console.log(str)
}
test();
```

在终端运行tsc demo1.ts，将ts文件转换成js文件demo1.js(因为node不能直接运行ts文件). <br>
终端运行node demo1.ts，即可看到运行结果

`"learn TypeScript"`

__使用ts-node:__
每次都要先用tsc转换成js文件比较麻烦，可以安装ts-node插件: 

```
//全局安装
npm install -g ts-node

//在项目中安装
npm install -D ts-node

//还需安装一个运行库
npm install -D tslib @types/node
```

这样可以直接运行ts文件：ts-node demo.js