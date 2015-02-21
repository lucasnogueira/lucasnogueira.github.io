---
layout: en/post
title: Some tips for Rspec users
---

[Rspec](http://rspec.info/) is a BDD framweork (Behaviour-Driven Development) supported for a huge community.
With this in mind is safe to say that new versions come often and full of new functionalities. This adds
flexibility and a better coverage of possible scenarios in your tests.

I have used Rspec for some years in the places I worked, and is notable that with lots of functionalities
it is hard for everyone to know everything. Here I compiled some tips that I learned during my journeys. I
hope they are useful to you as they are to me :wink:

## 1 - Only one expect per test block

It's ok, you probably can have a good reason to use 2 `expects` inside the same test block, I understand.
But, in the majority of cases, your test is wrong or you are overloading everything for no reason, and
this will turn your code into something hard to maintain and understand.

{% highlight ruby %}
context '#method' do
  let(:object) { create(:object) }

  before { subject.do_something_with(object) }

  # Not Good!!
  it 'does what I want with the object' do
    expect(object).name to eq  'changed'
    expect(object).email to eq 'changed@email.com'
  end

 # Better :)
  it 'changes the object name' do
    expect(object).name to eq 'changed'
  end

  it 'changes the object email' do
    expect(object).email to eq 'changed@email.com'
  end
end
{% endhighlight %}

You can go and use the one line syntax when you are able to. Recently there were some chances in the general
Rspec syntax, but you still can use the one line `it` with both, new and old syntaxes:

{% highlight ruby %}
# Old
it { should be_empty }

# New
it { is_expected.to be_empty }
{% endhighlight %}

## 2 - FactoryGirl syntax

In the previous example I used a `factory` for `object`. If you already use or plan to use Rspec one day,
the chances that you have crossed paths with FactoryGirl before are high, because it is a common replacement
for the traditional _fixtures_.

My advice is: you do not need to use FactoryGirl before all the methods you call. I used `create` directly
in the previous example, but sometime ago we would be using `FactoryGirl.create`.

A simple and generic configuration to enable this syntax to work in most cases is this:

{% highlight ruby %}
RSpec.configure do |config|
  config.include FactoryGirl::Syntax::Methods
end
{% endhighlight %}

If you ever need something different just visit this page [here](https://github.com/thoughtbot/factory_girl/blob/master/GETTING_STARTED.md).

## 3 - Let, let! and why to use

Instead of instantiating variables all around your test file, the Rspec provides an important tool: `let`.
In the first tip I created one `let` for `object` to use in my tests. The most important here is: `object`
will only be created when I call it, or been precisely, in the time I called `subject` passing `object`
to the method. From this moment I am able to use `object` until the end of my test without problems.

But sometimes you need more than this. Imagine for example that my `subject` has no direct contact with an object.
I could not call `object` in any place, but still need one created object so my tests can pass.

{% highlight ruby %}
let(:object) { create(:object) }

# What now?
before { subject.do_something_with_object }
{% endhighlight %}

In this case (and many others) we can use `let!`. This helper will create the object we need before any test example!

## 4 - Be true, be_truthy and the _matchers_ saga

Rspec has many _matchers_ that you can use, some already pointed out in my title. We can use opposite as well,
like `be false` and `be_falsey`. The problem here is, you may have noticed that their names are... almost the same?
How can we know what to use and when? :fearful:

Here is the answer: `be true` will check if the value of something is _literally_ `true`. `be_truthy` will check
if the value of something is truthy.

To the examples:

{% highlight ruby %}
# Works!
it { expect(true).to be true }
it { expect(false).to be false }

it { expect(true).to be_truthy }
it { expect('String').to be_truthy }
it { expect(false).to be_falsey }
it { expect(nil).to be_falsey }

# Fails!
it { expect('String').to be true }
it { expect(nil).to be false}

it { expect(nil).to be_truthy }
it { expect('String').to be_falsey }
{% endhighlight %}

## 5 - Wow, this syntax changed, what now?

If you used or still uses an old Rspec version you know that the newer versions support a new syntax now.
What will you do with your infinite thousand lines of test using `should` when the new hype is `expect`?

A gem for the rescue! [Trasnpec](https://github.com/yujinakayama/transpec) will help you change your tests
so you can happily move on to a newer version of Rspec. With it you can even customize what changes you
want to happen or not.

The good part is that even if you are already safe and sound in this syntax problem, you can
still run Transpec and check their suggestions. You may learn something as well!

I hope that these tips will be really useful for you. And if you happen to know something really cool
that is not covered here, please share with us in the comments bellow!
