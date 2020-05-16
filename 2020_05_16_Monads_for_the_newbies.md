Hi. There are plenty of articles in the Internet about Monads but nothing is simple enough to understand for a specific set of audience who used to predominantly written imperative Java methods and has just moving to FP especially Scala (like the past me several years back).

So in this article I try to explain Monads in a way that is easily understandable for them. If you already know what is a Monad, then don't read this and judge. 

For the intended audience, after reading this article, I suggest to have a look at this video from an expert about the same topic -> https://www.youtube.com/watch?v=t1e8gqXLbsU

I hope the video will be much clearer after reading this article. Without further ado, here we go..

But first lets think about a jargon I dropped earlier - "Imperative Programming"

**Imperative Programming** is what you do when you explicitly type instructions for things to happen inside the world of processors, memory and file system etc. Like define this variable, set this value to it, define another variable, increment a counter - all these technical instructions. For eg. Your quick sort or fibonacci program you learnt in school.

**Declarative Programming** on the other hand is what you do when you type code that says what you want to happen. It is more expressive than technical. For example, SQL statements like `SELECT name from people`. You just say what you want but imperative code for how it should happen is hidden inside the SQL query processor.

> Monads are data structures with built-in operations to let you express your code in a more readable declarative way than hard to process (in your mind) imperative way to do the same thing.

Don't worry if it didn't make sense. Let's look at examples..

Here is a Java method that is a sum of squares of even elements in a list (which may be null)..


```
    public int sumOfSquaresOfEvenElements(List<Integer> list) {
        if (list == null) {
            return 0;
        }
        int acc = 0;
        for (int i = 0; i < list.size(); i++) {
            int elem = list.get(i);
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
3. Iterate through the non-null list which may be empty.
4. For each element that is even, add its square value to the `acc`
5. Finally return acc


As a programmer, when you type such code several times, you'll start to think that some pieces are like boiler plates. You have to imperatively handle the null checking, the for loop for iteration for example. This type of code is more technical.


Compare this to the equivalent thing in Scala world.



