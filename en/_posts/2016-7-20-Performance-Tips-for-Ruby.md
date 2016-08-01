---
layout: post
title: Performance Tips for Ruby
---

Some time ago I was reading the really good book [Ruby Performance Optimization - Why Ruby Is Slow, and How to Fix It](https://pragprog.com/book/adrpo/ruby-performance-optimization),
written by [Alexander Dymo](http://www.alexdymo.com/) (you should buy it!) and I crossed paths with some
interesting concepts that not so many people know or at least never really spent the necessary time to
fully understand them.

My goal with this post is to share a little of what I learned and put into practice, because as we know, performance
cannot be something left behind while we are developing.

## Garbage Collector - The Famous GC

Handle memory never is an easy task, but for this we can always count with the GC. Responsible for detecting allocated
objects that the system is not using anymore, GC takes back their memory, avoiding some problems we could have and also
giving more resources to the application.

The older versions of Ruby (older than 2.1) suffered a lot because their GC was not optimized, and anytime
it jumped into action the performance would drop in a considerable way.

I will use the `benchmark` library to illustrate how it works and the GC implications on a Ruby code.
As an example, this will be the first code to be tested:

{% highlight Ruby %}
require "benchmark"

test = Array.new(1000) { Array.new(1000) { 'Foo Bar' * 100 } }

time = Benchmark.realtime do
  test.each do |row|
    row.each do |val|
      val = val + val * 2
    end
  end
end

puts time.round(2)
{% endhighlight %}

The Ruby versions will be alternated to test how long it takes for each version to run this code.
By using [RVM](https://rvm.io/) we can do this:

{% highlight SQL %}
rvm use X
ruby performance_test.rb
{% endhighlight %}

Where `X` is the desired Ruby version. For example, `2.3.1`.

Let's check the results of this initial test(in seconds):

{% highlight Ruby %}
| 1.9.3-p551 | 2.0.0-p594 | 2.1.5 | 2.2.3 | 2.3.1 |
|------------|------------|-------|-------|-------|
|    12.1    |   15.58    |  4.35 |  3.24 |  3.21 |
{% endhighlight %}

Just remember that running examples once one by one is not the best way to measure performance.
The results can suffer from countless external interferences. But, as we can see, the results are so
discrepant that this is enough to notice that something is different between the versions.

Now I will run the same examples, but with a small change: disabling the GC. This is easy as inserting a line of
code before the block for the `benchmark`:

{% highlight Ruby %}
#...
test = Array.new(1000) { Array.new(1000) { 'Foo Bar' * 100 } }

GC.disable
time = Benchmark.realtime do
#...
{% endhighlight %}

And now the new results:

{% highlight Ruby %}
| 1.9.3-p551 | 2.0.0-p594 | 2.1.5 | 2.2.3 | 2.3.1 |
|------------|------------|-------|-------|-------|
|    3.15    |    3.37    |  3.11 |  2.13 |  2.10 |
{% endhighlight %}

As soon as we take a look we can check that now all the versions run this same code in a time that is almost equal.
This is the perfect way to show how GC is evolving with time. Older versions would spend a lot of time dealing with memory
usage problems while newer ones can solve them faster.

You must be thinking that the solution is shutting down GC forever. Please, do not do this. If this ever happens, you
will probably face terrible problems. The next part of this post will target small tips to avoid unnecessary GC calls,
making the app perform way better.

## The Performance Tips

### Modify Strings in Place

The situation here is this: almost every time using methods with `bang(!)` will save you. This happens because it is not necessary
to allocate memory copying a `String` that will be modified, and besides saving memory we can avoid calling GC for
the wrong reasons.

The initial code will be modified to measure how many times GC is called while the code is running. Also, the Ruby version
will be locked in `2.3.1`. Now the code and some results for the modified version not using any `bang`:

{% highlight Ruby %}
require "benchmark"

text = 'Foo Bar' * 1000 * 1000 * 1000

gc_on_start = GC.stat

time = Benchmark.realtime do
  text = text.reverse
end

gc_on_end = GC.stat

puts time.round(2)
puts gc_on_end[:count] - gc_on_start[:count]
{% endhighlight %}

Results:

{% highlight Ruby %}
36.57
1
{% endhighlight %}

So we got 1 call for GC and 36 seconds spent running the code. Changing the code we get:

{% highlight Ruby %}
time = Benchmark.realtime do
  text.reverse!
end
{% endhighlight %}

And the result is:

{% highlight Ruby %}
13.19
0
{% endhighlight %}

A great time and ZERO GC calls. Better right?

### The Same But With Arrays and Hashes

The same tip can be applied for `Arrays` and `Hashes`, and the reasons are the same. Both have multiple methods that happen
to use `bang` as well. For example, `map!` and `select!`.

{% highlight Ruby %}
require "benchmark"

array = Array.new(1000) { 'Foo Bar' * 1000 * 500  }

gc_on_start = GC.stat

time = Benchmark.realtime do
  array.map { |text| text.reverse }
end

gc_on_end = GC.stat

puts time.round(2)
puts gc_on_end[:count] - gc_on_start[:count]
{% endhighlight %}

When running this is what we get:

{% highlight Ruby %}
4.84
100
{% endhighlight %}

Yes, GC is on fire here with 100 participations on the show. Terrible right?
The good part is that we can replace `map` for `map!` and `reverse` for `reverse!`. Results:

{% highlight Ruby %}
2.11
0
{% endhighlight %}

Wow! No calls and we are once again fine. Besides this, the time spent decreased for some extra points.
With this it is possible to conclude that, whenever possible, the methods with `bang` are the way to go.

### Methods to Look For

Some methods can be especially harmful, mainly inside `Iterators`. For the sake of science we will look at the method `is_a?`:

{% highlight Ruby %}
require "benchmark"

array = Array.new(500_000) { 'Foo Bar' }

time = Benchmark.realtime do
  array.each do |text|
    text.is_a?(String)
  end
end

puts time.round(2)
{% endhighlight %}

The result here is `0.04` ms. You can think that this is nothing, but imagine a Rails application calling this type of comparison
way more times than this per request done? Things can (and will) stink.

There are some other methods that suffer from this same "problem", mainly `class` and `kind_of?`.
Most of the times try not to forget them inside some `Iterator` and everything will be fine.

## Conclusion

It is hard to be 100% focused on performance while developing, and we can say that [cache](https://en.wikipedia.org/wiki/Cache_(computing))
and scalability can play a role here. They are easy solutions for this problem almost all the time. Also, sometimes the best solution
in terms of performance is not the best solution in terms of code beauty.

But we need to understand and think about the consequences of our choices in this matter. It is important to understand how things
work behind the scenes and it is equally important to know the array of tools we have to help during our journey.
