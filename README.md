# Homework: UDF Caching in Spark

In this assignment you'll implement UDF (user-defined function) result caching in [Apache Spark](http://spark.apache.org), which is a framework for distributed computing in the mold of MapReduce. This project will illustrate key concepts in data rendezvous and query evaluation, and you'll get some hands-on experience modifying Spark, which is widely used in the field. In addition, you'll get exposure to Scala, a JVM-based language that is gaining popularity for its clean functional style.

This assignment is due **Monday, October 19th at 11:59 PM** and is worth **10% of your final grade**. This project is an excellent opportunity to collaborate, and if you want to complete it **in pairs** and haven't filled out the CS186 Partner Google Form yet, ***Please fill out [this form](http://goo.gl/forms/P0OLVPg4La) by October 12th, at 11:59pm***.

Lastly, there is a lot of code in this directory. Please look [here](https://github.com/berkeley-cs186/course/tree/master/hw3#fetching-the-code) here to find the directory where the code is located.

## Assignment tasks

1. [Implement disk hash-partitioning](https://github.com/berkeley-cs186/course/tree/master/hw3#disk-hash-partitioning)
2. [Implement in-memory UDF caching](https://github.com/berkeley-cs186/course/tree/master/hw3#in-memory-udf-caching)
3. [Implement hash-partitioned UDF caching](https://github.com/berkeley-cs186/course/tree/master/hw3#disk-partitioned-udf-caching)

# Spark

Spark is an open-source distributed computing system written in [Scala](http://www.scala-lang.org). The project was started by Ph.D. students from the [AMPLab](https://amplab.cs.berkeley.edu) and is an integral part of the [Berkeley Data Analytics Stack](https://amplab.cs.berkeley.edu/software/) (BDAS—affectionately pronounced "bad-ass").

Like Hadoop MapReduce, Spark is designed to run functions over large collections of data, by supporting a simplified set of high-level data processing operations akin to the iterators we've been learning about in class. One of the most common uses of such systems is to implement parallel query processing in high level languages such as SQL. In fact, many recent research and development efforts in Spark have gone towards supporting a scalable and interactive relational database abstraction.

We'll be using, modifying, and studying aspects of Spark in this class to understand key concepts of modern data systems. More importantly you will see that the ideas we're covering in class -- some of which are decades old -- are still very relevant today. Specifically, we will be adding features to [Spark SQL](https://spark.apache.org/sql/).

One key limitation of Spark SQL is that it is currently a main-memory-only system.  As part of this class, we will extend it to include some out-of-core algorithms as well.

## Scala

Scala is a statically-typed language that supports many different programming paradigms. Its flexibility, power, and portability have become especially useful in distributed-systems research. 

Scala resembles Java, but it possesses a much broader set of syntax features to facilitate multiple paradigms. Knowing Java will help you understand some Scala code, but not much of it, and not knowing Scala will prevent you from fully taking advantage of its expressive power. Because you must write code in Scala, we strongly recommend you to acquire at least a passing familiarity with the language. 

[IntelliJ IDEA](https://www.jetbrains.com/idea/) tends to be the most commonly used IDE for developing in Spark. IntelliJ is a Java IDE that has a Scala (and vim!) plugin. There are also other options such as [Scala-IDE](http://scala-ide.org). 

You might find the following tutorials to be useful:
* [Twitter's Scala School](https://twitter.github.io/scala_school/)
* [Getting Started by Scala-Lang](http://www.scala-lang.org/documentation/getting-started.html)
* [A Scala Tutorial for Java Programmers](http://www.scala-lang.org/docu/files/ScalaTutorial.pdf)
* [TutorialsPoint's Scala Tutorial](http://www.tutorialspoint.com/scala/)

## Additional Reading for the Interested
* [Spark: Cluster Computing with Working Sets (HotCloud 2009)](https://amplab.cs.berkeley.edu/wp-content/uploads/2011/06/Spark-Cluster-Computing-with-Working-Sets.pdf)
* [Resilient Distributed Datasets: A Fault-Tolerant Abstraction for In-Memory Cluster Computing (NSDI 2012)](http://www.cs.berkeley.edu/~matei/papers/2012/nsdi_spark.pdf)
* [Shark: SQL and Rich Analytics at Scale (SIGMOD 2013)](http://www.cs.cmu.edu/~pavlo/courses/fall2013/static/papers/p13-xin.pdf)
* [Spark SQL: Relational Data Processing in Spark (SIGMOD 2015)](https://amplab.cs.berkeley.edu/wp-content/uploads/2015/03/SparkSQLSigmod2015.pdf) 

# Background and Framework

## UDFs (**U**ser **D**efined **F**unctions)

User-defined functions allow developers to define and exploit custom operations within expressions.  Imagine, for example, that you have a product catalog that includes photos of the product packaging.  You may want to register a user-defined function `extract_text` that calls an OCR algorithm and returns the text in an image, so that you can get queryable information out of the photos.  In SQL, you could imagine a query like this:
	
	SELECT P.name, P.manufacturer, P.price, extract_text(P.image), 
	  FROM Products P;
	 
The ability to register UDFs is very powerful -- it essentially turns your data processing framework into a general distributed computing framework.  But UDFs can often introduce performance bottlenecks, especially as we run them over millions of data items. 

If the input column(s) to a UDF contain a lot of duplicate values, it can be beneficial to improve performance by ensuring that the UDF is only called once per *distinct input value*, rather than once per *row*.  (For example in our Products example above, all the different configurations of a particular PC might have the same image.) In this assignment, we will implement this optimization.  We'll take it in stages -- first get it working for data that fits in memory, and then later for larger sets that require an out-of-core approach. We will use external hashing as the technique to "rendezvous" all the rows with the same input values for the UDF.

1. Implement disk-based hash partitioning.
1. Implement in-memory UDF caching.
1. Combine the above two techniques to implement out-of-core UDF caching.

If you're interested in the topic, the following paper will be an interesting read (including additional optimizations beyond what we have time for in this homework):

* [Query Execution Techniques for Caching Expensive Methods (SIGMOD 96)](http://db.cs.berkeley.edu/cs286/papers/caching-sigmod1996.pdf) 

## Project Framework

All the code you will be touching will be in three files -- `CS186Utils.scala`, `basicOperators.scala`, and `DiskHashedRelation.scala`. You might however need to consult other files within Spark or the general Scala APIs in order to complete the assignment thoroughly. Please make sure you look through *all* the provided code in the three files mentioned above before beginning to write your own code. There are a lot of useful functions in `CS186Utils.scala` as well as in `DiskHashedRelation.scala` that will save you a lot of time and cursing -- take advantage of them!

In general, we have defined most (if not all) of the methods that you will need. In that way, this is very much like a 61A project -- you just need to find the `// IMPLEMENT ME` comments and fill in the code. Correspondingly, the amount of code you will write is not very high -- the total staff solution is less than a 100 lines of code (not including tests). However, stringing together the right components in a memory-efficient way (i.e., not reading the whole relation into memory at once) will require some thought and careful planning.

## Some Terminology Differences

There are some potentially confusing differences between the terminology we use in class, and the terminology used in the SparkSQL code base:

* The "iterator" concept we learned in lecture is called a "node" in the SparkSQL code -- there are definitions in the code for UnaryNode and BinaryNode.  A query plan is called a SparkPlan, and in fact UnaryNode and BinaryNode extend SparkPlan (after all, a single iterator is a small query plan!)  You may want to find the file `SparkPlan.scala` in the SparkSQL source to see the API for these nodes.

* In some of the comments in SparkSQL, they also use the term "operator" to mean "node".  The file `basicOperators.scala` defines a number of specific nodes (e.g. Sort, Distinct, etc.).

* Don't confuse the [Scala interface `Iterator`](http://www.scala-lang.org/api/current/index.html#scala.collection.Iterator) with the iterator concept that we covered in lecture. The `Iterator` that you will be using in this project is a Scala language feature that you will use to implement your SparkSQL nodes.  `Iterator` provides an interface to Scala collections that enforces a specific API: the `next` and `hasNext` functions. 

# Setup


## `git` and GitHub

`git` is a *version control* system, helping you track different versions of your code, synchronize them across different machines, and collaborate with others. [GitHub](https://github.com) is a site which supports this system, hosting it as a service.

If you don't know much about `git`, we *strongly recommend* you to familiarize yourself with this system; you'll be spending a lot of time with it!
There are many guides to using `git` online - [here](http://git-scm.com/book/en/v1/Getting-Started) is a great one to read. 


## Setting up the virtual machine (optional)


To get started with the VM:

1. [Download and Install VirtualBox](https://www.virtualbox.org/wiki/Downloads) for your computer.  Current version is 5.0.6.
2. [Download and Install vagrant](https://www.vagrantup.com/downloads).  Current version is 1.7.4.

Now open a terminal on your machine, create a cs186 directory to work in, and build your VM with vagrant. This will download a large VM image from the cloud (more specifically, the VM image from last semester's iteration of CS 186!); make sure you've got some decent bandwidth.

    % mkdir cs186
    % cd cs186
    % vagrant init cs186-spring15/cs186spring15 // Not a typo!
    % vagrant up

If all goes well, your VM should now be up and running.  You can ssh into it as follows:

    % vagrant ssh

This is almost everything you need to know about vagrant, but you may want to [reference the vagrant docs](https://docs.vagrantup.com/v2/) occasionally.

Once you're logged into your VM, you should check out the git repository for the course within the VM:

    vagrant@precise64:~$ cd
    vagrant@precise64:~$ git clone https://github.com/berkeley-cs186/course.git
    
When you're done working you'll want to log out of the VM and shut it down:

    vagrant@precise64:~$ exit
    logout
    Connection to 127.0.0.1 closed.
    % vagrant halt

Next time you want to fire it up, just cd back to the cs186 directory you created and type `vagrant up` to reboot it.

## Setting up your repository and pulling the framework

You should first set up a remote **private** repository (e.g., spark-homework).

    $ cd ~

Clone your personal repository. It should be empty.

    $ git clone "https://github.com/xx/yy.git"

Enter the cloned repository, track the course repository and clone it.

    $ cd yy/
    $ git remote add course "https://github.com/UCLA-BDL/spark-homework.git"
    $ git pull course master

NOTE: Please do not be overwhelmed by the amount of code that is here. Spark is a big project with a lot of features. The code that we will be touching will be contained within one specific directory: sql/core/src/main/scala/org/apache/spark/sql/execution/. The tests will all be contained in sql/core/src/test/scala/org/apache/spark/sql/execution/

Push clone to your personal repository.

    $ git push origin master
    
Every time that you add some code, you can commit the modifications to the remote repository.

    $ git commit -m 'update to homework'
    $ git push origin master


### Receiving assignment updates

It may be necessary to receive updates to our assignment (even though we try to release them as "perfectly" as possible the first time). Assuming you set up the tracking correctly, you can simply run this following command to receive assignment updates:

    $ git pull course master


### Synced Folders in Vagrant (optional)

If you want to code on your local machine and then run your code in the VM without any `git push, git pull, etc.` hassle, vagrant has the capability of syncing local files to the guest machine!

Taken from Vagrant's website: "Synced folders enable Vagrant to sync a folder on the host machine to the guest machine, allowing you to continue working on your project's files on your host machine, but use the resources in the guest machine to compile or run your project. By default, Vagrant will share your project directory (the directory with the Vagrantfile) to /vagrant."

What this means is that all files that you move into the directory you set up vagrant in will appear in the `/vagrant` folder when you `vagrant ssh` into your VM. Here's what you can do to update code in your VM.

1. Move your wanted project and files into the directory that you set up vagrant in.
2. `vagrant ssh` into your VM and type `cd /vagrant` (type `ls` to make sure your folders are synced into the VM).
3. `cd` into your hw3 files and run a `make` command.

Voila! Hopefully it worked out! :)

## Building Spark

**Note:** Your VM might ask you to do this when you try to run `make`: `sudo apt-get install make`. Make sure you do it!

Once you have the pulled the code, `cd` into `{repo root}/hw3` and run `make compile`. The first time you run this command, it should take a while -- `sbt` will download all the dependencies and compile all the code in Spark (there's quite a bit of code). Once the initial assembly commands finish, you can start your project! (Future builds should not take this long -- `sbt` is smart enough to only recompile the changed files, unless you run `make clean`, which will remove all compiled class files.)

# Your Task

## Disk hash-partitioning

We have provided you skeleton code for `DiskHashedRelation.scala`. This file has 4 important things:
* `trait DiskHashedRelation` defines the DiskHashedRelation interface
* `class GeneralDiskHashedRelation` is our implementation of the `DiskedHashedRelation` trait
* `class DiskPartition` represents a single partition on disk
* `object DiskHashedRelation` can be thought of as an object factory that constructs `GeneralDiskHashedRelation`s

### Task #1: Implementing `DiskPartition` and `GeneralDiskHashedRelation`

First, you will need to implement the `insert`, `closeInput`, and `getData` methods in `DiskPartition` for this part. For the former two, the docstrings should provide a comprehensive description of what you must implement. The caveat with `getData` is that you *cannot* read the whole partition into memory in once. The reason we are enforcing this restriction is that there is no good way to enforce freeing memory in the JVM, and as you transform data to different forms, there would be multiple copies lying around. As such, having multiple copies of a whole partition would cause things to be spilled to disk and would make us all sad. Instead, you should stream *one block* into memory at a time.

At this point, you should be passing the tests in `DiskPartitionSuite.scala`.

### Task #2: Implementing `object DiskHashedRelation`

Your task in this portion will be to implement phase 1 of external hashing -- using a coarse-grained hash function to stream an input into multiple partition relations on disk. For our purposes, the `hashCode` method that every object has is sufficient for generating a hash value, and taking the modulo by the number of the partitions is an acceptable hash function. 

At this point, you should be passing all the tests in `DiskHashedRelationSuite.scala`.

## In-Memory UDF Caching

In this section, we will be dealing with `case class CacheProject` in `basicOperators.scala`. You might notice that there are only 4 lines of code in this class and, more importantly, no `// IMPLEMENT ME`s. You don't actually have to write any code here. However, if you trace the function call in [line 66](https://github.com/berkeley-cs186/course/blob/master/hw3/sql/core/src/main/scala/org/apache/spark/sql/execution/basicOperators.scala#L66), you will find that there are two parts of this stack you must implement in order to have a functional in-memory UDF implementation. 

### Task #3: Implementing `CS186Utils` methods

For this task, you will need to implement `getUdfFromExpressions` and the `Iterator` methods in `CachingIteratorGenerator#apply`. Please read the docstrings -- especially for `apply` -- closely before getting started.

After implementing these methods, you should be passing the tests in `CS186UtilsSuite.scala`.

Hint: Think carefully about why these methods might be a part of the Utils

## Disk-Partitioned UDF Caching

Now comes the moment of truth! We've implemented disk-based hash partitioning, and we've implemented in-memory UDF caching -- what is sometimes called [memoization](http://en.wikipedia.org/wiki/Memoization). Memoization is very powerful tool in many contexts, but here in databases-land, we deal with larger amounts of data than memoization can handle. If we have more unique values than can fit in an in-memory cache, our performance will rapidly degrade. 
Thus, we fall back to the time-honored databases tradition of divide-and-conquer. If our data does not fit in memory, then we can partition it to disk once, read one partition in at a time (think about why this works (hint: rendezvous!)), and perform UDF caching, evaluating one partition at a time. 

### Task 4: Implementing `PartitionProject`

This final task requires that you fill in the implementation of `PartitionProject`. All the code that you will need to write is in the `generateIterator` method. Think carefully about how you need to organize your implementation. You should *not* be buffering all the data in memory or anything similar to that. 

At this point, you should be passing ***all*** given tests.

### Thought Exercise 

There is no code you have to write here, but for your own edification, spend some time thinking about the following question:

One of Spark's main selling points is that it is "in-memory". What they mean is the following: When you string a number of Hadoop (or any other MapReduce framework) jobs together, Hadoop will write the results of each phase to disk and read them in again which is very expensive; Spark, on the other hand, maintains its data in memory. However, if our assumption is that if our data doesn't fit in memory, then why does Spark SQL not ship with disk-based implementation of these operators already? In this respect, why is Spark different from the "traditional" parallel relational databases we learn about in class?  There is no right answer to this question! 

## Testing

We have provided you some sample tests in `DiskPartitionSuite.scala`, `DiskHasedRelationSuite.scala`, `CS186UtilsSuite.scala` and `ProjectSuite.scala`. These tests can guide you as you complete this project. However, keep in mind that they are *not* comprehensive, and you are well advised to write your own tests to catch bugs. Hopefully, you can use these tests as models to generate your own tests. 

In order to run our tests, we have provided a simple Makefile. In order to run the tests for task 1, run `make t1`. Correspondingly for task, run `make t2`, and the same for all other tests. `make all` will run all the tests. 

### Assignment autograder

We will provide an autograder on this assignment. To run the autograder on your assignment, you must push to your submission branch: `release/hw3` (this ensures that you're submitting the assignment correctly in addition to getting an autograder email).

    $ git checkout -b release/hw3
    $ git push origin release/hw3

Our machines will e-mail you the results of the autograder within an hour. If you do not receive a response after an hour, please *first* double-check that all your files are in the right place and that you pushed a commit to your release branch, and *then* notify us that you haven't received a response. **Note that these tests are the same as the ones we've provided for you and are not comprehensive!**

### Assignment submission

To submit your assignment, as before, push a branch containing the commit you want us to grade to `release/hw3`. **Finally, remember that copying all or part of another person's work, or using reference material not specifically allowed, are forms of cheating and will not be tolerated.**

### Credits and Acknowledgements
Special thanks to Vikram Sreekanti!

**Good luck!**
