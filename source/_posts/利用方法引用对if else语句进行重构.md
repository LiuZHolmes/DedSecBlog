---
title: 利用方法引用对if else语句进行重构
date: 2019-10-08 10:14:24
tags: Refactor
---

## 前言
在业务代码中，有许多复杂的业务逻辑需要用代码实现。一般来说，最常见的就是一些分支逻辑。那么对应的，在代码中出现最多的自然就是一些if else的判断。如果只是少量的判断语句，是不会影响太多代码的可读性的。但如果**有太多类似的判断语句在做着大致相同的操作**，那么其实是可以利用Java语言中[方法引用](https://www.runoob.com/java/java8-method-references.html)的特性进行重构的。
## 简单的例子
首先来看一个比较简单的情况，

```java
void calHandLevelAndAce(){
    if(tryThreeOfAKind()!=null) setPower(tryThreeOfAKind());
    else if(tryTwoPairs()!=null) setPower(tryTwoPairs());
    else if(tryPair()!=null) setPower(tryPair());
    ...
}
```
这个例子中的代码是一个高度抽象的方法，每个if表达式里面只有一个变量，分支里也只有一两个调用语句。这样可以更方便理清重构的思路。
可以看到，每个分支里做的事情其实是大致相同的。当在重构时见到这种情况时，就应该比较意识到这里是一个可以下手的地方。再上一层，甚至if表达式里都是这种类似的情况。那么，就可以发现，其实几个if语句的骨架是一致的，只是一些具体的调用不同。

如果不同的不是调用方法，而只是，比如说，一个整数变量的值，那么这个重构就非常直观简单了：这种一系列整数值的操作，哪怕是一个新手也会用一个数组去存储这几个值，然后用for循环处理。但如果就只是调用不同呢？其实也可以采用和数值类似的思想，用一个数组保存这些方法，再同样用一个for循环处理，
```java
[TYPE] methods[] = [tryThreeOfAKind,tryTwoPairs, tryPair];
void calHandLevelAndAce(){
    for(int i = 0;getPower() == null && i < methods.length;i++) { // power is null at the beginning
        setPower(methods[i]);
    }
    ...
}
```
需要注意的是，power一旦为null就跳出if else语句的逻辑是放在了for循环的循环条件中。

这段代码很好地体现了我们重构的思路，但却不能通过编译。幸好这种写法在Java是被支持的，只是因为我们并没有按照方法引用需要的语法写好而已。

要把它改成可以通过编译的形式，首先要做的就是把方法存到变量里——[函数接口](https://www.runoob.com/java/java8-functional-interfaces.html)提供了可能。我们的try方法都是不接受参数，返回同一类型变量的方法——对应的接口就是Supplier。

因此，代码可以改为，
```java
List<Supplier<Power>> tryDifferentPower = Arrays.asList(this::tryStraight,this::tryThreeOfAKind,this::tryTwoPairs,this::tryPair,this::tryHighCard);
void calHandLevelAndAce(){
    for(int i = 0;getPower() == null && i < methods.length;i++) { // power is null at the beginning
        setPower(tryDifferentPower.get(i).get());
    }
    ...
```
更进一步地，引入lambda和stream，可以写成更简洁的形式。
```java
void calHandLevelAndAce(){
    final List<Supplier<Power>> tryDifferentPower = Arrays.asList(this::tryStraight,this::tryThreeOfAKind,this::tryTwoPairs,this::tryPair,this::tryHighCard);
    setPower(tryDifferentPower.stream().map(Supplier::get).filter(Objects::nonNull).findFirst().orElse(null));
}
```
如果有大概了解stream的用法就可能会对这种写法有些疑问，因为它看上去是把所有的方法都试了一遍，再取其中不为null的结果。实际上，如果打上断点进入debug模式可以看到，stream的实现是一个流水线的方式，当前方法才走到map的时候，上一个方法可能已经走到了findFirst。如果这样，stream内部就应该有对应的优化，当流里面有走到findFirst的数据时，可能就会中断流水线后面的操作了，因为那些都是没有必要的。（实际上的表现也是这样的，但没有具体去确定是不是真的这样实现了）

一个更复杂的情况在[这里](https://github.com/LiuZHolmes/PokerHand/commit/3bf412314abca719e9da5c148ee67474b864f66f)，如果有兴趣或者自信对这种方式理解到位了可以尝试以下。
## 理论分析
从理论上来说，这种重构的方式可以应对绝大多数if else语句的情况。因为这个方式对原始代码主要的要求只有两个，第一是原始代码if语句内部的操作可以都抽象为一致的函数接口，第二就是if表达式的判断都是逻辑一致的。

第一个条件基本上是肯定满足的。要求接口一致，其实就是要求方法的输入输出的参数列表一致。输入是肯定能保证一致的。因为原始代码中方法的输入是固定的，这就说明整个方法内部需要的数据集是确定的。因此，每个if语句内部用到的数据也一定是这个数据集的子集。所以，肯定有一个集合（比如说，就是最大的数据集）是能够包含每个if语句所需要的数据的。这个集合就可以当作一致的输入。

输出的话，因为判断的逻辑写在代码里，如何体现都是可以自定义的。如果没有太特殊的情况，都可以固定为统一的值，比如说都是布尔值。

同理，第二个条件也是很大程度可以自己控制的，比如说可以规定为判断是否为真。

综上所述，原理上说，这个方法可以重构绝大多数的if else冗杂的情况。

## 碎碎念
其实这种实践某种程度来说可以算是一种“奇技淫巧”。我之所以想到这种方法是因为在阅读这些代码时总感觉有太多重复类似的地方，应该用一种更优雅的方式去实现。因此，针对每个if语句内的操作，我第一个想到的是采用“函数指针”的方式去提取出其中的方法出来。但函数指针是C++中的一种技巧，我也是查阅资料才知道在Java中类似的叫方法引用。在Java1.8的新特性中很多都是互相呼应的，比如这个重构其实综合了lambda，方法引用和函数接口。

最后还想唠叨的是，虽然我没有去了解方法引用的实现是不是接近函数指针，但我相信它们应该都是类似的。函数指针其实就是保存了那段函数代码在内存中的地址，而我们之所以能够这样做，是得益于目前的计算机都是冯诺依曼结构，即代码视作与数据一致，都是同样的存储方式。