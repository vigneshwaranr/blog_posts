**Intended audience:** People who come from Java background and just starting to explore Scala/FP and wanna learn what's a Monad in simplest way possible.

## Intro to Monad

There are plenty of articles in the Internet about Monads but nothing is simple enough to understand for such specific set of audience I mentioned above (like the past me several years back and like my newly joined colleagues).

So in this article I try to explain Monads in a way that is easily understandable for them. If you already know what is a Monad, then don't read this and judge. 

For the intended audience, after reading this article, I suggest to have a look at this video from an expert about the same topic -> https://www.youtube.com/watch?v=t1e8gqXLbsU

I hope the video will be much clearer after reading this article. Without further ado, here we go..

First lets think about these jargons - "Imperative Programming" and "Declarative Programming"

**Imperative Programming** is what you do when you explicitly type instructions for things to happen inside the world of processors, memory and file system etc. Like define this variable, set this value to it, define another variable, increment a counter - all these technical instructions. For eg. Your quick sort or fibonacci program you learnt in school.

**Declarative Programming** on the other hand is what you do when you type code that says what you want to happen. It is more expressive than technical. For example, SQL statements like `SELECT name from people`. You just say what you want but imperative code for how it should happen is hidden inside the SQL query processor.

**What's a Monad?**

> Monad is a fancy name for any data structures with "built-in imperative operations" that let you express your code in a more readable declarative way than the hard to process (in your mind.. not cpu) imperative way of doing the same thing.

Don't worry if it didn't make sense. Let's look at examples and then come back to this statement.

Here is a Java method that does a sum of squares of even elements in a list (the list could be null as well)..


```
    public int sumOfSquaresOfEvenElements(List<Integer> list) {
        if (list == null) {
            return 0;
        }
        int acc = 0;
        for (int elem: list) {
            if (elem % 2 == 0) {
                acc += (elem * elem);
            }
        }
        return acc;
    }
```

As an experienced Java 7 developer, this is what you'd have written. 

1. First check for null argument just to make it safe for people who use nulls, 
2. then define a mutable variable `acc` with 0 as initial value
3. Iterate through the non-null list which could be an empty list.
4. For each element that is even, add its square value to the `acc`
5. Finally return acc


As a programmer, when you type such code several times, you'll start to think that some pieces are like boiler plates. You have to imperatively handle the null checking, the for loop for iteration for example. This type of code appears more technical/imperative than expressive/declarative.


Compare this to the equivalent thing in Scala world.

![sum of squares](https://raw.githubusercontent.com/vigneshwaranr/blog_posts/master/screenshots/Monads_for_the_newbies/sumOfSquaresOfEvenElements.png)

Looks neat like a pipeline of operations, right? Here is the breakdown:

* We uplift the input argument into Option monad. If the input argument is null, it becomes a None object and if the argument is non-null, it becomes Some(thatValue) object. Both Some and None are sub types of Option.

Note that in Scala, you usually don't use null. But if you are exposing a scala library to a java program, it makes sense to make use of Option to verify argument is null or not.

If it is a method to be used in a full Scala application, you might actually skip that Option step.

* Since what we actually need is a List monad, the next statement converts it into a List (so it becomes `List[List[Int]]`.. look at the type annotation). Then the next statement `flatten`s it into a single `List[Int]`.

Note that if you didn't need to use Option, the code might actually look better like this. (Also I hate to use flatten cos it sounds more technical than being expressive)

```
  list
    .filter(_ % 2 == 0)
    .map(elem => elem * elem)
    .sum
```

*  Next we call `filter` operation on the List Monad to filter out some elements, then `map` on List Monad to convert remaining elements into their squares and finally `sum` on List Monad to get the sum.


So we have used two different Monads here: Option and List

Notice any common properties? 

* Both act as a ***container*** for the elements of interest.
* Both containers have the imperative operations like filtering, transforming (map) and aggregating (sum) "built-in" or "baked-in" INSIDE the data type itself so you don't have to do that. Now your code looks just like a chain of declarative operations that you want to happen.
* And the containers intelligently take care of the error scenarios which you have to handle imperatively in the Java code. 
  * For eg, there are 6 lines in `sumOfSquaresOfEvenElements`. 
  * If in the first line, Option found that the argument is null, it won't proceed to execute any of the rest of the lines
  * If the argument was non-null but an empty list instead, lines after `flatten` wouldn't run
  * If the list was non-empty but the result of `filter` is empty, lines after it wouldn't execute.

Monads should also have at the minimum these three laws:

1. A way to ***uplift*** an interested object into itself so it can be represented as the monad of that object. For example, Option(1), Try(1/0), Future.successful("response"), List(1, 2, 3) etc.

2. A way to ***extract*** the element out of Monad. Most often, that method is called `get`

3. A `flatMap` method so that it can accept a function and provide its contained value to it and get the result after applying that function while still retaining the monad container. (More on this coming..)


---

## Creating our own Monad


In this second section, I want to help you to be able to understand Professor Graham Hutton's video on Monad I linked above. He used Haskell and the syntax may be difficult for unfamiliar people so since Scala is more popular I'll use similar examples but in a slightly scala style way. I will show you how you can create a simple monad, then you will find it easier to watch the video afterwards.

The example he uses is division of two numbers using MayBe monad.

The problem with division is you cannot divide by 0. So I'm gonna create a Monad `MaybeDivideable` that has the baked-in operation to take care of this edge case leaving you to only worry about the successful division.


Here is the full code utilizing only the minimal laws (mentioned above) required for a Monad. (I just inspired from Option monad.)


```
sealed trait MaybeDivideable {
  def get: Int //Law 2
  def flatMap(f: Int => MaybeDivideable): MaybeDivideable //Law 3
}

object MaybeDivideable {
  def apply(value: Int): MaybeDivideable = 
  	if (value == 0) UnDivideable else Divideable(value) //Law 1
}

case class Divideable(value: Int) extends MaybeDivideable {
  require(value != 0, "value must not be 0")

  override def get: Int = value
  override def flatMap(f: Int => MaybeDivideable): MaybeDivideable = f(get)
}

case object UnDivideable extends MaybeDivideable {
  override def get: Int = throw new NoSuchFieldException
  override def flatMap(f: Int => MaybeDivideable): MaybeDivideable = UnDivideable
}
```

This is all it took to create my monad. Now my divide method will look like this if I prefer pattern matching style

```
def divide1(a: Int, b: Int): MaybeDivideable = {
  MaybeDivideable(a) match {
    case UnDivideable =>
      UnDivideable
    case Divideable(x) =>
      MaybeDivideable(b) match {
        case UnDivideable =>
          UnDivideable
        case Divideable(y) =>
          MaybeDivideable(x / y)
      }
  }
}
```

Quite ugly, yes. But this is just to show the equivalent of the following haskell example in the video.


![hutton1](https://raw.githubusercontent.com/vigneshwaranr/blog_posts/master/screenshots/Monads_for_the_newbies/hutton1.png)

And this was using only the apply and get methods. Let's try to take advantage of the flatMap method.

```
def divide2(a: Int, b: Int): MaybeDivideable = {
  val aMayBe = MaybeDivideable(a)
  val bMayBe = MaybeDivideable(b)
  aMayBe.flatMap { x =>
    bMayBe.flatMap { y =>
      MaybeDivideable(x / y)
    }
  }
}
```

Looks better than the horrible pattern matching one, right?

Mr. Hutton's Haskell example:

![hutton2](https://raw.githubusercontent.com/vigneshwaranr/blog_posts/master/screenshots/Monads_for_the_newbies/hutton2.png)


Still when you have loads of nesting, it may not look good. For-comprehension style is there to help us in that case!

For that to work I need to add `map` support in my Monad so lets do that.

```
sealed trait MaybeDivideable {
  def get: Int
  def flatMap(f: Int => MaybeDivideable): MaybeDivideable
  
  /* New map method */
  def map(f: Int => Int): MaybeDivideable
}
case class Divideable(value: Int) extends MaybeDivideable {
  require(value != 0, "value must not be 0")

  override def get: Int = value
  override def flatMap(f: Int => MaybeDivideable): MaybeDivideable = f(get)
  
  /* New map method */
  override def map(f: Int => Int): MaybeDivideable = MaybeDivideable(f(get))
}

case object UnDivideable extends MaybeDivideable {
  override def get: Int = throw new NoSuchFieldException
  override def flatMap(f: Int => MaybeDivideable): MaybeDivideable = UnDivideable
  
  /* New map method */
  override def map(f: Int => Int): MaybeDivideable = UnDivideable
}

object MaybeDivideable {
  def apply(value: Int): MaybeDivideable = if (value == 0) UnDivideable else Divideable(value)
}
```

Now using for-comprehension it looks great like:

```
def divide3(a: Int, b: Int): MaybeDivideable = {
  for {
    x <- MaybeDivideable(a)
    y <- MaybeDivideable(b)
  } yield (x / y)
}
```

Equivalent thing in haskell is called the do-notation:

![hutton3](https://raw.githubusercontent.com/vigneshwaranr/blog_posts/master/screenshots/Monads_for_the_newbies/hutton3.png)


So in this two part blog post, we have seen what are Monads and how to create one ourselves and use them. Hope it was useful for someone and please share some feedbacks in the comments.


Again, don't forget to watch this video (Professor Graham Hutton) if you are into Haskell as well -> https://www.youtube.com/watch?v=t1e8gqXLbsU


If you are college student, check out this video as well in which Professor Harold Abelson (one of the authors of SICP book) teaching the chapter "Compound data" (he never uses the word monads but the concepts are similar) -> https://www.youtube.com/watch?v=DrFkf-T-6Co&list=PLE18841CABEA24090&index=4

Thank you for patiently reading until here. :)