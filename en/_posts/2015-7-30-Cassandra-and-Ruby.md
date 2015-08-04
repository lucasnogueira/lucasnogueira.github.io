---
layout: post
title: Cassandra & Ruby
redirect_from:
  - /en/Cassandra-&-Ruby/
---

Recently I had the opportunity to play a little bit with the database [Apache Cassandra](http://cassandra.apache.org/).
For you that may find this name odd, Cassandra is a database known for its scalability. Its replication and data partitioning
across nodes are key features that make Cassandra safer and capable of recovering from eventual failures. To finish here,
companies like eBay, GitHub, Netflix and Reddit all use Cassandra so we are talking serious here.

Those previous statements lead me to try Cassandra out. In this post I will resume my adventure, telling a little about
how Cassandra works, some configurations and the connection with its own Ruby driver.

## More about Cassandra

_OBS: I will not cover too much details about the installation of Cassandra, but in the next steps we will be installing it together
with other things. If you have the desire to learn more about all the ways you can install the database I can direct you to this
[link](http://docs.datastax.com/en/cassandra/2.0/cassandra/install/installDeb_t.html).
To learn how to execute the CQL (Cassandra Query Language) direct from the terminal you must execute the [CQLSH](http://docs.datastax.com/en/cql/3.0/cql/cql_reference/cqlsh.html)._

The main structure here is the cluster. One cluster is composed of as many nodes as you want. Each node holds its own piece of
data and is nothing more than a separated Cassandra instance running somewhere. Later I will explain how to simulate different nodes
locally in an easy way, but in an ideal scenario you would want to have separated machines for each node.

The data division between those nodes is yours to decide. For example, you can choose to not have any type of data replication
across nodes (please do not do that) or you can replicate the information between 2 different nodes. A signature feature of Cassandra is
that the replicated data will not be put inside nodes that contain only replicated data. Instead, Cassandra will always elect the best way to
store replications between all the existing nodes.

To access our cluster for the first time we will have to create a keyspace to hold all the tables we will create one day.
It is now that we will choose how will happen the replication:

{% highlight SQL %}
CREATE KEYSPACE Test WITH REPLICATION = { 'class' : 'SimpleStrategy', 'replication_factor' : 2 };
{% endhighlight %}

The first option we are specifying here tells to Cassandra the way we will use it, where "SimpleStrategy" is the common choice for tests
and sandboxes. If you need something more powerful, to hold multiple data centers for example, it is necessary to use the option
"NetworkToplogyStrategy".

The second option here is the level of replication we are selecting, where 1 means that there will be no replication at all. We need
to pass a number that is lesser or equal to the number of nodes our cluster has. Choosing 2 in this example means that the replicated data
will be stored in another node besides the original one.

Now we are ready to go and use our keyspace in order to create our first table:

{% highlight SQL %}
USE Test;

CREATE TABLE users (
    user_name varchar PRIMARY KEY,
    password varchar,
    gender varchar,
    birth_year bigint
);
{% endhighlight %}

### Do not mess up with the Keys

In the last example we said that `user_name` is our `PRIMARY KEY`. But that exactly this statement means?

First, it means that all the users we create must have a unique `user_name`. But that is not all. The data will be organized
across the nosed based in the `PRIMARY KEY`. It is safe to say now that a singular `Primary Key` in Cassandra is the same thing as a
`PARTITION KEY`.

With this in mind I can tell you that Cassandra shares all the data between nodes from the same cluster using the partition key as a
rule to sort everything. But, what if there is a Composite Primary Key?

{% highlight SQL %}
create table keys (
    part_one text,
    part_two int,
    anything text,
    PRIMARY KEY(part_one, part_two)
);
{% endhighlight %}

Here in this example the `PARTITION KEY` is the first part of our `PRIMARY KEY` (`part_one`). The second part of the `PRIMARY KEY`
will be the one sorting the data in a partition, been called `CLUSTERING KEY` (`part_two`).

I will be ending the explanations of the main reasons that I started studying Cassandra. The rest of the post will focus on a better way
to use multiple nodes locally for test and learning purposes. In the end, I will show how to properly use Cassandra together with its Ruby
driver.

## I want a lot of nodes an I want them all locally

It is possible to maintain several Cassandra installations running separately and simulate nodes locally, but this can be a pain to maintain.
Thinking in a way that will not drive people crazy a script was made to make the things a little bit easier: [CCM](https://github.com/pcmanus/ccm).

To learn more about dependencies and how many ways possible you can install CCM I suggest to have a look at the link I provided in the last
paragraph, so you can follow a path and go.

With everything up and running lets jump to terminal and have some fun:

{% highlight SQL %}
ccm create test -v 2.0.5 -n 3
{% endhighlight %}

We are creating our cluster "test" and installing automatically Cassandra with version 2.0.5 for this. You are free to use the version
you like most in this part.

The last option holds the information about the number of nodes we will create together with the cluster. In our case, 3 nodes (node1, node2 and node3).

If you are using Mac take a moment and read please: it is necessary to create a new network interface for each new node besides the first:

{% highlight python %}
sudo ifconfig lo0 alias 127.0.0.2
sudo ifconfig lo0 alias 127.0.0.3
{% endhighlight %}

Here we are assuming that the given addresses are available. If you happen to see an error like this:

{% highlight YAML %}
Inet address 127.0.0.1:9042 is not available: [Errno 48] Address already in use
{% endhighlight %}

be safe and configure new interfaces for the nodes ok?

If everything is right, you can start the cluster with this command:

{% highlight python %}
ccm start
{% endhighlight %}

Now you can connect via cqlsh giving the name of a node (node1, etc.) that you wish to connect and apply any command.

## The Ruby Driver

Let us start with the installation of the gem:

{% highlight Ruby %}
gem install cassandra-driver
{% endhighlight %}

If you happen to have a started project just add this line to your Gemfile:

{% highlight Ruby %}
gem 'cassandra-driver'
{% endhighlight %}

With everything set in the previous session you will have a cluster with 3 nodes to use in your tests.
We will start connecting to the cluster now:

{% highlight Ruby %}
require 'cassandra'

cluster = Cassandra.cluster
{% endhighlight %}

It is possible to specify other options, like authentication and exactly what nodes do you wish to connect from a specific cluster:

{% highlight Ruby %}
cluster = Cassandra.cluster(
    username: username,
    password: password,
    hosts: ['10.0.1.1', '10.0.1.2', '10.0.1.3']
)
{% endhighlight %}

With our cluster saved inside the variable `cluster` we can now interact with the database. But first lets access the default
keyspace to create our own:

{% highlight Ruby %}
keyspace = 'system'

session = cluster.connect(keyspace)

ks = <<-KEYSPACE
  CREATE KEYSPACE Test
  WITH replication = {
    'class': 'SimpleStrategy',
    'replication_factor': 2
  }
KEYSPACE

session.execute(ks)
session.execute('USE Test')
{% endhighlight %}

We used the keyspace "system" to connect with the cluster, storing the returned object from this connection in the `session` variable.
After this we created another variable to save the definitions for our new keyspace.

With the `execute` method we can make any type of command, been it simple as a query or index creation. In this example, we executed the
creation of our new keyspace that was stored inside `ks`.

In the end, we executed a new command to use our new keyspace that we just created. From now all the tables we create will be available only
inside "Test" and that is what we are going to do now:

{% highlight Ruby %}
table = <<-TABLE
CREATE TABLE foo (
    id INT,
    anything VARCHAR,
    PRIMARY KEY (id)
    )
TABLE

session.execute(table)
{% endhighlight %}

### Prepared Statements

As you have imagined, to perform a query or a data insertion you will have to use the method `execute`:

{% highlight Ruby %}
session.execute('SELECT * FROM foo')
{% endhighlight %}

A cool thing we can do is to make use of the Prepared Statements. Those are ways to store commands that are defined previously
in a way that resembles variables.

{% highlight Ruby %}
insert = session.prepare('INSERT INTO foo (id, anything) VALUES (?, ?)')

get_all = session.prepare('SELECT * FROM foo')

session.execute(insert, arguments: [999, 'potatoes'])

session.execute(get_all)
{% endhighlight %}

This is an excellent way of reusing code, making everything cleaner. But do not overuse those benefits.

### Parallelism

It is possible to execute several commands in a parallel way easily with Cassandra. We saw before how `execute` works, but here we will use
the `execute_async` to execute commands asynchronously.

{% highlight Ruby %}
foo = [
  [1, 'potatoes'],
  [2, 'apples'],
  [3, 'tomatoes']
]

futures = foo.map do |(id, anything)|
  session.execute_async(insert, arguments: [age, username])
end
{% endhighlight %}

The variable `futures` will receive objects from the class `Future`. I will not enter in details about everything you can do with those objects,
but if it is your necessity to use something like this please read more [here](http://datastax.github.io/ruby-driver/api/future/).

### Pagination

Sometimes a query we make can return something that are bigger than we are ready to face. With this in mind, we can set the option `page_size`
on our queries to keep things clear:

{% highlight Ruby %}
foos = session.execute("SELECT * FROM foo WHERE anything = 'lol'", page_size: 100)
{% endhighlight %}

Making the size of our pages visible will give you more control over the application. To change from a page to another just use the method `next_page`.

### Compression

The last thing I will cover is a way to optimize requests with big responses. Cassandra supports two compression algorithms: Snappy and LZ4.
The official documentation says that you should use the second option if needed.

To use the compression algorithm together with the driver we need to install the respective gem.

The configuration will happen directly in the cluster that we are using:
{% highlight Ruby %}
cluster = Cassandra.cluster(compression: :lz4)
{% endhighlight %}

## Conclusions

My intention with this post was to cover in a direct way all my experiences with this technology, trying to make it easier and create something
like a guide covering my steps. I hope that with this more and more people will show interest in learning and trying this amazing database.

I want also to remind you that there are several official documentations around the web. I have used some during my learnings and during
this post. If you want to learn more about the Ruby driver for Cassandra click [here](http://datastax.github.io/ruby-driver/features/).

For now our show is over guys! Do not forget to come back ok? :stuck_out_tongue_winking_eye:
