---
layout: post
title: TypeScript-Guide
date: 2022-12-01 16:45:30.000000000 +09:00
categories: [typescript]
tags: [typescript] 
---

## 简介
- ts 是 js 的超集，可以认为 ts 是有类型的 js
- ts 不能直接被运行，需要编译成 js 后再执行
- ts 可以编译成任意版本的 js

## TypeScript 开发环境搭建
1. 安装 Node.js
2. 安装 TypeScript，$ npm install -g typescript
- 查看 ts 版本 $ npm view typescript version
3. 运行 .ts 文件
- $ ts-node hello.ts

## 类型声明
a 的类型设置为了 number 数字类型，以后只能是数字类型

```ts
let a: number;
a = 100;
console.log(a);

let b: string;
b = "hello";
console.log("b is", b);
```

如果变量声明和赋值时同时进行的，ts 可以自动对变量进行类型检测，这样就不用声明变量的类型了，跟 swift 类型推断一样啊
```ts
let c = false; // c 就是 boolean 类型

function sum(a: number, b: number) {
  return a + b;
}

函数参数声明类型，返回值声明类型
function sum1(a: number, b: number): number {
  return a + b;
}
let result = sum1(1, 3);
console.log(sum(1, 3));
console.log(result);
```

## TypeScript 中的类型

### 使用字面量进行类型声明
```ts
// 常量使用
// a1 的类型是 10, 不能是其他数字了, 这样类似常量了
let a1: 10;
a1 = 10;
// a1 = 11; error

// 类似枚举使用
let b1: "male" | "female";
b1 = "male";
b1 = "female";

// 联合类型使用
let c1: boolean | string;
c1 = true;
c1 = "hello";
```

### any 和 unknown 类型

- any 类型，即任意类型。一个变量声明为 any 后。相当于对该变量关闭了 ts 类型检测，变回 js 了。使用 ts 时不建议使用 any 类型，建议少用。
```ts
let d: any;
// let d; 这样 b 也是 any 类型
d = 10;
d = "hello";
```
- unknown 表示未知类型
```ts
let e: unknown;
e = 10;
e = "hello";
e = true;
```

-  unknown vs any

```ts
let s: string;
// d 的类型是 any 它可以赋值给任意类型，意思就是可以嚯嚯别人
s = d;

// unknown 不嚯嚯别人 这里报错
// s = e; error

// 解决方案一
if (typeof e === "string") {
  s = e;
}

// 解决方案二 类型断言，告诉解析器变量实际类型
s = e as string; // 类型断言 写法一
s = <string>e; // 类型断言 写法二
```

### void 和 never
- void 空值，即没有值或 undefined，以函数为例，表示没有返回值
```ts
function fn(): void {}
```
- never 没有值，即不能是任何值，就是什么都没有，连空都没有
```ts
function fn2(): never {
  throw new Error("报错了!");
}
```

### object
object 表示一个 js 对象, 用的少
```typescript
let o: object;
o = {};
```
指定对象中需要包含哪些属性。语法：{属性名: 属性值 ...}，属性名后有 ？ 表示属性名是可选的。
```typescript
let o1: { name: string; age?: number };
o1 = { name: "mike" };
```
propName 表示属性名，它是 string 类型，属性值 any 表示可以是任意类型。c2 表示对象必须有一个 name 属性，其他属性都可以
```typescript
let c2: { name: string; [propName: string]: any };
```

### Function

设置函数结果的类型声明：语法：(形参：类型，形参：类型 ...) => 返回值类型。d1 是参数为 number 的函数 返回值也是 number
```typescript
let d1: (a: number, b: number) => number;
```

### Array 数组
数组的两种声明方式

```
1 类型[]
2 Array<类型>
```

```typescript
let list: number[]; // string[] 表示字符串数组
list = [1, 2, 3];

let g: Array<string>;
g = ["1", "2"];
```

### 元组，就是固定长度的数组
语法 [类型，类型]
```typescript
// 声明两个元素的元组
let t: [string, string];
t = ["hello", "abc"];
```

### 枚举
```typescript
enum Gender {
  Male = 0,
  Female = 1,
}

let gender: Gender;
gender = Gender.Male;
```

### & 表示同时满足
```typescript
let j: { name: string } & { gender: Gender };
j = { name: "hello", gender: Gender.Female };
```

### 类型别名
```typescript
type myType = 1 | 2 | 3 | 4 | 5;
let k: myType;
k = 3;
```


## 参考
[入门](https://juejin.cn/post/7018805943710253086)

