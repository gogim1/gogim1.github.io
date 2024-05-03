---
title: "Unification & Type inference"
tags: [compiler, notes]
---

<!--more-->

> 修订历史
> - 2024.05.03 创建笔记


## Unification
合一算法（Unification Algorithm）是一种用于求解类型约束方程的算法，它用于确定类型变量的具体类型，使得方程得以满足。

合一可以被看作是模式匹配的一种推广，模式匹配给定一个常量项和一个模式项。模式项中包含变量。模式匹配的问题是找到一个变量赋值，使得这两个项匹配。

合一与模式匹配类似，不同之处在于两个项都可以包含变量。因此，我们不能再说一个是模式项，另一个是常量项。寻找一个变量替换使它们等价的过程被称为合一。

对于某些可解的合一问题，可能存在无限多个可能的合一结果。合一算法的目标是找到一组最一般的类型替换，使得所有的类型约束方程都得以满足，这被称为最一般合一子或MGU（Most General Unifier）。通过替换，最一般合一子可以转化为任何其他合一子。

### 代码实现
合一算法的基本思想是逐个处理类型约束方程，根据方程中的类型关系和特定规则进行类型替换，直到所有方程都得到满足或无法继续替换为止。如果无法找到满足所有方程的最一般类型替换，则表示类型约束不可满足，推断失败。

```
class Type:             ...
class IntType(Type):    ...
class BoolType(Type):   ...
class FuncType(Type):   ...
class TypeVar(Type):    ... # 类型变量

# typ_x、typ_y 是等式两边的表达式，subst 是能够替换的变量集合
# subst == None 表示合一失败
def unify(typ_x, typ_y, subst):
    if subst is None:
        return None
    elif typ_x == typ_y:
        return subst
    elif isinstance(typ_x, TypeVar):
        return unify_variable(typ_x, typ_y, subst)
    elif isinstance(typ_y, TypeVar):
        return unify_variable(typ_y, typ_x, subst)
    elif isinstance(typ_x, FuncType) and isinstance(typ_y, FuncType):
        if len(typ_x.argtypes) != len(typ_y.argtypes):
            return None
        else:
            subst = unify(typ_x.rettype, typ_y.rettype, subst)
            for i in range(len(typ_x.argtypes)):
                subst = unify(typ_x.argtypes[i], typ_y.argtypes[i], subst)
            return subst
    else:
        return None
        
def unify_variable(v, typ, subst):
    assert isinstance(v, TypeVar)
    if v.name in subst:
        return unify(subst[v.name], typ, subst)
    elif isinstance(typ, TypeVar) and typ.name in subst:
        return unify(v, subst[typ.name], subst)
    elif occurs_check(v, typ, subst):
        return None
    else:
        return {**subst, v.name: typ}
        
# 检查 v 是否出现在 term 中，防止像 X=f(X) 的死循环
def occurs_check(v, typ, subst):
    assert isinstance(v, TypeVar)
    if v == typ:
        return True
    elif isinstance(typ, TypeVar) and typ.name in subst:
        return occurs_check(v, subst[typ.name], subst)
    elif isinstance(typ, FuncType):
        return (occurs_check(v, typ.rettype, subst) or
                any(occurs_check(v, arg, subst) for arg in typ.argtypes))
    else:
        return False
```
## Type inference
单向类型推断和双向类型推断是两种不同的类型推断方法，用于确定表达式的类型。

单向类型推断（Uni-directional type inference）是一种从表达式到类型的推断过程，它根据表达式的结构和上下文信息，逐步推导出表达式的类型。在单向类型推断中，类型信息从上游的表达式传递到下游的子表达式，直到推断出整个表达式的类型。这种推断方法常用于静态类型语言中，例如Java和C++。

双向类型推断（Bi-directional type inference (Hindley-Milner)）是一种结合了类型推导和类型检查的推断方法。它将类型推导（从表达式到类型的推断）和类型检查（从类型到表达式的检查）结合在一起，通过双向的信息流来推断表达式的类型。在双向类型推断中，表达式的类型信息既从上游的表达式传递到下游的子表达式，也从下游的子表达式传递到上游的表达式。这种推断方法常用于函数式编程语言中，例如Haskell和ML。

### 算法实现
Hindley-Milner 算法实现步骤：

1. 为所有子表达式分配符号类型名称（如t1、t2等）。
2. 使用语言的类型规则，根据这些类型名称编写类型方程（或约束）的列表。
3. 使用合一算法解决类型方程的列表。

例子：
```
foo f g x = if f(x == 1) then g(x) else 20
```

第一步结果如下。注意，表达式需要去重处理，常量节点具有已知类型。

|表达式|类型符号|
|-|-|
|foo|t0|
|f|t1|
|g|t2|
|x|t3|
|if f(x == 1) then g(x) else 20|t4|
|f(x == 1)|t5|
|x == 1|t6|
|x|t3|
|g(x)|t7|
|20|Int|

第二步的类型方程列表如下

|Lhs | Rhs | 类型方程节点|
|- | - | - |
|Int|Int| 1|
|t3|Int| (x == 1)|
|Int|Int| (x == 1)|
|t6|Bool| (x == 1)|
|t1|(t6 -> t5)| App(f, [(x == 1)])|
|t2|(t3 -> t7)| App(g, [x])|
|Int|Int| 20|
|t5|Bool| If(App(f, [(x == 1)]), App(g, [x]), 20)|
|t4|t7| If(App(f, [(x == 1)]), App(g, [x]), 20)|
|t4|Int| If(App(f, [(x == 1)]), App(g, [x]), 20)|
|t0|((t1, t2, t3) -> t4) | Lambda([f, g, x], If(App(f, [(x == 1)]), App(g, [x]), 20))|

第三步使用合一算法计算得到类型变量的实际类型。例子中，算法找到foo的类型为：
```
((Bool -> Bool), (Int -> Int), Int) -> Int
```
另一个例子：
```
foo f g x = if f(x) then g(x) else 20
```
类型推断过程将为foo计算以下类型：
```
((a -> Bool), (a -> Int), a) -> Int
```

### 代码实现
```
# 算法第一步，给 ast 节点的 _type 字段赋值
def assign_typenames(node, symtab={}):
    if isinstance(node, ast.Identifier):
        if node.name in symtab:
            node._type = symtab[node.name]
        else:
            raise TypingError('unbound name "{}"'.format(node.name))
    elif isinstance(node, ast.LambdaExpr):
        node._type = TypeVar(_get_fresh_typename())
        local_symtab = dict()
        for argname in node.argnames:
            typename = _get_fresh_typename()
            local_symtab[argname] = TypeVar(typename)
        node._arg_types = local_symtab
        assign_typenames(node.expr, {**symtab, **local_symtab})
    elif isinstance(node, ast.OpExpr):
        node._type = TypeVar(_get_fresh_typename())
        node.visit_children(lambda c: assign_typenames(c, symtab))
    elif isinstance(node, ast.IfExpr):
        node._type = TypeVar(_get_fresh_typename())
        node.visit_children(lambda c: assign_typenames(c, symtab))
    elif isinstance(node, ast.AppExpr):
        node._type = TypeVar(_get_fresh_typename())
        node.visit_children(lambda c: assign_typenames(c, symtab))
    elif isinstance(node, ast.IntConstant):
        node._type = IntType()
    elif isinstance(node, ast.BoolConstant):
        node._type = BoolType()
    else:
        raise TypingError('unknown node {}', type(node))
        
# 算法第二步。类型方程列表结果在type_equations
def generate_equations(node, type_equations):
    if isinstance(node, ast.IntConstant):
        type_equations.append(TypeEquation(node._type, IntType(), node))
    elif isinstance(node, ast.BoolConstant):
        type_equations.append(TypeEquation(node._type, BoolType(), node))
    elif isinstance(node, ast.Identifier):
        pass
    elif isinstance(node, ast.OpExpr):
        node.visit_children(lambda c: generate_equations(c, type_equations))
        type_equations.append(TypeEquation(node.left._type, IntType(), node))
        type_equations.append(TypeEquation(node.right._type, IntType(), node))
        if node.op in {'!=', '==', '>=', '<=', '>', '<'}:
            type_equations.append(TypeEquation(node._type, BoolType(), node))
        else:
            type_equations.append(TypeEquation(node._type, IntType(), node))
    elif isinstance(node, ast.AppExpr):
        node.visit_children(lambda c: generate_equations(c, type_equations))
        argtypes = [arg._type for arg in node.args]
        type_equations.append(TypeEquation(node.func._type,
                                           FuncType(argtypes, node._type),
                                           node))
    elif isinstance(node, ast.IfExpr):
        node.visit_children(lambda c: generate_equations(c, type_equations))
        type_equations.append(TypeEquation(node.ifexpr._type, BoolType(), node))
        type_equations.append(TypeEquation(node._type, node.thenexpr._type, node))
        type_equations.append(TypeEquation(node._type, node.elseexpr._type, node))
    elif isinstance(node, ast.LambdaExpr):
        node.visit_children(lambda c: generate_equations(c, type_equations))
        argtypes = [node._arg_types[name] for name in node.argnames]
        type_equations.append(
            TypeEquation(node._type,
                         FuncType(argtypes, node.expr._type), node))
    else:
        raise TypingError('unknown node {}', type(node))

# 类型变量替换。合一的结果在 subst
def apply_unifier(typ, subst):
    if subst is None:
        return None
    elif len(subst) == 0:
        return typ
    elif isinstance(typ, (BoolType, IntType)):
        return typ
    elif isinstance(typ, TypeVar):
        if typ.name in subst:
            return apply_unifier(subst[typ.name], subst)
        else:
            return typ
    elif isinstance(typ, FuncType):
        newargtypes = [apply_unifier(arg, subst) for arg in typ.argtypes]
        return FuncType(newargtypes,
                        apply_unifier(typ.rettype, subst))
    else:
        return None
```
