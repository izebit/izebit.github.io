---
layout: post 
tags: 
  - algorithm 
title: The valid parentheses problem
image: /assets/img/parentheses.jpg
---

<img src="/assets/img/parentheses.jpg" alt="the valid parentheses problem"/>

<sub><sup>
Photo by <a href="https://unsplash.com/@claybanks" rel="nofollow">Clay Banks</a> on Unsplash
</sup></sub>


This problem is about checking all the parentheses are matched, i.e., each open parentheses has a closing parentheses in right order. 
It's also so-called "__balanced parenthesis problem__". 
Given a string that contains some symbols and some parentheses. We should say "__correct__" if all the parentheses are matched, otherwise "__invalid__".
To make it clear, let's look at some examples.   
`()` - correct    
`()(` - invalid   
`(()())` - correct   
`)()(` - invalid   

In order not to make the solutions cumbersome, in this article you don't observe cases with more than one type of parentheses. 
However, you can extend these solutions for these cases easily.

First of all, there are several solutions for this problem. The less sophisticated of them is an approach with using a stack.
We should check each symbol of the string, and we can have 3 cases:
1. A symbol is an open parentheses `(`. In this case, we put it into a stack
2. If a symbol is a closing parentheses `)`, we take an element from a stack if it is not empty, otherwise the answer is "__invalid__"
3. A symbol is a something else, so we just ignore it.

After all, we check the stack and if it is not empty, so the answer is "__invalid__" otherwise "__correct__".

The implementation is quite simple:
```java
public static boolean isCorrect(String str) {
    if(str == null ||  str.strim().length() == 0)
        return false;
    
    Stack<Character> stack = new Stack<>();
    for (int i = 0; i < str.length(); i++)
        if (str.charAt(i) == '(')
            stack.push('(');
        else if (str.charAt(i) == ')') {
            if (stack.isEmpty())
                return false;
            stack.pop();
        }
    
    return stack.isEmpty();
}
```

We can use the same approach but use a counter instead of a stack. By this way, we should check each symbol from the very beginning.
If a symbol is an open parentheses, we increase a counter. If a symbol is a closing parentheses, we decrease the counter. 
In case if a counter is less than zero, we can give an answer "__invalid__", otherwise keep checking symbols.
The solution is great to implement with using recursion. The code in _scala_ is below:

```scala
def isCorrect(str: String): Boolean = {
  @scala.annotation.tailrec
  def isCorrect(index: Int, result: Int): Boolean = {
    if (str.length == index)
      return result == 0

    str(index) match {
      case '(' => isCorrect(index + 1, result + 1)
      case ')' => result - 1 >= 0 && isCorrect(index + 1, result - 1)
      case _ => isCorrect(index + 1, result)
    }
  }

  isCorrect(0, 0)
}
```
The last solution is with using multithreading. 
The main point of it is 2 statements:
> (1) count of open parentheses of each prefix of the string must not be less than count of closing parentheses of the rest string.

and the second one is pretty obvious:

> (2) count of open parentheses of the whole string must equal count of closing parentheses of the whole string.  

Some details are necessary to understand it.   
Given:
$$ S_1, S_2, S_3 ... S_n $$ - parts of the original string, i.e., $$ \sum_{i=1}^{n}S_i = S  $$  is a concatenation of these _n_ parts, 
and it equals the whole string $$ S  $$.

Each $$ S_i $$ has 2 variables: 
- `finalValue` - final value. It is like a counter, at least it is estimated in the same way. 
  Each time when we meet `(` we increase it, and decrease when we meet `)` 
- `minValue` - minimum value is the minimum of final value when we estimate during the processing of the whole string.
  $$ minValue = \min_{1 \leq i \leq n}S_i $$

By this way, the two equations should be right:   

(1) $$ \sum_{j=1}^{i-1} finalValue_j + minValue_i \geq 0  $$ and

(2) $$ \sum_{j=1}^{i-1} finalValue_j = 0  $$ 

the implementation of this solution in scala is below:

```scala
def parIsCorrect(str: String) = {
  val threshold = 100

  private def traverse(start: Int, end: Int): (Int, Int) = {
    var finalValue = 0
    var minValue = 0

    var i = start
    while (i < end) {
      if (str(i) == '(')
        finalValue += 1
      else if (str(i) == ')')
        finalValue -= 1

      minValue = Math.min(finalValue, minValue)
      i += 1
    }

    (minValue, finalValue)
  }

  private def reduce(start: Int, end: Int): (Int, Int) = {
    if (end - start <= threshold)
      traverse(start, end)
    else {
      val middle = (end + start) / 2

      val firstTask = new Callable[(Int, Int)] {
        override def call(): (Int, Int) = reduce(start, middle)
      }
      val secondTask = new Callable[(Int, Int)] {
        override def call(): (Int, Int) = reduce(middle, end)
      }

      val results = ForkJoinPool.commonPool().invokeAll(List(firstTask, secondTask))

      val r1 = results(0).get()
      val r2 = results(1).get()

      var finalValue = 0
      var minValue = 0
      minValue = Math.min(minValue, finalValue + r1._1)
      finalValue += r1._2
      minValue = Math.min(minValue, finalValue + r2._1)
      finalValue += r2._2

      (minValue, finalValue)
    }
  }

  reduce(0, str.length) == (0, 0)
}
```
