---
title: Transformation of sequences
---

# Transformation of sequences			

The values from the sequences we consume are not always in the format we need. Sometimes there is too much noise in the data so we strip the values down. Sometimes each value needs to be expanded either into a richer object or into more values. By composing operators, Rx allows you to control the quality as well as the quantity of values in the observable sequences you consume.

Up until now, we have looked at creation of sequences, transition into sequences, and, the reduction of sequences by filtering, aggregating or folding. In this chapter we will look at _transforming_ sequences. This allows us to introduce our third category of functional methods, _bind_. A bind function in Rx will take a sequence and apply some set of transformations on each element to produce a new sequence.

To review:

```
Ana(morphism) T --> IObservable<T>
Cata(morphism) IObservable<T> --> T
Bind IObservable<T1> --> IObservable<T2>
```

Now that we have been introduced to all three of our higher order functions, you may find that you already know them. Bind and Cata(morphism) were made famous by [MapReduce](http://en.wikipedia.org/wiki/MapReduce) framework from Google. Here Google refer to Bind and Cata by their perhaps more common aliases; Map and Reduce.

It may help to remember our terms as the `ABCs` of higher order functions.

```
Ana enters the sequence. T --> IObservable<T>
Bind modifies the sequence. IObservable<T1> --> IObservable<T2>
Cata leaves the sequence. IObservable<T> --> T
```

## Select							

The classic transformation method is `Select`. It allows you provide a function that takes a value of `TSource` and return a value of `TResult`. The signature for `Select` is nice and simple and suggests that its most common usage is to transform from one type to another type, i.e. `IObservable<TSource>` to `IObservable<TResult>`.

```csharp
IObservable<TResult> Select<TSource, TResult>(
    this IObservable<TSource> source, 
    Func<TSource, TResult> selector)
```

Note that there is no restriction that prevents `TSource` and `TResult` being the same thing. So for our first example, we will take a sequence of integers and transform each value by adding 3, resulting in another sequence of integers.

```csharp
var source = Observable.Range(0, 5);
source.Select(i=>i+3)
      .Dump("+3")
```

Output:

```
+3 --> 3
+3 --> 4
+3 --> 5
+3 --> 6
+3 --> 7
+3 completed
```

While this can be useful, more common use is to transform values from one type to another. In this example we transform integer values to characters.

```csharp
Observable.Range(1, 5);
          .Select(i =>(char)(i + 64))
          .Dump("char");
```

Output:

```
char --> A
char --> B
char --> C
char --> D
char --> E
char completed
```

If we really want to take advantage of LINQ we could transform our sequence of integers	to a sequence of anonymous types.

```csharp
Observable.Range(1, 5)
          .Select(i => new { Number = i, Character = (char)(i + 64) })
          .Dump("anon");
```

Output:

```
anon --> { Number = 1, Character = A }
anon --> { Number = 2, Character = B }
anon --> { Number = 3, Character = C }
anon --> { Number = 4, Character = D }
anon --> { Number = 5, Character = E }
anon completed
```

To further leverage LINQ we could write the above query using [query comprehension syntax](http://www.albahari.com/nutshell/linqsyntax.aspx).

```csharp
var query = from i in Observable.Range(1, 5)
            select new {Number = i, Character = (char) (i + 64)};

query.Dump("anon");
```

In Rx, `Select` has another overload. The second overload provides two values to the `selector` function. The additional argument is the element's index in the sequence. Use this method if the index of the element in the sequence is important to your selector function.

## Cast and OfType					

If you were to get a sequence of objects i.e. `IObservable<object>`, you may find it less than useful. There is a method specifically for `IObservable<object>` that will cast each element to a given type, and logically it is called `Cast<T>()`.

```csharp
var objects = new Subject<object>();
objects.Cast<int>().Dump("cast");
objects.OnNext(1);
objects.OnNext(2);
objects.OnNext(3);
objects.OnCompleted();
```

Output:

```
cast --> 1
cast --> 2
cast --> 3
cast completed
```

If however we were to add a value that could not be cast into the sequence then we get errors.

```csharp
var objects = new Subject<object>();
objects.Cast<int>().Dump("cast");
objects.OnNext(1);
objects.OnNext(2);
objects.OnNext("3");//Fail
```

Output:

```
cast --> 1
cast --> 2
cast failed --> Specified cast is not valid.
```

Thankfully, if this is not what we want, we could use the alternative extension method `OfType<T>()`.

```csharp
var objects = new Subject<object>();
objects.OfType<int>().Dump("OfType");
objects.OnNext(1);
objects.OnNext(2);
objects.OnNext("3");//Ignored
objects.OnNext(4);
objects.OnCompleted();
```

Output:

```
OfType --> 1
OfType --> 2
OfType --> 4
OfType completed
```

It is fair to say that while these are convenient methods to have, we could have created them with the operators we already know about.

```csharp
// source.Cast<int>(); is equivalent to
source.Select(i=>(int)i);

// source.OfType<int>();
source.Where(i=>i is int).Select(i=>(int)i);
```

## Timestamp and TimeInterval		

As observable sequences are asynchronous it can be convenient to know timings for when elements are received. The `Timestamp` extension method is a handy convenience method that wraps elements of a sequence in a light weight `Timestamped<T>` structure. The `Timestamped<T>` type is a struct that exposes the value of the element it wraps, and the timestamp it was created with as a `DateTimeOffset`.

In this example we create a sequence of three values, one second apart, and then transform it to a time stamped sequence. The handy implementation of `ToString()` on `Timestamped<T>` gives us a readable output.

```csharp
Observable.Interval(TimeSpan.FromSeconds(1))
          .Take(3)
          .Timestamp()
          .Dump("TimeStamp");
```

Output

```
TimeStamp --> 0@01/01/2012 12:00:01 a.m. +00:00
TimeStamp --> 1@01/01/2012 12:00:02 a.m. +00:00
TimeStamp --> 2@01/01/2012 12:00:03 a.m. +00:00
TimeStamp completed
```

We can see that the values 0, 1 &amp; 2 were each produced one second apart. An alternative to getting an absolute timestamp is to just get the interval since the last element. The `TimeInterval` extension method provides this. As per the `Timestamp` method, elements are wrapped in a light weight structure. This time the structure is the `TimeInterval<T>` type.

```csharp
Observable.Interval(TimeSpan.FromSeconds(1))
          .Take(3)
          .TimeInterval()
          .Dump("TimeInterval");
```

Output:

```
TimeInterval --> 0@00:00:01.0180000
TimeInterval --> 1@00:00:01.0010000
TimeInterval --> 2@00:00:00.9980000
TimeInterval completed
```

As you can see from the output, the timings are not exactly one second but are pretty close.

## Materialize and Dematerialize			

The `Timestamp` and `TimeInterval` transform operators can prove useful for logging and debugging sequences, so too can the `Materialize` operator. `Materialize` transitions a sequence into a metadata representation of the sequence, taking an `IObservable<T>` to an `IObservable<Notification<T>>`. The `Notification` type provides meta data for the events of the sequence.

If we materialize a sequence, we can see the wrapped values being returned.

```csharp
Observable.Range(1, 3)
          .Materialize()
          .Dump("Materialize");
```

Output:

```
Materialize --> OnNext(1)
Materialize --> OnNext(2)
Materialize --> OnNext(3)
Materialize --> OnCompleted()
Materialize completed
```

Note that when the source sequence completes, the materialized sequence produces an 'OnCompleted' notification value and then completes. `Notification<T>` is an abstract class with three implementations:

 * OnNextNotification
 * OnErrorNotification
 * OnCompletedNotification

`Notification<T>` exposes four public properties to help you discover it: `Kind`, `HasValue`, `Value` and `Exception`. Obviously only `OnNextNotification` will return true for `HasValue` and have a useful implementation of `Value`. It should also be obvious that `OnErrorNotification` is the only implementation that will have a value for `Exception`. The `Kind` property returns an `enum` which should allow you to know which methods are appropriate to use.

```csharp
public enum NotificationKind
{
    OnNext,
    OnError,
    OnCompleted,
}
```

In this next example we produce a faulted sequence. Note that the final value of the materialized sequence is an `OnErrorNotification`. Also that the materialized sequence does not error, it completes successfully.

```csharp
var source = new Subject<int>();
source.Materialize()
      .Dump("Materialize");

source.OnNext(1);
source.OnNext(2);
source.OnNext(3);

source.OnError(new Exception("Fail?"));
```

Output:

```
Materialize --> OnNext(1)
Materialize --> OnNext(2)
Materialize --> OnNext(3)
Materialize --> OnError(System.Exception)
Materialize completed
```

Materializing a sequence can be very handy for performing analysis or logging of a sequence. You can unwrap a materialized sequence by applying the `Dematerialize` extension method. The `Dematerialize` will only work on `IObservable<Notification<TSource>>`.

## SelectMany						

Of the transformation operators above, we can see that `Select` is the most useful. It allows very broad flexibility in its transformation output and can even be used to reproduce some of the other transformation operators. The `SelectMany` operator however is even more powerful. In LINQ and therefore Rx, the _bind_ method is `SelectMany`. Most other transformation operators can be built with `SelectMany`. Considering this, it is a shame to think that `SelectMany` may be one of the most misunderstood methods in LINQ.

In my personal discovery of Rx, I struggled to grasp the `SelectMany` extension method. One of my colleagues helped me understand `SelectMany` better by suggesting I think of it as <q>from one, select many</q>. An even better definition is <q>From one, select zero or more</q>. If we look at the signature for `SelectMany` we see that it takes a source sequence and a function as its parameters.

```csharp
IObservable<TResult> SelectMany<TSource, TResult>(
    this IObservable<TSource> source, 
    Func<TSource, IObservable<TResult>> selector)
```

The `selector` parameter is a function that takes a single value of `T` and returns a sequence. Note that the sequence the `selector` returns does not have to be of the same type as the `source`. Finally, the `SelectMany` return type is the same as the `selector` return type.

This method is very important to understand if you wish to work with Rx effectively, so let's step through this slowly. It is also important to note its subtle differences to `IEnumerable<T>`'s `SelectMany` operator, which we will look	at soon.

Our first example will take a sequence with the single value '3' in it. The selector function we provide will produce a further sequence of numbers. This result sequence will be a range of numbers from 1 to the value provided i.e. 3. So we take the sequence [3] and return the sequence [1,2,3] from our `selector` function.

```csharp
Observable.Return(3)
          .SelectMany(i => Observable.Range(1, i))
          .Dump("SelectMany");
```

Output:

```
SelectMany --> 1
SelectMany --> 2
SelectMany --> 3
SelectMany completed
```

If we modify our source to be a sequence of [1,2,3] like this...

```csharp
Observable.Range(1,3)
          .SelectMany(i => Observable.Range(1, i))
          .Dump("SelectMany");
```

...we will now get an output with the result of each sequence ([1], [1,2] and [1,2,3]) flattened to produce [1,1,2,1,2,3].

```
SelectMany --> 1
SelectMany --> 1
SelectMany --> 2
SelectMany --> 1
SelectMany --> 2
SelectMany --> 3
SelectMany completed
```

This last example better illustrates how `SelectMany` can take a `single` value and expand it to many values. When we then apply this to a `sequence` of values, the result is each of the child sequences combined to produce the final sequence. In both examples, we have returned a sequence that is the same type as the source. This is not a restriction however, so in this next example we return a different type. We will reuse the `Select` example of transforming an integer to an ASCII character. To do this, the `selector` function just returns a char sequence with a single value.

```csharp
Func<int, char> letter = i => (char)(i + 64);
Observable.Return(1)
          .SelectMany(i => Observable.Return(letter(i)));
          .Dump("SelectMany");
```

So with the input of [1] we return a sequence of [A].

```
SelectMany --> A
SelectMany completed
```

Extending the source sequence to have many values, will give us a result with many values.

```csharp
Func<int, char> letter = i => (char)(i + 64);
Observable.Range(1,3)
          .SelectMany(i => Observable.Return(letter(i)))
          .Dump("SelectMany");
```

Now the input of [1,2,3] produces [[A], [B], [C]] which is flattened to just [A,B,C].

```
SelectMany --> A
SelectMany --> B
SelectMany --> C
```

Note that we have effectively recreated the `Select` operator.

The last example maps a number to a letter. As there are only 26 letters, it would be nice to ignore values greater than 26. This is easy to do. While we must return a sequence for each element of the source, there aren't any rules that prevent it from being an empty sequence. In this case if the element value is a number outside of the range 1-26 we return an empty sequence.

```csharp
Func<int, char> letter = i => (char)(i + 64);
Observable.Range(1, 30)
          .SelectMany(
              i =>
              {
                if (0 < i &amp;&amp; i < 27)
                {
                    return Observable.Return(letter(i));
                }
                else
                {
                    return Observable.Empty<char>();
                }
            })
            .Dump("SelectMany");
```

Output:

```
A
B
C
...
X
Y
Z
Completed
```

To be clear, for the source sequence [1..30], the value 1 produced a sequence [A], the value 2 produced a sequence [B] and so on until value 26 produced a sequence [Z]. When the source produced value 27, the `selector` function returned the empty sequence []. Values 28, 29 and 30 also produced empty sequences. Once all the sequences from the calls to the selector had been fattened to produce the final result, we end up with the sequence [A..Z].

Now that we have covered the third of our three higher order functions, let us take time to reflect on some of the methods we have already learnt. First we can consider the `Where` extension method. We first looked at this method in the chapter on [Reducing a sequence](05_Filtering.html#Where). While this method does reduce a sequence, it is not a fit for a functional _fold_ as the result is still a sequence. Taking this into account, we find that `Where` is actually a fit for _bind_. As an exercise, try to write your own extension method version of `Where` using the `SelectMany` operator. Review the last example for some help...

An example of a `Where` extension method written using `SelectMany`:

```csharp
public static IObservable<T> Where<T>(this IObservable<T> source, Func<T, bool> predicate)
{
    return source.SelectMany(item =>
    {
        if (predicate(item))
        {
            return Observable.Return(item);
        }
        else
        {
            return Observable.Empty<T>();
        }
    });
}
```

Now that we know we can use `SelectMany` to produce `Where`, it should be a natural progression for you the reader to be able to extend this to reproduce other filters like `Skip` and `Take`.

As another exercise, try to write your own version of the `Select` extension method using `SelectMany`. Refer to our example where we use `SelectMany` to convert `int` values into `char` values if you need some help...

An example of a `Select` extension method written using `SelectMany`:

```csharp
public static IObservable<TResult> MySelect<TSource, TResult>(
    this IObservable<TSource> source, 
    Func<TSource, TResult> selector)
{
    return source.SelectMany(value => Observable.Return(selector(value)));
}
```

### IEnumerable<T> vs. IObservable<T> SelectMany	

It is worth noting the difference between the implementations of `IEnumerable<T>` `SelectMany` and `IObservable<T>` `SelectMany`. Consider that `IEnumerable<T>` sequences are pull based and blocking. This means that when an `IEnumerable<T>` is processed with a `SelectMany` it will pass one item at a time to the `selector` function and wait until it has processed all of the values from the `selector` before requesting (pulling) the next value from the source.

Consider an `IEnumerable<T>` source sequence of `[1,2,3]`. If we process that with a `SelectMany` operator that returns a sequence of `[x*10, (x*10)+1, (x*10)+2]`, we would get the `[[10,11,12]`, `[20,21,22]`, `[30,31,32]]`.

```csharp
private IEnumerable<int> GetSubValues(int offset)
{
    yield return offset * 10;
    yield return (offset * 10) + 1;
    yield return (offset * 10) + 2;
}
```

We then apply the `GetSubValues` method with the following code:

```csharp
var enumerableSource = new [] {1, 2, 3};
var enumerableResult = enumerableSource.SelectMany(GetSubValues);

foreach (var value in enumerableResult)
{
    Console.WriteLine(value);
}
```

The resulting child sequences are flattened into [10,11,12,20,21,22,30,31,32].

```
10
11
12
20
21
22
30
31
32
```

The difference with `IObservable<T>` sequences is that the call to the `SelectMany`'s `selector` function is not blocking and the result sequence can produce values over time. This means that subsequent 'child' sequences can overlap. Let us consider again a sequence of [1,2,3], but this time values are produced three second apart. The `selector` function will also produce sequence of [x*10, (x*10)+1, (x*10)+2] as per the example above, however these values will be four seconds apart.

To visualize this kind of asynchronous data we need to represent space and time.

### Visualizing sequences			

Let's divert quickly and talk about a technique we will use to help communicate the concepts relating to sequences. Marble diagrams are a way of visualizing sequences. Marble diagrams are great for sharing Rx concepts and describing composition of sequences. When using marble diagrams there are only a few things you need to know

1. a sequence is represented by a horizontal line 
2. time moves to the right (i.e. things on the left happened before things on the right)
3. notifications are represented by symbols:
 *. '0' for OnNext 
 *. 'X' for an OnError 
 *. '|' for OnCompleted 
1. many concurrent sequences can be visualized by creating rows of sequences

This is a sample of a sequence of three values that completes:

<div class="marble">
    <pre class="line">--0--0--0-|</pre>
</div>

This is a sample of a sequence of four values then an error:

<div class="marble">
    <pre class="line">--0--0--0--0--X</pre>
</div>

Now going back to our `SelectMany` example, we can visualize our input sequence by using values in instead of the 0 marker. This is the marble diagram representation of the sequence [1,2,3] spaced three seconds apart (note each character represents one second).

<div class="marble">
    <pre class="line">--1--2--3|</pre>
</div>

Now we can leverage the power of marble diagrams by introducing the concept of time and space. Here we see the visualization of the sequence produced by the first value 1 which gives us the sequence [10,11,12]. These values were spaced four seconds apart, but the initial value is produce immediately.

<div class="marble">
    <pre class="line">1---1---1|</pre>
    <pre class="line">0   1   2|</pre>
</div>

As the values are double digit they cover two rows, so the value of 10 is not confused with the value 1 immediately followed by the value 0. We add a row for each sequence produced by the `selector` function.

<div class="marble">
    <pre class="line">--1--2--3|</pre>
    <pre class="line"> </pre>
    <pre class="line" style="color: blue">  1---1---1|</pre>
    <pre class="line" style="color: blue">  0   1   2|</pre>
    <pre class="line"> </pre>
    <pre class="line" style="color: red">     2---2---2|</pre>
    <pre class="line" style="color: red">     0   1   2|</pre>
    <pre class="line"></pre>
    <pre class="line" style="color: green">        3---3---3|</pre>
    <pre class="line" style="color: green">        0   1   2|</pre>
</div>

Now that we can visualize the source sequence and its child sequences, we should be able to deduce the expected output of the `SelectMany` operator. To create a result row for our marble diagram, we simple allow the values from each child sequence to 'fall' into the new result row.

<div class="marble">
    <pre class="line">--1--2--3|</pre>
    <pre class="line"> </pre>
    <pre class="line" style="color: blue">  1---1---1|</pre>
    <pre class="line" style="color: blue">  0   1   2|</pre>
    <pre class="line"> </pre>
    <pre class="line" style="color: red">     2---2---2|</pre>
    <pre class="line" style="color: red">     0   1   2|</pre>
    <pre class="line"></pre>
    <pre class="line" style="color: green">        3---3---3|</pre>
    <pre class="line" style="color: green">        0   1   2|</pre>
    <pre class="line"></pre>
    <pre class="line">--<span style="color: blue">1</span>--<span style="color: red">2</span><span
        style="color: blue">1</span>-<span style="color: green">3</span><span style="color: red">2</span><span
            style="color: blue">1</span>-<span style="color: green">3</span><span style="color: red">2</span>--<span
                style="color: green">3</span>|</pre>
    <pre class="line">&nbsp;&nbsp;<span style="color: blue">0</span>&nbsp;&nbsp;<span
        style="color: red">0</span><span style="color: blue">1</span>&nbsp;<span style="color: green">0</span><span
            style="color: red">1</span><span style="color: blue">2</span>&nbsp;<span style="color: green">1</span><span
                style="color: red">2</span>&nbsp;&nbsp;<span style="color: green">2</span>|</pre>
    <pre class="line"></pre>
</div>

If we take this exercise and now apply it to code, we can validate our marble diagram. First our method that will produce our child sequences:

```csharp
private IObservable<long> GetSubValues(long offset)
{
    //Produce values [x*10, (x*10)+1, (x*10)+2] 4 seconds apart, but starting immediately.
    return Observable.Timer(TimeSpan.Zero, TimeSpan.FromSeconds(4))
                     .Select(x => (offset*10) + x)
                     .Take(3);
}
```

This is the code that takes the source sequence to produce our final output:

```csharp
// Values [1,2,3] 3 seconds apart.
Observable.Interval(TimeSpan.FromSeconds(3))
          .Select(i => i + 1) //Values start at 0, so add 1.
          .Take(3)            //We only want 3 values
          .SelectMany(GetSubValues) //project into child sequences
          .Dump("SelectMany");
```

The output produced matches our expectations from the marble diagram.

```
SelectMany --> 10
SelectMany --> 20
SelectMany --> 11
SelectMany --> 30
SelectMany --> 21
SelectMany --> 12
SelectMany --> 31
SelectMany --> 22
SelectMany --> 32
SelectMany completed
```

We have previously looked at the `Select` operator when it is used in Query Comprehension Syntax, so it is worth noting how you use the `SelectMany` operator. The `Select` extension method maps quite obviously to query comprehension syntax, `SelectMany` is not so obvious. As we saw in the earlier example, the simple implementation of just suing select is as follows:

```csharp
var query = from i in Observable.Range(1, 5)
            select i;
```

If we wanted to add a simple `where` clause we can do so like this:

```csharp
var query = from i in Observable.Range(1, 5)
            where i%2==0
            select i;
```

To add a `SelectMany` to the query, we actually add an extra `from` clause.

```csharp
var query = from i in Observable.Range(1, 5)
            where i%2==0
            from j in GetSubValues(i)
            select j;

// Equivalent to 
var query = Observable.Range(1, 5)
                      .Where(i=>i%2==0)
                      .SelectMany(GetSubValues);
```

An advantage of using the query comprehension syntax is that you can easily access other variables in the scope of the query. In this example we select into an anon type both the value from the source and the child value.

```csharp
var query = from i in Observable.Range(1, 5)
            where i%2==0
            from j in GetSubValues(i)
            select new {i, j};

query.Dump("SelectMany");
```

Output

```
SelectMany --> { i = 2, j = 20 }
SelectMany --> { i = 4, j = 40 }
SelectMany --> { i = 2, j = 21 }
SelectMany --> { i = 4, j = 41 }
SelectMany --> { i = 2, j = 22 }
SelectMany --> { i = 4, j = 42 }
SelectMany completed
```

This brings us to a close on Part 2. The key takeaways from this were to allow you the reader to understand a key principal to Rx: functional composition. As we move through Part 2, examples became progressively more complex. We were leveraging the power of LINQ to chain extension methods together to compose complex queries.

We didn't try to tackle all of the operators at once, we approached them in groups.

- Creation
- Reduction
- Inspection
- Aggregation
- Transformation

On deeper analysis of the operators we find that most of the operators are actually	specialization of the higher order functional concepts. We named them the ABC's of functional programming:

- Anamorphism, aka:
  - Ana
  - Unfold
  - Generate
- Bind, aka:
  - Map
  - SelectMany
  - Projection
  - Transform
- Catamorphism, aka:
  - Cata
  - Fold
  - Reduce
  - Accumulate
  - Inject

Now you should feel that you have a strong understanding of how a sequence can be manipulated. What we have learnt up to this point however can all largely be applied to `IEnumerable` sequences too. Rx can be much more complex than what many people will have dealt with in `IEnumerable` world, as we have seen with the `SelectMany` operator. In the next part of the book we will uncover features specific to the asynchronous nature of Rx. With the foundation we have built so far we should be able to tackle the far more challenging and interesting features of Rx.