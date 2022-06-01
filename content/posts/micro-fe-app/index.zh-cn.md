---
weight: 1
title: "微前端-Qiankun"
date: 2022-04-02
lastmod: 2020-04-02T21:40:32+08:00
draft: true
author: "chenzheqi"
# authorLink: "https://dillonzq.com"
description: "介绍微前端和qiankun框架"

tags: ["前端", "微前端", "qiankun"]
categories: ["技术"]

lightgallery: true

toc:
  auto: true
---

<!-- markdownlint-disable no-duplicate-heading no-inline-html -->

## Qiankun

### 做了哪些事

创建沙箱

### 如何加载子应用

一个微前端框架无非就是做两件事：注册微应用，启动主应用。而这两件事基本都交给了 `single-spa` 实现了，qiankun 只是在这些API的基础上，对少量参数做了一些易用性的增强。

- 优化微应用入口参数：`single-spa` 默认要求传入一个包含所有声明周期函数的对象，或者一个返回这个对象的Promise。qiankun利用

### 怎么实现沙箱

#### Legacy Sandbox

一个Legacy Sandbox对象，包含以下几个属性：

| 属性名                                 | 意义                                                                  |
| -------------------------------------- | --------------------------------------------------------------------- |
| name                                   | 沙箱名                                                                |
| proxy                                  | 沙箱的代理对象，对外暴露的window                                      |
| globalContext                          | 全局上下文，Origin Window                                             |
| type                                   | 沙箱类型                                                              |
| sandboxRunning                         | 沙箱是否运行的状态                                                    |
| latestSetProp                          | 最后一个添加到window对象上的属性名                                    |
| addedPropsMapInSandbox                 | 沙箱期间新增的全局变量                                                |
| modifiedPropsOriginalValueMapInSandbox | 沙箱期间更新的全局变量                                                |
| currentUpdatedPropsValueMap            | 持续记录更新的(新增和修改的)全局变量的 map，用于在任意时刻做 snapshot |

##### 创建沙箱

1. 创建一个没有原型链的对象`const fakeWindow = Object.create(null)`
2. 代理这个对象，并将代理对象赋值给proxy属性
    1. set
        1. set新属性，addedPropsMapInSandbox新增这次新增的键值对
        2. 首次更新orgin window上已有的属性，modifiedPropsOriginalValueMapInSandbox缓存原始属性值
        3. 更新currentUpdatedPropsValueMap
        4. 同步更新到Origin Window
        5. 更新latestSetProp
    2. get
        1. 单独处理特殊属性，防止沙箱逃逸（top, parent, window, self)
        2. 查找orign window上的属性值，如果是函数，且不是构造函数，做一下特殊处理：重新绑定this；赋值原函数的属性值；单独处理函数的prototype属性；重新定义toString方法
        3. 返回属性值
    3. 处理其他trap方法：has; getOwnPropertyDescriptor; defineProperty;

##### 激活(active)沙箱

将`modifiedPropsOriginalValueMapInSandbox`中的所有属性重新赋值到globalContext上

##### 休眠(inactive)沙箱

通过`modifiedPropsOriginalValueMapInSandbox`重置所有的原始值

通过`addedPropsMapInSandbox`删除新增的属性值

#### Proxy Sandbox

一个Proxy Sandbox对象包含以下属性

| 属性名          | 意义                               |
| --------------- | ---------------------------------- |
| name            | 沙箱名                             |
| proxy           | 沙箱的代理对象，对外暴露的window   |
| globalContext   | 全局上下文，Origin Window          |
| type            | 沙箱类型                           |
| sandboxRunning  | 沙箱是否运行的状态                 |
| latestSetProp   | 最后一个添加到window对象上的属性名 |
| updatedValueSet | window 值变更记录                  |

##### 创建沙箱

1. 创建一个Fake Window
    1. 创建一个空对象{} fakeWindow
    2. 复制Origin Window对象上不可配置的属性到fakeWindow上

        这类属性不多，下面显示的是你的这个窗口下，满足要求的属性

        <pre style="white-space: pre-line;">
            <code id="a-1"></code>
        </pre>
        {{< script >}}
        const propArr = Object.getOwnPropertyNames(window)
            .filter((p) => {
            const descriptor = Object.getOwnPropertyDescriptor(window, p);
            return !descriptor?.configurable;
        });
        document.getElementById('a-1').innerText = propArr.join(', ');
        {{< /script >}}

2. 代理Fake Window对象，并复制给proxy
    1. set
        1. 设置当前应用为激活中，存在全局变量中
        2. 如果Fake Window对象上没有set的属性，但是Origin Window上有：复制PropertyDescriptor到Fake Window对象上
        3. 否则，更新Fake Window上的属性
        4. 少量白名单属性直接穿透设置到Origin Window上
        5. updatedValueSet新增属性
        6. 更新latestSetProp
    2. get
        1. 设置当前应用为激活中，存在全局变量中
        2. 通过`{{< link "https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Symbol/unscopables" Symbol.unscopables >}}`控制部分属性可以在with环境下逃逸

            ```js
            {
                undefined: true,
                Array: true,
                Object: true,
                String: true,
                Boolean: true,
                Math: true,
                Number: true,
                Symbol: true,
                parseFloat: true,
                Float32Array: true,
                isNaN: true,
                Infinity: true,
                Reflect: true,
                Float64Array: true,
                Function: true,
                Map: true,
                NaN: true,
                Promise: true,
                Proxy: true,
                Set: true,
                parseInt: true,
                requestAnimationFrame: true,
            };
            
            ```

        3. 拦截`window` / `self` / `globalThis` 的访问，返回 `proxy` 对象
        4. 如果主应用在iframe下运行，访问`top`/parent时，允许逃逸，直接访问Origin Window的属性
        5. hasOwnProperty属性同时判断是否在Fake Window和Origin Window上
        6. document和eval都直接返回原始对象
        7. 其它属性从Origin Window或Fake Window上取，此处有一点[性能优化](https://github.com/umijs/qiankun/blob/412113d078dfdf7f134a42c2dc3f103e6b4d946f/src/sandbox/proxySandbox.ts#L292)的处理
        8. 处理函数的访问，和上面一致；对于fetch方法，需要[调整this](https://github.com/umijs/qiankun/blob/412113d078dfdf7f134a42c2dc3f103e6b4d946f/src/sandbox/proxySandbox.ts#L303)
    3. defineProperty / getOwnPropertyDescriptor

        调用`defineProperty`默认都是在Fake Window上定义。如果通过`Object.getOwnPropertyDescriptor(window, p)`获取属性描述符后，再调用`defineProperty`，需要定义到正确的**window**对象上。

        因此在`getOwnPropertyDescriptor`的trap中，需要判断并缓存属性到底是在哪个**window**上。

        > Descriptor must be defined to native window while it comes from native window via Object.getOwnPropertyDescriptor(window, p), otherwise it would cause a TypeError with illegal invocation.

##### 激活(active)沙箱

将当前沙箱的sandboxRunning置为true，同时全局已激活沙箱数+1。

##### 休眠(inactive)沙箱

将当前沙箱的sandboxRunning置为false，同时全局已激活沙箱数-1。

如果沙箱全部休眠了，需要[清除](https://github.com/umijs/qiankun/blob/412113d078dfdf7f134a42c2dc3f103e6b4d946f/src/sandbox/proxySandbox.ts#L189)Origin Window上的白名单属性。

## 如何找到子应用的生命周期函数

[https://github.com/umijs/qiankun/blob/b1030783dfaacde9ab8c3d941855a111db329302/src/loader.ts#L206](https://github.com/umijs/qiankun/blob/b1030783dfaacde9ab8c3d941855a111db329302/src/loader.ts#L206)

- 解析子应用html内容
- 如果有入口（entry）script，使用eval执行这段代码后，拿到global对象（通常是window.proxy）上最后一个属性值，判断这个属性值对象中是否有三个生命周期函数
- 如果html中没有设置入口script，则默认取html里最后一个script为entry，后续逻辑和上面一致
- 如果上面2个判断都没有找到三个钩子函数，则获取并判断global[appName]的属性值
- 如果仍未找到，报错

> 这就是为什么qiankun推荐在Vue、React等项目的入口文件中里添加三个export function，并且打包成umd格式。

## Qiankun和Vite

### Sourcemap

vite生态下有一个叫vite-plugin-qiankun的插件，支持将项目（子应用）集成到qiankun，这款插件做了什么呢？只说构建阶段吧，它修改了index.html里的`<script modules>`标签，将标签里的src值提取出来，转换成`import(`${原src}`).finally(() ⇒ {...})`。finally块的代码将qiankun子应用的3个声明周期函数暴露了出去，这里不详述。最主要的区别是，这么修改后，**原来的外链脚本(external script)变成了内联脚本(inline script)**。

qiankun本身处理这两类script，并没有什么不同，无非是外链脚本首先fetch下来，然后将code放在沙箱里运行，而内联脚本则跳过fetch阶段，直接扔到沙箱里运行。从执行的结果来看，是一样的。**但是对于sourcemap来讲，就不太一样了。**

在沙箱里运行外链脚本的内容时，其实是在原代码的外层包裹了下面这个上下文：

```jsx
// 补上GitHub代码地址
// 这里插入的sourceUrl是外链url，不是sourcemapURL
;(function(window, self, globalThis){with(window){;${scriptText}\n${sourceUrl}}}).bind(window.proxy)(window.proxy, window.proxy, window.proxy);
```

这样相当于原代码被劫持修改了，而sourcemap没有同步更新，如果js代码发生了运行时错误，sourcemap肯定定位不到正确的位置。对于Sentry这些依赖sourcemap做错误追踪的系统来说，就无法发挥其应有的作用。

那么内联脚本的执行后会怎么样呢，其实对于qiankun来说，代码本身是否内联，并无区别，但是vite构建项目生成的是ESModule的产物，也就是上面提到的，通过import引入js文件。那么即使被qiankun包了一层上下文，**最终js文件仍然是由浏览器发起下载和执行的**。这意味着，js代码如果有运行时错误，浏览器能够根据文件底部的sourceMappingURL找到正确的位置。

<aside>
💡 归根到底，qiankun只解析了入口html里的script标签，通过动态import引入的脚本，并没有被乾坤劫持。

</aside>

### ESModule

qiankun诞生之初，是没有考虑兼容Module Script的。

## Qiankun和Sentry
