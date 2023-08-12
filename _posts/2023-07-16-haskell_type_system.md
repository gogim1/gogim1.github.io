---
title: "Haskell 类型系统进阶"
tags: [notes, haskell, wip]
---

总结自「 Haskell in Depth 」第十一章 Type system advances

<!--more-->

本文涉及：

* types system
* kinds system
* data kinds
* type families
* generalized algebraic data types（GADT）
* polymorphism

## Terms, types and kinds
Term 就是具体的值，不同的 Term 可按照 Type 划分，不同的 Type 又可以按照 Kind 划分:

```
ghci> :type True
True :: Bool
ghci> :kind Bool
Bool :: *
ghci> :kind Maybe
Maybe :: * -> *
ghci> :kind Maybe Bool
Maybe Bool :: *
```
通过 `*` 和 `->` 可以来区分具体的 Type 和 Type constructors（Type 是 `*`，Type constructors 是 `* -> * ->...`），但是区分不了 type classes。因此可以开启 `NoStarIsType` 扩展：
```
ghci> :kind Int
Int :: Type
ghci> :kind Maybe
Maybe :: Type -> Type
ghci> :kind Monad
Monad :: (Type -> Type) -> Constraint
ghci> :kind Monad Maybe
Monad Maybe :: Constraint
```
kind checker 会进行检查，这也是 Maybe 可以是 Monad，而 Int 不能是 Monad 的原因：
```
ghci> :kind Monad Int
<interactive>:1:7: error:
    ? Expected kind ‘Type -> Type’, but ‘Int’ has kind ‘Type’
    ? In the first argument of ‘Monad’, namely ‘Int’
      In the type ‘Monad Int’
```
我们不能以如下的方式对 Kind 进行测试，因为 `a` 和 `m` 没有被定义：
```
ghci> :kind Maybe a 
<interactive>:1:7: error: Not in scope: type variable ‘a’
ghci> :kind Monad m
<interactive>:1:7: error: Not in scope: type variable ‘m’
```
可以通过开启 `ExplicitForAll` 扩展显性地定义 `a` 和 `m`：
```
ghci> :set -XExplicitForAll
ghci> :kind forall a. Maybe a
forall a. Maybe a :: Type
ghci> :kind forall m. Monad m
forall m. Monad m :: Constraint
```
目前而言，Kind 也是一类 Type：
```
ghci> import Data.Kind
ghci> :kind Type
Type :: Type
ghci> :kind Type -> Type
Type -> Type :: Type
ghci> :kind Constraint
Constraint :: Type
```
不过目前 Haskell 社区在尝试统一 Term 和 Type。当这种尝试成功后，所有的 Type 都是一种 Term，也就不再需要 Kind。

## Delivering information with types
```
newtype Temp unit = Temp Double
    deriving (Num, Fractional)
    
data F
data C

paperBurning :: Temp F
paperBurning = 451

absoluteZero :: Temp C
absoluteZero = -273.15
```
这里的 unit 没有被 constructors 使用到，因此是 `phantom parameter`。可以对计算进行检查，防止类型不匹配：
```
gchi> paperBurning - absoluteZero
    ? Couldn't match type ‘C’ with ‘F’
      Expected: Temp F
        Actual: Temp C
    ? In the second argument of ‘(-)’, namely ‘absoluteZero’
      In the expression: paperBurning - absoluteZero
      In an equation for ‘fun’: fun = paperBurning - absoluteZero
```
值得注意的是，这里对 unit 的具体 Kind 没有太多限制，只要是 Type 即可（不能是 Type -> Type 等等）

假设要为针对 Temp 中的 unit 实现 type class：
```
class UnitName u where
    unitName :: u -> String

instance UnitName C where
    unitName _ = "C"

instance UnitName F where
    unitName _ = "F"

instance UnitName unit => Show (Temp unit) where 
    show _ = unitName (? :: unit)
```
这里的 `？` 要求传入 term，而 C 和 F 只是 tag，并没有 constructor。base 包中的 Data.Proxy 定义这样的类型：
```
data Proxy t = Proxy
```
Proxy 用在需要类型而不需要值的情况：
```
class UnitName u where
    unitName :: Proxy u -> String

instance UnitName unit => Show (Temp unit) where 
    show _ = unitName (Proxy :: Proxy unit)
    
show' :: forall u. UnitName u => Temp u -> String
show' _ = unitName (Proxy :: Proxy u)
```

在 UnitName typeclass 的定义中，u 的 Kind 默认是 Type，所以不能为 kind 是 Type -> Type 或其他实现 instance。typeclass 中的 kind 默认不是多态，可以开启拓展 `PolyKinds`：
```
instance UnitName Temp where
    unitName _ = "__unspecified unit__"
```

前面的代码通过 Proxy 来传递类型信息。也可以通过开启 `TypeApplications` 拓展，使用 @ 在调用函数时指定 type variable 的具体类型
```
ghci> read @Int "42"
```
@ 指定类型的顺序与函数定义时 type variable 的出现顺序一致：
```
elem :: (Foldable t, Eq a) => a -> t a -> Bool  -- the first @ specifies the t, and the second the a
const :: a -> b -> a                            -- the first @ specifies the a, and the second the b
myConst :: forall b a. a -> b -> a              -- the first @ specifies the b, and the second the a
```
因此前面的代码可以修改成如下：
```
class UnitName u where
    unitName :: String

instance UnitName C where
    unitName = "C"

instance UnitName F where
    unitName = "F"

instance UnitName u => Show (Temp u) where 
    show _ = unitName @u
    
show' :: forall u. UnitName u => Temp u -> String
show' _ = unitName @u
```
其中 unitName 的类型是 `unitName :: forall {k} (u :: k). UnitName u => String`。这里需要开启 `AllowAmbiguousTypes` 拓展，因为编译器没办法推断 u 的类型

## Type operators
`TypeOperators` 拓展允许含有运算符的类型名字：
```
data a + b = Inl a | Inr b
```

## Promoting types to kinds and values to types
当 `DataKinds` 拓展打开时，
```
data TempUnits = F | C

newtype Temp (u :: TempUnits) = Temp Double 
    deriving Show
```
其中：

1. TempUnit type 有 term F 和 C
2. TempUnit kind 有 type F 和 C

即值 F 和 C 被提升到 Type，类型 TempUnit 被提升到 Kind。

在有歧义的地方，可以为 C 和 F 加上 `'` 前缀，表示其是类型而不是值：
```
func :: Temp 'F
func = undefined
```
## Type-level literals
开启 `DataKinds` 拓展后，base 库内置的类型也会被提升到 Kind，GHC.TypeLists 模块定义了这些类型。
```
ghci> :kind 42
42 :: Natural
ghci> :kind "hello"
"hello" :: Symbol
ghci> :kind []
[] :: Type -> Type
ghci> :kind [Int, String, Bool]
[Int, String, Bool] :: [Type]
```
下面是将指针的步进长度记录到类型中的一个示例：
```
newtype Pointer (align :: Nat) = Pointer Integer

zeroPtr :: Pointer n
zeroPtr = Pointer 0

inc :: Pointer align -> Pointer align
inc (Pointer p) = Pointer (p + 1)

ptrValue :: forall align. KnownNat align => Pointer align -> Integer
ptrValue (Pointer p) = p * natVal (Proxy :: Proxy align)
 
ghci> ptrValue (inc $ zeroPtr :: Pointer 4)
4
ghci> ptrValue (inc $ zeroPtr :: Pointer 8)
8
```
其中 `KnowNat` typeclass 定义了 natVal 方法，该方法以 Proxy 为参数，将数字类型转为其值

下面是将字符串记录到类型中的一个示例：
```
newtype SuffixedString (suffix :: Symbol) = SuffixedString String

asString :: forall suffix. KnownSymbol suffix => SuffixedString suffix -> String
asString (SuffixedString str) = str ++ symbolVal (Proxy :: Proxy suffix) 

ghci> asString (SuffixedString "hello" :: SuffixedString "world") 
"helloworld"
```

## Computations over types with type families
type families 用于把类型映射到另一种类型，Haskell 提供了多种 type families，本节讨论 type synonym families、data families 和 associated families。

type families 有两种定义方式，open 和 close：
```
-- open 方式可以在其他位置增加新的 instance，在其他 module 也能增加。
-- open 方式不能增加一个为任意类型做映射的 instance
type family Simplify t
type instance Simplify Integer = Integer
type instance Simplify Int = Integer

-- close 方式只能在 where 增加 instance
-- close 方式可以增加一个为任意类型做映射的 instance
type family Simplify t where
    Simplify Integer = Integer
    Simplify Int = Integer
    Simplify a = String
```


