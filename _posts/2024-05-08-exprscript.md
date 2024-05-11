---
title: "exprscript 源码分析"
tags: [compiler, code]
---

<!--more-->

> 修订历史
> - 2024.05.08 创建笔记

## 语言特点
- 动态类型函数式语言
- 小核心, 丰富表达力
    - 面向对象编程
    - 协程
    - 异常
    - 结构/记录
    - 惰性求值
    - 多阶段求值
- 一等延续
- 支持词法作用域/动态作用域
- 全精度有理数
- 与 Python 的交互
- 尾调用优化
- 垃圾回收

## 实现
### Lex/Parse
语法与语义见 readme。

AST节点
```
class ExprNode:                 ...
class NumberNode(ExprNode):     ...
class StringNode(ExprNode):     ...
class IntrinsicNode(ExprNode):  ...
class VariableNode(ExprNode):   ...
class LambdaNode(ExprNode):     ...
class LetrecNode(ExprNode):     ...
class IfNode(ExprNode):         ...
class CallNode(ExprNode):       ...
class SequenceNode(ExprNode):   ...
class QueryNode(ExprNode):      ...
class AccessNode(ExprNode):     ...
```

### Runtime
#### 对象类型
```
class Value:                ...
class Void(Value):          ...
class Number(Value):        ...
class String(Value):        ...
class Closure(Value):       ...
class Continuation(Value):  ...
```

#### 运行模型
```
class Layer:
    def __init__(self, env: list[tuple[str, int]], expr: ExprNode, frame: Union[bool, None] = None, tail: Union[bool, None] = None):
        self.env = env  # frame 之间的 Layer 共享同一个 env
        self.expr = expr
        self.frame = False if frame is None else frame
        self.pc = 0     # 记录自身 Layer 的执行位置
        ...

class State:
    def __init__(self, expr: ExprNode):
        ...
        self.stack = [Layer(main_env, None, frame = True), Layer(main_env, expr)]
        self.value = None
        ...

    def execute(self) -> 'State':
        ...
        while True:
            layer = self.stack[-1]
            if layer.expr is None:
                return self
            elif type(layer.expr) == CallNode:
                if 2 < layer.pc <= len(layer.expr.arg_list) + 2:
                    layer.local['args'].append(self.value)
                if layer.pc == 0:
                    self.stack.append(Layer(layer.env, layer.expr.callee))
                    layer.pc += 1
                elif layer.pc == 1:
                    layer.local['callee'] = self.value
                    layer.local['args'] = []
                    layer.pc += 1
                elif layer.pc <= len(layer.expr.arg_list) + 1:
                    self.stack.append(Layer(layer.env, layer.expr.arg_list[layer.pc - 2]))
                    layer.pc += 1
                elif layer.pc == len(layer.expr.arg_list) + 2:
                    callee = layer.local['callee']
                    args = layer.local['args']
                    if type(callee) == Closure:
                        closure = callee
                        new_env = closure.env[:]
                        for i, v in enumerate(closure.fun.var_list):
                            addr = args[i].location if args[i].location != None else self._new(args[i])
                            new_env.append((v.name, addr))
                        ...
                        self.stack.append(Layer(new_env, closure.fun.expr, frame = True))
                        layer.pc += 1
                    ...
                else:
                    self.stack.pop()
                ...
```

- 深度优先遍历 AST
- 一个 ExprNode 对应一个 Layer
- Layer 栈形成调用栈，循环执行栈顶的 Layer，执行结束 pop Layer
- 栈帧内的多个 Layer 共享相同的 env
- main 和 closure call 的 Layer.frame 为 true

以 `(lambda (x y) { [(.+ x y)] } 1 2)` 为例，调用栈的变化如下
```
------------------------------------
- Layer(main_env, None, frame = True)
- Layer(main_env, CallNode)                 # (lambda (x y) { [(.+ x y)] } 1 2)
------------------------------------
- Layer(main_env, None, frame = True)
- Layer(main_env, CallNode)                 # (lambda (x y) { [(.+ x y)] } 1 2)
- Layer(main_env, LambdaNode)               # lambda (x y) { [(.+ x y)] } => result: Closure
------------------------------------
- Layer(main_env, None, frame = True)
- Layer(main_env, CallNode)                 # (lambda (x y) { [(.+ x y)] } 1 2)
- Layer(main_env, NumberNode)               # 1 => pop Layer, result: Number
------------------------------------
- Layer(main_env, None, frame = True)
- Layer(main_env, CallNode)                 # (lambda (x y) { [(.+ x y)] } 1 2)
- Layer(main_env, NumberNode)               # 2 => pop Layer, result: Number
------------------------------------
- Layer(main_env, None, frame = True)
- Layer(main_env, CallNode)                     # (lambda (x y) { [(.+ x y)] } 1 2)
- Layer(main_env, SequenceNode, frame = True)   # [(.+ x y)]
------------------------------------
- Layer(main_env, None, frame = True)
- Layer(main_env, CallNode)                     # (lambda (x y) { [(.+ x y)] } 1 2)
- Layer(main_env, SequenceNode, frame = True)   # [(.+ x y)]
- Layer(main_env, CallNode)                     # (.+ x y)
------------------------------------
- Layer(main_env, None, frame = True)
- Layer(main_env, CallNode)                     # (lambda (x y) { [(.+ x y)] } 1 2)
- Layer(main_env, SequenceNode, frame = True)   # [(.+ x y)]
- Layer(main_env, CallNode)                     # (.+ x y)
- Layer(main_env, VariableNode)                 # x => pop Layer, result: Number
------------------------------------
- Layer(main_env, None, frame = True)
- Layer(main_env, CallNode)                     # (lambda (x y) { [(.+ x y)] } 1 2)
- Layer(main_env, SequenceNode, frame = True)   # [(.+ x y)]
- Layer(main_env, CallNode)                     # (.+ x y)
- Layer(main_env, VariableNode)                 # y => pop Layer, result: Number
------------------------------------
- Layer(main_env, None, frame = True)
- Layer(main_env, CallNode)                     # (lambda (x y) { [(.+ x y)] } 1 2)
- Layer(main_env, SequenceNode, frame = True)   # [(.+ x y)]
- Layer(main_env, CallNode)                     # (.+ x y) => pop Layer, result: Number
------------------------------------
...
```

#### 一等延续
callcc 使用例子：
```
[
    (.put "main begin" "\n")
    (.call/cc lambda (k) {
        [
            (.put "sub begin" "\n")
            (k "anything you want to return, even the Closure 'k'")
            (.put "sub end" "\n")
        ]
    } )
    (.put "main end" "\n")
]

# 输出 'main begin\nsub begin\nmain end\n'
```
- callcc 的参数是闭包，该闭包的参数为 k
- 调用 callcc，程序会保存当前的 State.stack，并执行闭包体
- 在闭包体内执行时，当调用参数 k，则会替换回之前保存的 State.stack，表现为跳出闭包
- 调用参数 k 可以带上参数，作为跳出闭包的返回值

```
if intrinsic == '.call/cc':
    self.stack.pop()
    cont = Continuation(layer.expr.sl, deepcopy(self.stack))
    addr = cont.location if cont.location != None else self._new(cont)
    closure = args[0]
    self.stack.append(Layer(closure.env[:] + [(closure.fun.var_list[0].name, addr)], closure.fun.expr, frame = True))
    ...


if type(callee) == Continuation:
    # "self.value" already contains the lastevaluation result of the args
    self.stack = deepcopy(callee.stack)
    continue
    ...
```

更多 callcc 例子见异常、协程

#### 变量作用域
例子：

```
letrec (
  f = letrec (lex = 1 Dyn = 101) {
    lambda () { [(.put lex "\n") (.put Dyn "\n")] }
  }
) {
  letrec (lex = 3 Dyn = 303) { (f) }
}
# output: 1 303
```

- 动态作用域：变量首字母大写；词法作用域：变量首字母小写
- callcc、闭包执行会使用新 env（上一层 env 的词法作用域变量 + 参数）
- 新帧只保留上一帧的词法作用域变量
- 词法作用域变量查询：只查当前帧的 env；动态作用域变量逆序查询所有帧的 env

```
...
if type(layer.expr) == LambdaNode:
    self.value = Closure(filter_lexical(layer.env), layer.expr)
    self.stack.pop()
...

def lookup_env(name: str, env: list[tuple[str, int]]) -> Union[int, None]:
    ''' lexically scoped variable lookup '''
    for i in range(len(env) - 1, -1, -1):
        if env[i][0] == name:
            return env[i][1]
    return None

def lookup_stack(name: str, stack: list[Layer]) -> Union[int, None]:
    ''' dynamically scoped variable lookup '''
    for i in range(len(stack) - 1, -1, -1):
        if stack[i].frame:
            for j in range(len(stack[i].env) - 1, -1, -1):
                if stack[i].env[j][0] == name:
                    return stack[i].env[j][1]
    return None
```

#### 尾调用优化
闭包调用时，如果是尾调用，则 pop 当前帧的所有 Layer。*我认为 callcc 也可以尾调用优化*

```
...
if type(callee) == Closure:
    ...
    if layer.frame or layer.tail:
        while not self.stack[-1].frame:
            self.stack.pop()
        self.stack.pop()
    self.stack.append(Layer(new_env, closure.fun.expr, frame = True))
...

```

#### 垃圾回收
```
class State:
    def __init__(self, expr: ExprNode):
        self.store = []
        self.end = 0
        ...

class Layer:
    def __init__(self, env: list[tuple[str, int]], expr: ExprNode, frame: Union[bool, None] = None, tail: Union[bool, None] = None):
        self.env = env
        ...
```
- Value 放在堆中，env 存储堆 offset -> Value

```
def _traverse(self, value_callback: Callable, location_callback: Callable)-> None:
    # ids
    visited_values = set()
    # locations
    visited_locations = set()

    def traverse_value(value: Value) -> None:
        if id(value) not in visited_values:
            value_callback(value)
            visited_values.add(id(value))
            if type(value) == Closure:
                for _, loc in value.env:
                    traverse_location(loc)
            elif type(value) == Continuation:
                for layer in value.stack:
                    if layer.frame:
                        for _, loc in layer.env:
                            traverse_location(loc)
                    if layer.local:
                        for _, v in layer.local.items():
                            traverse_value(v)

    def traverse_location(location: int) -> None:
        if location not in visited_locations:
            location_callback(location)
            visited_locations.add(location)
            traverse_value(self.store[location])

    traverse_value(Continuation(SourceLocation(-1, -1), self.stack))
    traverse_value(self.value)
```
- Mark/Sweep 垃圾回收
    - 记录能被访问的 offset
    - 记录旧 offset 到新 offset 的映射
    - 更新 env 中的 offset

### 其他
#### FFI
使用例子：
```
# call ExprScript function from Python
state = es.State(es.parse(es.lex('(.reg "plus1" lambda (x) { (.+ x 1) })')))
print(state.execute().call_expr_function("plus1", [-6])) # -5

# call Python function from ExprScript
state = es.State(es.parse(es.lex('(.py "rev" (.str+ "na" "me"))')))
print(state.register_python_function("rev", lambda s: s[::-1]).execute()value.value) # "eman"
```
实现：
- py 调脚本：构建 ast
- 脚本调 py：将 py 函数加入 env
```
if intrinsic == '.py':
    py_args = []
    for i in range(1, len(args)):
        if type(args[i]) == Number:
            py_args.append(args[i].to_int(layer.expr.sl))
        ...
        
    ret = self.python_functions[args[0].value](*py_args)

if intrinsic == '.reg':
    self.stack[0].env.insert(0, (args[0].value, args[1]location if args[1].location != None else self._ne(args[1])))
    self.value = Void()

def call_expr_function(self, name: str, args: list[Union[str, int]]) -> Union[str, int]:
    sl = SourceLocation(-1, -1)
    callee = VariableNode(sl, name)
    arg_list = []
    for a in args:
        if type(a) == str:
            arg_list.append(StringNode(sl, a))
        ...

    self.stack.append(Layer(self.stack[0].env, CallNode(sl, callee,arg_list)))
    self.execute()
    if type(self.value) == String:
        return self.value.value
    ...
```

#### 全精度有理数
用分数表示
```
class NumberNode(ExprNode):
    def __init__(self, sl: SourceLocation, n: int, d: int):
        self.sl = sl
        g = gcd(abs(n), d)
        self.n = n // g
        self.d = d // g

    def add(self, other: 'Number') -> 'Number':
        n1 = self.n * other.d + other.n * self.d
        d1 = self.d * other.d
        g1 = gcd(abs(n1), d1)
        return Number(n1 // g1, d1 // g1)

    ...

def parse_number() -> NumberNode:
    ...
    if '/' in s:
        n, d = s.split('/')
        return NumberNode(token.sl, sign * int(n), int(d))
    elif '.' in s:
        n, d = s.split('.')
        return NumberNode(token.sl, sign * (int(n) * (10 ** len(d)) + int(d)), 10 ** len(d))
    else:
        return NumberNode(token.sl, sign * int(s), 1)
```

## 有趣的例子
### quine
quine 程序表示一个可以生成他自己的源代码的程序。[quine.expr](https://github.com/sdingcn/cvm.experimental/blob/495b7babdd1cc8f78d4b3f49333b716e4423ea95/test/quine.expr)
### 协程
获取当前位置的 continuation
```
getCC = lambda () {
    (.call/cc lambda (k) { (k k) })
}
```
获取 cc 后互相传递，具体见 [coroutines.expr](https://github.com/sdingcn/cvm.experimental/blob/c217060fb45182e298e55b6b257115aabb30937c/test/coroutines.expr)
### 异常
同样使用 callcc
[exception.expr](https://github.com/sdingcn/cvm.experimental/blob/c217060fb45182e298e55b6b257115aabb30937c/test/exception.expr)
### OOP
闭包 + 动态作用域
[oop.expr](https://github.com/sdingcn/cvm.experimental/blob/a50ca30f0df42250d7f5af8c1069b1ec57eac11d/test/oop.expr)
### 结构/记录
[bianary-tree.expr](https://github.com/sdingcn/cvm.experimental/blob/2f94e22a19429b726d0a3866dddc246094677c5b/test/binary-tree.expr)
[bianary-tree.expr](https://github.com/sdingcn/cvm.experimental/blob/2f94e22a19429b726d0a3866dddc246094677c5b/test/quicksort.expr)

为避免迷失在层层闭包中，可以这么类比
```
null = lambda () {        # 参数对应 struct 的成员变量，null 没有成员变量
  lambda (x) {            # 参数用来指定调用哪个成员函数
    if (eq x 0) then 1    # null 只有一个成员函数：判断是否为空
    else (exit)
  }
}
cons = lambda (head tail) {       # 参数对应 struct 的成员变量，cons 有成员变量 head 和 tail
  lambda (x) {                    # cons 有三个成员函数
    if (eq x 0) then 0            # 判断是否为空
    else if (eq x 1) then head    # 获取成员变量 head
    else if (eq x 2) then tail    # 获取成员变量 tail
    else (exit)
  }
}
```
于是调用 struct 实例是这样的
```
letrec (
  null = lambda () {        
    lambda (x) {            
      if (.== x 0) then 1   
      else (exit)
    }
  }
  cons = lambda (head tail) {       
    lambda (x) {                    
      if (.== x 0) then 0            
      else if (.== x 1) then head    
      else if (.== x 2) then tail    
      else (exit)
    }
  }
  isnull = lambda (list) {
    (list 0)
  }
  instanceA = (null)
  instanceB = (cons 1 (null))
)
{
  [
    (.put (isnull instanceA) "\n")
    (.put (isnull instanceB) "\n")
  ]
}
```