---
weight: 1
title: "Typescript Chanllenges"
date: 2022-04-02
lastmod: 2020-04-02T21:40:32+08:00
draft: true
author: "chenzheqi"
# authorLink: "https://dillonzq.com"
description: "挑战TypeScript的类型体操"

tags: ["前端", "TypeScript"]
categories: ["技术"]

lightgallery: true

toc:
  auto: true
---

### Pick

``` typescript
interface Todo {
  title: string
  description: string
  completed: boolean
}

type MyPick<T, K extends keyof T> = {[key in K]: T[key]}

type TodoPreview = MyPick<Todo, 'title' | 'completed'>
```

```typescript
type MyExclude<T, V> = T extends V ? never : T;

type T0 = MyExclude<"a" | "b" | "c", "a">;

type T1 = MyExclude<"a" | "b" | "c", "a" | "b">;

type T2 = MyExclude<string | number | (() => void), Function>
```

```typescript
interface Todo {
  title: string
  description: string
}

type MyReadonly<T> = {readonly [k in keyof T]: T[k]}

const todo: MyReadonly<Todo> = {
  title: "Hey",
  description: "foobar"
}

todo.title = "Hello" // Error: cannot reassign a readonly property
todo.description = "barFoo" // Error: cannot reassign a readonly property
```

```typescript
const tuple = ['tesla', 'model 3', 'model X', 'model Y'] as const

type TupleToObject<T extends readonly (string | number | symbol)[]> = {[key in T[number]]: key}

type result = TupleToObject<typeof tuple>
```

```typescript
type arr1 = ['a', 'b', 'c']
type arr2 = [3, 2, 1]
type arr3 = []

type First<T> = T extends unknown[] ? T[0] : never;

// type First<T> = T extends [infer F, ...infer R] ? F : never

type head1 = First<arr1> // expected to be 'a'
type head2 = First<arr2> // expected to be 3
type head3 = First<arr3>
```
