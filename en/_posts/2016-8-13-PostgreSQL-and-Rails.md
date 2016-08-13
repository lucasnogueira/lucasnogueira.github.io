---
layout: post
title: PostgreSQL and Rails
---
We have some good Open Source Database options available today, and this is really good. Relational or not,
what matters is that today we are so covered that sometimes it is hard to decide between so many options,
mainly because changing in the future will probably always lead to a lot of time consumption.

I have had the chance to work with a lot of databases, but I must say that [PostgreSQL](https://www.postgresql.org/)
(let's just call Postgres from now on) is a strong choice, capable of offering tools that others cannot
(or offer in a way that Postgres is better).

This post will show some useful features that Postgres can offer for us and how to use it together with Rails in order to
improve our applications. Every topic here could be a post itself, so I will try to summarize. Let's start!

### Json(b) Fields

In version 9.2 of Postgres, native support for JSON was added. After some time, version 9.4 brought to us the power of JSONB,
"Binary JSON". When we compare both, we can say that JSONB is slower in terms of writing speed, but it support indexes,
so the queries will have a huge bonus.

Was [Rails 4.2 version](http://guides.rubyonrails.org/4_2_release_notes.html#active-record-notable-changes) the one
responsible for adding JSONB support for the Postgres adapter. It is very simple to use it, for example:

{% highlight Ruby %}
create_table :flags do |t|
  t.string :name, null: false
  t.jsonb :colors
end
{% endhighlight %}

One thing you must bear in mind is that Rails will represent JSON and JSONB fields as a Hash. Check this:

{% highlight Ruby %}
flag = Flag.new(
        name: 'Brazil',
        colors: {
          blue: '3200FE',
          green: '007600',
          yellow: 'FEFE00',
          white: 'FFFFFF'
        })

flag.colors
#=>{"blue"=>"3200FE", "green"=>"007600", "yellow"=>"FEFE00", "white"=>"FFFFFF"}

flag.colors['white']
#=>FFFFFF
{% endhighlight %}

We can build amazing queries with JSONB fields, just do not forget to create the appropriated indexes.

For example, flags that have a specific type of green:

{% highlight Ruby %}
Flag.where('colors @> ?', {green: '007600'}.to_json)
{% endhighlight %}

Flags that have blue and yellow colors:

{% highlight Ruby %}
Flag.where('colors ?& array[:keys]', keys: ['blue', 'yellow'])
{% endhighlight %}

### Constraints

Here we will be using a feature called "check constraint" to assist Rails validations. Rails have some methods, like
`update_attribute`, that will bypass your validations, and this can cause some troubles if you are not aware. A constraint
is perfect in this situation, blocking the error to reach our database.

It is easy to set  a constraint inside a migration. Let's use our previous example again, the `Flag` model.
Imagine that, for some reason, we just want to add flags with a name that does not start with vowels.
So pretty much countries like Angola, Ukraine and etc. must fail our validation.

For this we create a migration:

{% highlight Ruby %}
class AddNameConstraintToFlags < ActiveRecord::Migration
  def up
    execute <<-SQl
      ALTER TABLE flags
      ADD CONSTRAINT no_flags_starting_with_vowels
      CHECK ( namel ~* '\A(?=[^aeiou])(?=[a-z])' )
    SQL
  end

  def down
    execute << SQL
      ALTER TABLE flags
      DROP CONSTRAINT no_flags_starting_with_vowels
    SQL
  end
end
{% endhighlight %}

Here we are using regular expression to match our desires. Now we can tell that our flags will respect the rules.
Anyway, this leads to a problem. Rails doesn't know how to handle check constraints, and because of this our
changes will not show up in the `db/schema.rb`. We will have to tell Rails to use SQL instead of plain Ruby when storing the schema.
This is as easy as adding a line inside `config/application.rb`:

`config.active_record.schema_format = :sql`

To end this you just need to delete the actual schema and recreate it with `bundle exec rake db:migrate`.

### Regular Expressions

We used it already and they can be very powerful. It is really easy to use regex in a query using Rails and Postgres. So, what you must know is:

 `~*` is the operator for case insensitive and `~` is the one for case sensitive.

 `!` is used for negation, so `!~` and `!~*`

 For Rails 4 and beyond the general query format is `Flag.where("name ~* ?", "regex")`

 This will let you make complex queries easily using the power Postgres provides!

## Conclusion
Postgres is a solid choice for open source database when pairing it with Ruby on Rails. We do have excellent choices nowadays,
but with the growing support and new features it is hard to say no in most of the cases.
