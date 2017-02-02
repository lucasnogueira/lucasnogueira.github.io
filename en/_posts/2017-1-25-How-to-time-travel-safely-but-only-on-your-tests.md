---
layout: post
title: How to time travel safely, but only on your tests!
---
During your journey as developer you will need to manipulate time, and by this I mean freezing it, warping to the
future or back to the past. We use this when we want to test some code behavior determined by some date. For example,
we may need to test some status change of an object when the date is a past one.

As always, there are some gems that help our lives. I believe the most used one is the gem
[Timecop](https://github.com/travisjeffery/timecop). In the past [Delorean](https://github.com/bebanjo/delorean)
 was used as well, but for now I would say it looks a little bit outdated.

Besides those facts, it is valid to argue that the inclusion of another dependency in the system, for the
sake of handling a specific function on our tests, may be a bit overkill. Because of this, in the
[Rails 4.1 update](http://guides.rubyonrails.org/4_1_release_notes.html#active-support-notable-changes)
 a new helper was added to rescue us: [TimeHelpers](http://api.rubyonrails.org/classes/ActiveSupport/Testing/TimeHelpers.html).

Ok, I know. This version is old. The fact here is that a lot of people have **no knowledge** that this helper exists.
Sometimes it may be necessary to keep the Timecop inside the system, mostly because it has some other functions comparing
to the helper, but on my experience, the helper is enough in the majority of cases.

## How do I even start?

First, you must be using Rails 4.1 version or higher. Now you must decide if you want to turn this helper available
for all the tests or not. If you want to use it only on one test (and this is totally fine), you can do something
like this :

{% highlight Ruby %}
require 'spec_helper'

include ActiveSupport::Testing::TimeHelpers

describe Foo do
  before { travel_to(11.days.ago) }

  subject { true }

  it 'bars' do
    expect(subject).to be_truthy
  end
end
{% endhighlight %}

But if you think it is a better idea to let this helper available for all your tests, all you have to do is add one
line of code to your test configuration. If you are using Rspec it would be something like this:

{% highlight Ruby %}
RSpec.configure do |config|
  config.include ActiveSupport::Testing::TimeHelpers
end
{% endhighlight %}

## Features

With this you will have 3 new methods to use: `travel`, `travel_to` and `travel_back`. The first 2 are almost the same thing as
they are used to lock the time. When you use `travel` you will need to pass a specific time to be added on top of the current time.
For example:

{% highlight Ruby %}
Time.current
# => Wed, 25 Jan 2017 14:49:27 BRST -02:00

travel(1.day)

Time.current
# => Thu, 26 Jan 2017 14:49:27 BRST -02:00
{% endhighlight %}

With `travel_to` the time you passes will become the current time and will stay locked on it.

{% highlight Ruby %}
Time.current
# => Wed, 25 Jan 2017 14:56:37 BRST -02:00

travel_to(Time.new(2000, 03, 02, 20, 22, 14))

Time.current
# => Thu, 02 Mar 2000 14:56:37 BRST -02:00
{% endhighlight %}

As you can see both methods act almost the same: stubbing the return of some methods in order to lock the time.
Those methods are: `Time.now`, `Date.today`, and `DateTime.now`.

The last method of the helper, `travel_back`, is the one responsible to remove all the stubs made before. This way
methods like `Time.now` will return the real current time.

Both `travel` and `travel_to` accept a block as well:

{% highlight Ruby %}
Time.current
# => Wed, 25 Jan 2017 14:56:37 BRST -02:00

travel_to(Time.new(2000, 03, 02, 20, 22, 14)) do
  Time.current
  # => Thu, 02 Mar 2000 14:56:37 BRST -02:00
end

Time.current
# => Wed, 25 Jan 2017 14:56:37 BRST -02:00
{% endhighlight %}

If using a block the time will revert back when the block execution is over. If you are using those methods without a block,
depending on your tests, you may need to call `travel_back` to normalize the situation.

## Beware!

As stated at the top of this post, there are some differences if we compare the helper and the gem Timecop. The one you must have special attention is
the difference when we return time back to normal.

If you are using the helpers in sequence, for example:

{% highlight Ruby %}
travel_to(Time.new(2000, 03, 02, 20, 22, 14)) do
  travel_to(Time.new(2010, 03, 02, 20, 22, 14))
  # code
  travel_back
  # code
end
{% endhighlight %}

What will happen here is that when you call `travel_back` the time will return to the real current time, ignoring any other stub you could have made,
even if they were inside a block. We can understand it better if we check the [code for the method here](https://github.com/rails/rails/blob/27eccc27cbe987be04bb97b49aff1d7fd118634c/activesupport/lib/active_support/testing/time_helpers.rb#L123):

{% highlight Ruby %}
def travel_back
  simple_stubs.unstub_all!
end
{% endhighlight %}

All the `stubs` generated are removed! This will not happen on Timecop for example. The method that does the same on the gem is the `return`.
You can check the code for the whole class [here](https://github.com/travisjeffery/timecop/blob/master/lib/timecop/timecop.rb#L88).

Having a look at the method:

{% highlight Ruby %}
def return(&block)
  if block_given?
    instance.send(:return, &block)
  else
    instance.send(:unmock!)
    nil
  end
end
{% endhighlight %}

It is possible to notice that it has 2 different types of action here, where the first part will handle the situation where you have stubbed a previous
Time, enforcing that you will come back to this `stub` instead of the real current time. This will determinate if you need to use Timecop or not
on your tests.

## Conclusion

If you are cleaning your system or you cannot stand adding unnecessary dependencies to your code (me \o), here is a good example of native Rails feature
you can start using today. Just be aware of the differences. I recommend that you take a look at the GitHub page of Timecop and compare it to the helper
sou you can be safe and avoid future problems.
