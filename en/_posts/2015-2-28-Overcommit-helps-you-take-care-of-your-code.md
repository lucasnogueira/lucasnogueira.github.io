---
layout: en/post
title: Overcommit helps you take care of your code
---

Have you ever heard of the gem [Overcommit](https://github.com/causes/overcommit)? This gem goal is to help maintain
quality patterns, defined by you or by any programming language around the internet.

By definition, Overcommit is a tool to manage and configure Git hooks, and Git hooks are predetermined actions that
execute scripts when called. The gem contains some of those actions, but the most important ones are: `commit-msg`,
`pre-commit` and `post-checkout`. The first two actions will trigger while you are doing a commit while the last one
will be called when you switch branches via checkout. It is possible to create your own hooks, but this is subject
for a post on its own.

## What it does?

Let's move to what really matters: what Overcommit can do for me? Well, always you make a commit, for example, some
scripts will run in background. Those scrips will check from your commit message to your code syntax. Let's cover
the most important ones:

* __CommitMsg - HardTabs, RussianNovel TrailingPeriod__: Those scripts will take care of your commit message,
checking the length, formatting and characters.

* __Brakeman__: Will check for security flaws.

* __BundleCheck__: Checks dependencies on your Gemfile.

* __Coffee/Css/Html/Scss/Haml Lint__: Analyze the syntax of each of those types and many others, including some
programming languages like Java, Ruby, Python etc.

* __MergeConflicts__: You know that merge conflict you forgot to solve? Saved! :relieved:

* __PryBinding__: Forgot a `pry` inside your `Controller`? Stay calm, no one will make fun of you now. :joy:

* __Rubocop__: Probably the most important one for the Ruby programmers here. [Rubocop](https://github.com/bbatsov/rubocop)
is a static code analyzer that follows many of the guidelines presented in the community [Ruby Style Guide]
(https://github.com/bbatsov/ruby-style-guide). Rubocop is customizable in a way that you can choose what you want
you want to follow or not.

## Installing and Configuring

The journey is simple here:

`gem install overcommit`

Then on your repository:

`overcommit --install`

This will create a file named `.overcommit.yml` inside the repository. You can follow the instructions in this
file and in the github page of the gem to configure the hooks you like more and begin using Overcommit on your
project.

## Some tips

You can choose to block a commit to happen if it breaks any rules from the hooks or you can just allow it and show
what is not cool in the eyes of your analyzers. In the configuration file you will find something like this:

{% highlight YAML %}
PreCommit:
  Rubocop:
    on_warn: fail
{% endhighlight %}

In this way the comit will __not happen__ until the code you changed is in peace with Rubucop. If you think it is
necessary, it is possible to allow the commit to happen, but warnings will be displayed with the lines that are not
ok in your code. For this configuration you just need to change the last line of my example. Instead of `fail` you
change it to `warn`:

`on_warn: warn`

Besides that, all the other lines from a file that you have changed and are not ok with one of your hooks will
generate warnings, but your commit will be safe. Files that you did not touch will not generate any warning.

Maybe you think that it is the best to keep the `fail` in your config file. But, sometimes, you catch yourself in
a situation that you need to make a commit and some checking is not allowing you. No problem! You can use the
`SKIP` and say bye bye to restrictions. Just inform the hooks you want to skip and that's it.

`SKIP=rubocop git commit`

## The final result

![_config.yml]({{ site.baseurl }}/images/posts/three/overcommit-fail.png)

This first image shows my commit failing and the reasons are in red. In my configurations I am not able to finish
my actions while these "problems" still around or `SKIP`.

![_config.yml]({{ site.baseurl }}/images/posts/three/overcommit-success.png)

Here is another example. All my changes are ok with my hooks and my commit was a huge success! You can see that
Rubocop generated some warnings to show me there is code in the file I just changed that is not ok with
some patterns. Since the commit is already done, I can modify it later or just ignore. It is my call here now.

## Conclusion

Some projects are huge, team members change every time and the code is changed by dozens of programmers daily.
Sometimes is hard to keep some patterns due to physical and temporal barriers. If you and your team are suffering
from this and want a change to "keep in line", maybe the gem Overcommit can give you a hand or two.
