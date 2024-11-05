# Haskell 拓展

标签（空格分隔）： 未分类

---

### Reader 和 State 的区别
https://www.reddit.com/r/haskell/comments/7gl2wh/when_to_use_reader_vs_state/


---

### RecordWildCards
> https://riptutorial.com/haskell/example/13072/recordwildcards

### NamedFieldPuns
todo

### OverloadedStrings 
Haskell’s numeric literals are polymorphic over Num (in the case of integer literals) or Fractional (in the case of decimal literals)：
```
a :: Int
a = 1

b :: Double
b = 1

c :: Float
c = 3.5

d :: Rational
d = 3.5
```
String literals are always of type String, and are not polymorphic at all.

The OverloadedStrings extension simply changes the type of literals to `"test" :: Data.String.IsString a => a`, making string literals polymorphic over the IsString type class.

```
a :: String
a = "hello"

b :: Text
b = "hello"
```

OverloadedStrings also adds IsString to the list of defaultable type classes, so you can use types like String, Text, and Bytestring in a default declaration.

### ExtendedDefaultRules 
> https://stackoverflow.com/questions/26778415/using-overloaded-strings

### FlexibleInstances
通常情况下，我们不能给多态类型（polymorphic type）的特化版本（specialized version）写类型类实例

While you can create an instance for [a] you can't create an instance for specifically [Int].; FlexibleInstances relaxes that:
```
class C a where

-- works out of the box
instance C [a] where

-- requires FlexibleInstances
instance C [Int] where
```

### BinaryLiterals 
Haskell 字面值可以使用十进制、十六进制、八进制。开启 BinaryLiterals 拓展后能够使用二进制字面值

### PostfixOperators
感觉没啥用

### TupleSections
```
-- With TupleSections
(a,b) == (,) a b == (a,) b == (,b) a
(,2,) 1 3 == (1,2,3)
map (,"tag") [1,2,3] == [(1,"tag"), (2, "tag"), (3, "tag")]
```
### EmptyCase
允许空 case 语句。不是很实用

### MultiWayIf
```
if | x == 1    -> "a"
   | y <  2    -> "b"
   | otherwise -> "d"
```
### DoAndIfThenElse
Haskell2010 默认开启

### RankNTypes
> https://riptutorial.com/haskell/example/9606/rankntypes

