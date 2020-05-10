---
layout: article
title: 一步步推导Y combinator
tags: lisp
aside:
  toc: true

---


## 前言
最近在读`The little schemer`，读到`Y combinator`部分时，感觉复杂但十分有趣。`Y combinator`是用来解决匿名函数递归调用的利器，其scheme版本的实现只有几行代码，但确实优雅又强大。鉴于推导过程比较复杂，写了这篇长文来记录下Y组合子的推导和思考过程。

先贴一下最终代码有个初步的轮廓认识，然后对这部分代码进行推导和论证：
```scheme
(define Y
  (lambda (le)
    ((lambda (f) (f f))
     (lambda (f)
       (le (lambda (x) ((f f) x)))))))
```

接下来讲的内容假定你已经熟悉scheme的语法，递归调用和函数式编程的基础知识，如果这些内容尚不熟悉，建议先看一下`The little schemer`的前几章或`SICP`的前几章内容。

## 问题衍生的问题衍生的问题...
在我们编写代码时会时常用到递归函数，有的递归函数经过有限步的计算后一定可以得到想要的结果(total function)，例如下面这个计算列表长度的函数：
```scheme
(define length
  (lambda (l)
    (cond
     ((null? l) 0)
     (else
      (add1 (length (cdr l)))))))
```
而有的函数不尽如人意，并不能通过有限步骤得到正确的结果，且陷入死循环，例如：
```scheme
(define eternity
  (lambda (l)
    (eternity l)))
```
如果我们可以写一个函数，能够来判断我们的递归函数是否最终是收敛可终止的，emm~这一定非常nice...

实际上，科学家们也通过反证法进行了研究推导，结论是这种函数并不存在。这里我不展开讲原因，因为这并不是重点，只需要理解一点：验证函数在传入一个参数时是否可以收敛到停止，运行这个函数是前提，如果这个函数不能收敛到停止，这个函数的外层调用就永远停不下来，因此就得不到结果。简单的代码如下，感兴趣的可以自己体会一下：
```scheme
(define last-try
  (lambda (x )
    (and (will-stop? last-try)
         (eternity x))))
```

不过由此产生了一个有意思的问题，从上面我们能看到函数递归调用的前提是函数有一个名字，如果一个函数是匿名的，那它能不能调用自己呢，如果可以的话，该如何调用呢？如果不能的话，有没有方法可以解决呢...

没错，解决这个问题的方案正是`Y combinator`。

## 故事开始于边界，结束于任意...
我们观察一下上面提到的`length`函数，通过`(define...)`的形式先给自己取了一个函数名字，然后在函数实现的部分又用到了`length`这个名字进行调用，通过命名的方式实现了递归。但是，这也只是有一个函数名字在头部而已，真的非用不可吗？我们来做第一次尝试，不用`(define...)`命名的方式，改用匿名函数实现，记为`length*`：
```scheme
 (lambda (length)
    (lambda (l)
      (cond
       ((null? l) 0)
       (else
        (add1 (length (cdr l)))))))
```
emm~函数看起来不错，不过问题也随之而来了，逻辑虽然保持一致，但这个函数该如何使用呢，是不是真的可以得到跟之前`length`函数一样的结果呢？别着急，带着这些问题，我们来推导一下，可以举有限个数的几个具体例子，观察规律，一步步化简，向着目标进发...

接下来的内容需要你集中精神，因为如果你读一会走神了后面可能就看不懂了...

### 初探

首先来考虑边界为空列表的情况，然后在此基础上逐步外扩，向着可以任意长度列表作为参数都能递归的方向进发。为了方便区分逻辑层次，不至于看的眼花缭乱，我们给这些一些函数添加标识，但并不会递归使用，因此我们研究的问题也就没有发生变化。

以下只能接受空列表的函数，我们记为`length0`，只能接受长度为0的列表并返回正确结果：
```scheme
(lambda (l)
    (cond
     ((null? l) 0)
     (else
      (add1 (eternity (cdr l))))))
```
接下来我们可以顺利得到只接受长度为1和0的列表并返回正确结果的函数`length1`：
```scheme
 (lambda (l)
    (cond
     ((null? l) 0)
     (else
      (add1 (length0 (cdr l))))))
```
我们还可以继续得到接受长度为2及以下列表并返回正确结果的函数`length2`
```scheme
 (lambda (l)
    (cond
     ((null? l) 0)
     (else
      (add1 (length1 (cdr l))))))
```
当然，你可以继续写下去，但已经没有必要，也可以把`length0`和`length1`在`length2`中展开，你会得到重复度极高的一大坨，代码如下：
```scheme
(lambda (l)
  (cond
   ((null? l) 0)
   (else
    (add1
      ((lambda (l)
         (cond
          ((null? l) 0)
          (else
           (add1 ((lambda (l)
                    (cond
                     ((null? l) 0)
                     (else
                      (add1
                        (eternity
                          (cdr l)))))) (cdr l)))))) (cdr l))))))

```

现在，可以看到，越往外扩，代码层次越深，且重复度高。我们需要做的事情就是观察&化简&抽象。现在来尝试做一些化简变形，试着简化调用的代码，取出底层调用的函数作为单独的参数传入lambda来生成新的lambda。依然从空列表开始，记为`length0*`，代码如下：
```scheme
((lambda (length)
   (lambda (l)
     (cond
      ((null? l) 0)
      (else
       (add1 (length (cdr l)))))))
 eternity)
```
我们再来尝试小于等于1参数的`length1*`，代码如下：
```scheme
((lambda (length)
   (lambda (l)
     (cond
      ((null? l) 0)
      (else
       (add1 (length (cdr l)))))))
 length0*)
```

通过`length1*`代换得到`length2*`，功能完全同`length2`：
```scheme
((lambda (length)
   (lambda (l)
     (cond
      ((null? l) 0)
      (else
       (add1 (length (cdr l)))))))
 length1*)
```
看到这里你可能感觉好像也并没有化简什么，不过我们再来看展开`length2*`后的代码，是不是变得层次更清晰了？
```scheme
((lambda (length)
   (lambda (l)
     (cond
      ((null? l) 0)
      (else (add1 (length (cdr l)))))))
 ((lambda (length)
    (lambda (l)
      (cond
       ((null? l) 0)
       (else (add1 (length (cdr l)))))))
  ((lambda (length)
     (lambda (l)
       (cond
        ((null? l) 0)
        (else (add1 (length (cdr l)))))))
   eternity) ) )
```

故事进行到一半，或许你可以稍微休息一会...

### 再探

现在，我们继续。观察上面`length2*`的代码，你会发现代码结构有了好转，但重复度依然很高，我们需要再继续抽象&化简。不难发现，重复的是这段代码：
```scheme
 (lambda (length)
    (lambda (l)
      (cond
       ((null? l) 0)
       (else
        (add1 (length (cdr l)))))))
```
我们依然用上面函数作为参数来抽象代码逻辑的方法进行化简，不过这次我们连`length`逻辑本身也作为参数了，那就起个新名字吧，嗯，mk-length(by Friedman)就不错。

还是从可以接受空列表的函数开始，`mk-length0`代码如下，或许你刚看这段代码有点诧异，尝试把函数传入参数试试看？是的，它跟`length0`和`length0*`的功能是一样的。
```scheme
((lambda (mk-length)
   (mk-length eternity))
 (lambda (length)
   (lambda (l)
     (cond
      ((null? l) 0)
      (else
       (add1 (length (cdr l))))))))
```
如果上面代码理解了，那可以轻松得到`mk-length1`的版本了：
```scheme
 ((lambda (mk-length)
    (mk-length
     (mk-length eternity)))
  (lambda (length)
    (lambda (l)
      (cond
       ((null? l) 0)
       (else
        (add1 (length (cdr l))))))))
```
嗯，就连`mk-length3`的版本也不过如此：
```scheme
 ((lambda (mk-length)
    (mk-length
     (mk-length
      (mk-length
       (mk-length eternity)))))
  (lambda (length)
    (lambda (l)
      (cond
       ((null? l) 0)
       (else
        (add1 (length (cdr l))))))))
```
这时候也不用展开了，因为我们没有使用任何的标记就轻松写出了这段代码，lucky！

### 见到曙光...

接下来，我们尝试把尽可能多的变量名统一命名，并且使函数本身含义不发生变化，来帮助我们进一步化简，是的，命名越统一越容易化简。首先用`mk-length`来代替其中的一部分命名重写`mk-length0`函数，记为`mk-length2-0`。
```scheme
((lambda (mk-length)
   (mk-length mk-length))
 (lambda (mk-length)
   (lambda (l)
     (cond
      ((null? l) 0)
      (else
       (add1 (mk-length (cdr l))))))))
```

仔细观察一下，确实跟原先函数的语义是一样的，我们甚至可以再继续写出用`mk-length`来代替其中便一部分命名的`mk-length1`版本，记为`mk-length2-1`：
```scheme
((lambda (mk-length)
   (mk-length mk-length))
 (lambda (mk-length)
   (lambda (l)
     (cond
      ((null? l) 0)
      (else
       (add1 ((mk-length eternity) (cdr l))))))))
```

当然你也可以手动写一下替换版本的`mk-length2`和`mk-length3`...

这时候我们会有一个有趣的发现，mk-length包裹的层数决定了可以接纳列表的长度，eternity最终将超过限度的列表困死。这时我们不妨把`eternity`也替换成`mk-length`，记为`mk-length-n`：
```scheme
((lambda (mk-length)
   (mk-length mk-length))
 (lambda (mk-length)
   (lambda (l)
     (cond
      ((null? l) 0)
      (else
       (add1 ((mk-length mk-length) (cdr l))))))))
```

emm~这时候神奇的事情发生了，这个函数的功能不就是活生生的`length`吗？它确实可以接受任意长度的list并返回长度，而且它确实是一个匿名的函数，通过自身调用自身就实现了递归。

不过，我们发现一个问题，这个函数长得已经不像`length`了，距离一开始我们设想的`length*`也有一些距离，这样就很难直接得到其他匿名函数的递归调用写法，所以，我们还得把这个函数样式变回去，最好还能总结出一套通用的范式。

### 尾声...

这时候我们的目标已经比较明确，尽可能朝着最原始`length`逻辑样式去化简。

接下来需要一点小技巧，通过观察发现`mk-length-n`和`length*`逻辑区别在于前者`((mk-length mk-length) (cdr l))`的部分后者是`(add1 (length (cdr l))`，可能你已经忘了，再返回去看看吧~

这里可以用参数代换的方法来渐进两者的区别，直观想一下逻辑大概是这个样子：
```scheme
((lambda (mk-length)
     (mk-length mk-length))
   (lambda (mk-length)
     ((lambda (length)
         (lambda (l)
           (cond
            ((null? l) 0)
            (else
             (add1 (length (cdr l)))))))
      (mk-length mk-length))))
```

打住！！！千万别执行这段代码，否则你得重启电脑了...

至于为什么，建议读者自行思考一下，用一个长度为1的列表代入走一走容易得到答案的...

问题出在用`length`代换`(mk-length mk-length)`的地方，这个问题我们可以用再包装的方式处理下，用以下函数代替`(mk-length mk-length)`部分：
```scheme
(lambda (x)
  ((mk-length mk-length) x))
```

emm~是不是还一样？不过这时候就不会有死循环无限分配函数递归的问题了，读者也可以自行验证一下，完整代码如下：
```scheme
((lambda (mk-length)
   (mk-length mk-length))
 (lambda (mk-length)
   ((lambda (length)
      (lambda (l)
        (cond
         ((null? l) 0)
         (else
          (add1 (length (cdr l)))))))
    (lambda (x)
      ((mk-length mk-length) x)))))
```

这时候我们能比较清楚看到`length`的非递归版本`length*`是包含在里面的，也就是这部分代码：
```scheme
(lambda (length)
  (lambda (l)
    (cond
     ((null? l) 0)
     (else
      (add1 (length (cdr l)))))))
```

接下来我们还是用上面的函数参数代换的方法，把上述`length*`作为函数参数代入算式，就可以得到一个通用的算子和一个匿名函数协作完成递归的故事啦！
```scheme
((lambda (le)
   ((lambda (mk-length)
      (mk-length mk-length))
    (lambda (mk-length)
      (le (lambda (x)
            ((mk-length mk-length) x))))))
 (lambda (length)
   (lambda (l)
     (cond
      ((null? l) 0)
      (else
       (add1 (length (cdr l))))))))
```

是的，激动人心的东西得到虽然不易，但就是这么开心！这个通用的算子也就是这篇文章的主角--Y combinator(Y组合子)：
```scheme
(define Y
  (lambda (le)
    ((lambda (f) (f f))
     (lambda (f)
       (le (lambda (x) ((f f) x)))))))
```

写完这篇文章，最大的感触不是`Y combinator`最终的形态如何简单，而是推导的过程带给人的思考。所谓知其然易，知其所以然难...
