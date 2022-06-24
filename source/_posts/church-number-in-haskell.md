---
title: 在Hasekll中构造丘奇数及其运算
date: 2022-06-24 11:59:14
categories: Type Systems
tags: ["λ演算", "Haskell"]
---
## 0.布尔运算
### 0.0 true, false和if
在λ演算中，两个布尔值可以这样定义：
```haskell
tru = \t -> \f -> t
fls = \t -> \f -> f
```

为什么不直接构造一个`true`一个`false`解决问题呢？因为在无类型λ演算中不存在所谓的`if/else`或者`match/case`。
因此对于传入的参数我们无法判断究竟是哪一个。这样的定义可以允许我们很直接地构造出一个`if/else`函数出来：

```haskell
test = \l -> \m -> \n -> l m n
```

如果函数值为`tru`，那么返回值就为`m`，否则为`n`。这里的`test`与`if/else`是等价的：

```haskell
*Main> test tru 2 3
2
*Main> test fls 4 5
5
```

你也可以交换`\t`和`\f`的位置，只要确保语义正确即可。将`\t`放在前面更容易契合`if/else`的直观感受。

### 0.1 and, or, not
有了布尔值就可以进行相应的布尔运算。首先是`and`：

```haskell
b_and = \x -> \y -> x y fls
```

不需要真值表，我们可以很容易地理解上面这行代码：
- 如果左式为真，那么运算结果为右式
- 如果左式为假，结果为假

同理可以定义`or`：

```haskell
b_or = \x -> \y -> x tru y
```

`not`用于翻转布尔值，需要用到之前定义的`test`：
```haskell
b_not = \x -> test x fls tru
```

我们可以简单地验算一下这几个式子：
```haskell
*Main> test (b_and tru fls) 1 0
0
*Main> test (b_and tru tru) 1 0
1
*Main> test (b_or fls fls) 1 0
0
*Main> test (b_not fls) 1 0
1
```

### 0.2 pairs
布尔值还可以用来构造`pair`。比起普通的，我们常见的`pair`（比如`std::pair`），我们需要一个额外的参数：
```haskell
pair = \f -> \s -> \b -> b f s
```

第三个参数用于决定我们需要取出哪个参数。由于函数柯里化，我们可以不急着给出第三个参数：
```haskell
x = pair 1 2
```

当我们需要取出某个值时，就可以这样操作：
```haskell
first = \p -> p tru
second = \p -> p fls
```

现在我们可以得到第一个或者第二个分量：
```haskell
*Main> first x
1
*Main> second x
2
```

## 1.丘奇数
### 1.0 successor
众所周知，丘奇数的定义是后继函数的调用次数。我们可以这样定义数字0、1，以及后继函数：
```haskell
c0 = \s -> \z -> z
c1 = \s -> \z -> s z
succ = \n -> \s -> \z -> s(n s z)
```

也可以直接使用`c1`来构造后继，本质上就是让`s`再多执行一次：
```haskell
suc = \n -> \s -> \z -> n s (c1 s z)
```

### 1.1 加、乘和幂
所谓的加法`x+y`，就是在`y`的基础上，再执行`x`次后继运算。我们已经在`suc`函数的第二个定义中看到这个用法了：修改初始变量`z`（抽象意义上的0）为`c1`，就能实现加一的操作，那么加任意丘奇数的操作也是类似的：

```haskell
plus = \x -> \y -> \s -> \z -> x s (y s z)
```

同理，乘法`x × y`就是对`y`执行`x`次加法。如果我们可以替换掉`z`，同理我们也能够替换`s`。原先的后继每次只能加一，我们使用`plus n`作为新的后继函数：
```haskell
times = \x -> \y -> x (plus y) c0
```

注意这里不再需要提供`s`和`z`：`s`已经由我们所指定`plus y`，`z`也已经被`c0`所替换。前序的所有函数中，我们都没能替换`s`，而`s`的参数类型恰好是需要和`z`一致的。因此只要`s`还在，我们就无法省略`z`，即便初始变量已经被替换。
你也可以通过`[plus y/\t -> \s -> \z -> -> y s (t s z)]times`来删掉对`plus`函数的调用：
```haskell
times = \x -> \y -> x (\t -> \s -> \z -> y s (t s z)) c0
```

幂运算需要我们将`s`替换为乘法：
```haskell
power = \x -> \y -> y (times x) c1 -- x^y
```

### 1.2 前继和减法
众所周知，一个运算的逆运算往往比其本身要复杂得多。小学时你会认为除法比乘法更难算，就算到了大学你也认为微分比积分更简单。
这是无可厚非的事。因此我们也在定义完所有“正向”的运算后来考虑他们的逆运算。

对于前继运算来说，我们无法凭空让某次调用消失。因此需要借助`pair`来缓存一下上次运算的结果：
```haskell
zz = pair c0 c0
ss = \p -> pair (second p) (suc (second p))
pre = \m -> first (m ss zz)
```

由于丘奇数是自然数的表示形式，因此0的前继依然是0。

我们尝试着依葫芦画瓢来得到减法。虽然我们的加法里没有找到对`suc`的调用，但是我们可以稍加改写：
```haskell
plus = \x -> \y -> y (suc) x
```

我们将`x`视作`z`，并在上面施加`y`次`suc`操作，这等同于之前的写法。因此减法也可以被写作：
```haskell
sub = \x -> \y -> y (pre) x
```

这段代码在Haskell中只有当减数小于等于1时才能通过编译，原因是`c2`及以后的数字类型推导为`(t -> t) -> t -> t`，而`c1: (t1 -> t2) -> t1 -> t2`，在更多次调用`s`后，所有的`t`都被约束为同一个类型的变量，这会导致编译器产生无穷类型。如果你试着像这样给`c1`指定类型，那么`c1`也将无法通过编译：
```haskell
c1 :: (t -> t) -> t -> t
c1 = \s -> \z -> s z
```

不过我们可以人工验证算法的正确性，这里以`4 - 2`为例：
```
sub c4 c2
= c2 (pre) c4
= (\s -> \z -> s(s z)) (pre) c4
= pre(pre c4)
= pre(c3)
= c2
```

除法和根号非常复杂，在这里我们暂时不讨论。

## 2.其他函数
### 2.0 iszero
`iszero`判断一个数字是否为0，是的话返回`tru`，不是的话返回`fls`。一个简单的利用丘奇数性质的方法是定义一个常数函数，其永远返回`fls`，如果这个函数被调用，就说明该数字大于0：
```haskell
iszero = \n -> n (\t -> fls) tru
```

### 2.1 equal
另一个非常有用的函数是判断两个丘奇数是否相等。由于有了`iszero`，我们可以通过判断差是否为0来确定是否相等。
注意：丘奇数没有复数，如果`x`小于`y`，`sub x y`的结果也将是0。因此，我们需要判断两次：
```haskell
equal = \x -> \y -> and (iszero(sub x y) iszero(sub y x))
```

同理，因为类型推断的问题函数无法通过编译，我们可以简单地判断一下`3 == 5`：
```
equal c3 c5
= and (iszero(sub 3 5) iszero(sub 5 3))
= and (iszero(0) iszero(2))
= and tru fls
= fls
```

## 参考文献
- Pierce, B., n.d. Types and Programming Languages. London, England: The MIT Press, pp.58-65.