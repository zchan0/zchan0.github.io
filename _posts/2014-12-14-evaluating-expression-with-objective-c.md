---
layout: post
title: 表达式求值：Objective-C 实现
date: 2014-12-14 16:15 +0800

---

* TOC
{:toc}
# 思路

说起表达式求值，首先想到严奶奶版的数据结构教材就有现成的算法。

```objective-c
OperandType EvaluateExpression(){
//算术表达式求值的算符优先算法，设OPTR和OPND分别为运算符合运算数栈
//OP为运算符集合
InitStack(OPTR); Push(OPTR,'#')
InitStack(OPND); c = getchar();
while(c != '#' || GetTop(OPTR) != '#'){
    if(!In(c, OP)){
        Push(OPND, c); 
        c = getchar();
    }//不是运算符则进栈
    else
        switch(Precede(GetTop(OPTR), c)){
            case '<': //栈顶元素优先级低，栈外运算符入栈
                Push(OPTR, c);
                c = getchar();
                break;
            case '=': //脱括号并接收下一字符
                Pop(OPTR,x);
                c = getchar();
                break;
            case '>': //栈顶元素优先级高，出栈并将运算结果入栈
                Pop(OPTR, theta);
                Pop(OPND, b); Pop(OPND, a);//出栈顺序:先b后a
                Push(OPND, Operate(a, theta, b);//计算式:a theta b
                break;
        }//switch
 }//while
   return GetTop(OPND);
}//EvaluateExpression
```

分析这个算法，可以整理出首先要实现一个栈，然后实现下面几个方法：

```objective-c
//判断输入的字符是运算符还是操作数
-(BOOL)isOperator:(NSString *)ch;

//判断输入的字符串中是否含有非法字符（除了数字和运算符之外）
-(BOOL)isNumberic:(NSString *) ch;

//比较栈外和栈内元素优先级
-(NSString *)comparePriority:(NSString *)inOptr outOptr:(NSString *)outOptr;

//计算opnd1 optr opnd2
-(double)calculate:(double)opnd1 opnd2:(double)opnd2 optr:(NSString *)optr;
    
//实现上述算法
-(NSString *)ExpressionCalculate:(NSString *)inputString;
```

# 栈的实现

数据存储：考虑到数据的动态变化，使用 NSMutableArray 来存放

初始化：直接用 `initWithCapacity:` 数组是空的，因为它只分配内存，得往里面塞点东西。

pop函数：最开始使用的下面的函数定义：

```objective-c
-(BOOL)pop:(NSString *)element stack:(Stack *)stack
```

但是 `element` 没办法传回去，比如：

```objective-c
NSString *a;
[pop:a stack:self.opnd]; //NSLog: a = null
```
所以修改了下，用一个 `property` 存放 `pop` 出来的元素，并赋上 `pop` 的返回值：

```objective-c
-(NSString *)pop:(Stack *)stack
//调用
NSString *a;
a = [self.opnd pop:self.opnd];
```
这就好啦。

# 函数实现

## comparePriority

1. 使用字典 NSDictionary，按栈内和栈外分两个字典存放优先级；
2. 优先级采用[这里](http://wenku.baidu.com/view/0f90ab92daef5ef7ba0d3c60.html?re=view)定义的一个表；
3. 两个优先级字典采用*类方法*初始化；
4. 优先级比较的时候，用到 NSString 与 NSInteger 的转化

```objective-c
 //NSInteger转化NSString类型：
 [NSString stringWithFormat: @"%d", NSInteger];

 //NSString转化 NSInteger类型：
 NSInteger myInteger = [myString integerValue];
```

## calculate

Objective-C 的 `switch` 不能直接用 NSString 类型作为 expression， 在 [StackOverflow](http://stackoverflow.com/questions/8161737/can-objective-c-switch-on-nsstring) 上搜索了一下解决办法，出于已经写的代码的考虑，采用第三个回答的方法。

```objective-c
-(double)calculate:(double)opnd1 opnd2:(double)opnd2 optr:(NSString *)optr
{
    NSArray *items = @[@"+", @"-", @"*", @"/"];
    int item = (int)[items indexOfObject:optr];
    
    switch (item) {
        case 0:
            return (opnd1 + opnd2);
            break;
        case 1:
            return (opnd1 - opnd2);
            break;
        case 2:
            return (opnd1 * opnd2);
            break;
        case 3:
            if (opnd2 == 0) {
                NSLog(@"除法，除数为0");
                return 0;
                break;
            }
            return (opnd1 / opnd2);
            break;
        default:
            NSLog(@"calculate default case");
            return 0;
            break;
    }
}
```

## expressionCalculate

### 处理输入的字符串

对其进行分割，按数字和运算符分别存储在不同的数组中。用到`componentsSeparatedByCharactersInSet`和`characterSetWithCharactersInString`。比如：

```objective-c
self.arrayToCalculate = [inputString componentsSeparatedByCharactersInSet: [NSCharacterSet characterSetWithCharactersInString:@"+-*/%"]];
```
只是有个问题，这样分割出来的数组中，会在原表达式出现括号的地方出现空白符，这是因为 `componentsSeparatedByCharactersInSet` 这个函数：

> Adjacent occurrences of the separator characters produce empty strings in the result. Similarly, if the string begins or ends with separator characters, the first or last substring, respectively, is empty.

处理分割 string 的时候，如果连续出现了用于分割的字符集合 set 中的两个字符，或者刚好 set 中的字符处于 string 的开始或结尾的地方，就会出现空白。
[StackOverflow](http://stackoverflow.com/questions/18057167/componentsseparatedbycharactersinset-issue)上有人给了一个处理第二种情况的办法，即采用`stringByTrimmingCharactersInSet`函数，将开始和结尾处的 set 中的字符trim 掉。

对于运算符，是对字符串进行逐个扫描，提取出 `+-*/` 等运算符号；如果采用提取数字的同样的方法，会出现 `(-` 这样、括号和运算符连在一起的情况，因为它们都在我给的运算符集合中，连在一起的时候自然不会分割。

### 先分割输入的字符串

我是将输入的字符串按字符串 `index` 一个一个取出来的，运算符还好，但如果是两位数三位数就惨了。所以事先将要进行运算的操作数取出来，每次在扫描到下一个字符是运算符时，将 `arrayToCalculate` 中的第一个元素取出入栈，然后在 `arrayToCalculate` 中删除第一个元素。

```objective-c
//删除下标为0的（即第一个元素）
[self.arrayToCalculate removeObjectAtIndex:0];     
```
### 一些类型转换

double 与 NSString 的相互转换

```objective-c
double num = [NSString doubleValue];
NSString * str = [NSString stringWithFormat:@"%f", num];
```
NSArray 与 NSMutableArray 的相互转换

```objective-c
//componentsSeparatedByCharactersInSet返回的是NSArray
NSArray * tempArray = [inputString componentsSeparatedByCharactersInSet: 
[NSCharacterSet characterSetWithCharactersInString:@"(+-*/%=)"]];
self.arrayToCalculate = [NSMutableArray arrayWithArray:tempArray];
```
char 转换为 NSString

```objective-c
//方式一：使用stringWithFormat
char c = 'a';
NSString *s = [NSString stringWithFormat:@"%c", c];

//方式二：使用-stringWithUTF8String:
char *ch = ……;
NSString *str = [NSString stringWithUTF8String:&ch];
```
使用 `stringWithUTF8String:`的话，str 末尾会包含 `\n`，看[我的问题](http://segmentfault.com/q/1010000002416827)就知道这是一个多么让人伤心的领悟。

另外 `stringWithFormat:` 是个挺万金油的方法，好像可以把大家都变成NSString 的样子。

最后一点是，使用 for 循环也可以让当前元素停下来。

注意当栈顶元素的优先级高于栈外元素时，要将栈顶元素弹出，此时要注意保存当前的栈外元素。算法中用的 while 循环，很容易保存，不接受下一个元素就可以了；如果是for循环的话，每次循环一定会递增，所以如果要保存当前栈外元素的话，将 `i—` 就可以啦。

## isNumberic



再次使用了 `NSCharacterSet` 这个类，用到了一个 `decimalDigitCharacterSet` 类方法，苹果文档里是这么说的：

> Returns a character set containing the characters in the category of Decimal Numbers.
> nformally, this set is the set of all characters used to represent the decimal values 0 through 9. These characters include, for example, the decimal digits of the Indic scripts and Arabic.

但是将 Digits 打印出来是这样的：`__NSCFCharacterSet: 0x7ffb4ad23b60`，不太明白是什么玩意儿。

# 效果图

![效果图](https://sfault-image.b0.upaiyun.com/307/286/3072867390-5489c133c2177)

[Repo](https://github.com/zchan0/Calculator/blob/master/MyCalculator/Calculate.h)