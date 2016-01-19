---
title: "First-Class Commands: An unexpectedly fertile design pattern"
layout: default
tags: [allonge, noindex]
---

This talk was given at [NDC London](http://ndc-london.com) on January 14, 2016. The complete slide deck is [here](https://speakerdeck.com/raganwald/first-class-commands-an-unexpectedly-fertile-design-pattern). The more important slides are shown here, along with some annotations explaining the ideas being presented. This is not a transcript, nor is it a blog post. Some slides are elided, especially those showing code. It will be most valuable to those who attended the talk or watch it when it is released on video.

![](/assets/images/command/001.png)

*"In object-oriented programming, the [command pattern](https://en.wikipedia.org/wiki/Command_pattern) is a behavioural design pattern in which an object is used to encapsulate all information needed to perform an action or trigger an event at a later time."*

In this talk, we'll review what the command pattern is, then look at some interesting applications. We'll see why what matters about the command pattern is the underlying idea that behaviour can be treated as a first-class entity in its own right.

![](/assets/images/command/002.png)

The command patterns was popularized in the 1994 book [Design Patterns: Elements of Reusable Object-Oriented Software][GoF]. But it's 2016. Why do we care? Why is it worth another look?

[GoF]: http://www.amazon.com/gp/product/B000SEIBB8/ref=as_li_tl?tag=raganwald001-20

When the command pattern was first popularized, most people wrote desktop software and client-server software. Distributed software was relatively exotic. So naturally, the examples given of the command pattern in use were often those applicable to single users. Like "undo," or writing macros, or perhaps displaying a progress bar.

But as we'll see, the underlying idea of the command pattern becomes particularly interesting when we consider parallel and distributed software, whether we are thinking of job queues, thread pools, or algorithms that provide eventual consistency across a distributed system.

In 2016, software is parallel and distributed by default. And the command pattern deserves another look, with fresh eyes.

![](/assets/images/command/003.png)

So let's have a look at the "canonical example" of the command pattern, working with mutable data. Here's one such example, chosen because it fits on a couple of sides:

{% highlight javascript %}
class Buffer {
  constructor (text = '') { this.text = text; }

  replaceWith (replacement, from = 0, to = this.text.length) {
    this.text = this.text.slice(0, from) +
                  replacement +
                  this.text.slice(to);
    return this;
  }

  toString () { return this.text; }
}

let buffer = new Buffer();

buffer.replaceWith(
  "The quick brown fox jumped over the lazy dog"
);
buffer.replaceWith("fast", 4, 9);
buffer.replaceWith("canine", 40, 43);
 //=> The fast brown fox jumped over the lazy canine
{% endhighlight %}

We have  buffer that contains some plain text, and it has a single behaviour, a `replaceWith` method that replaces a selection of the buffer with some new text. Insertions can be managed by replacing a zero-length selection, and deletions can be handled by replacing a selection with the empty string.

![](/assets/images/command/006.png)

Ten years ago, Steve Yegge described OOP as a [Kingdom of Nouns](http://steve-yegge.blogspot.ca/2006/03/execution-in-kingdom-of-nouns.html): Everything is an object and objects own their behaviours.

There is a very explicit idea that objects model entities in the real world, and methods model changes to those entities. Objects are "first-class:" They can be stored in variables, we can query them for their properties, and we can transform them into different states or different entities altogether.

![](/assets/images/command/007.png)

Many languages also permit us to treat methods as first-class entities. In Python, we can easily extract a bound method from an object. In Ruby, we can manipulate both bound and unbound methods. In JavaScript, methods are just functions.

Typically, treating methods as first-class entities is rarer than treating "nouns" as first-class entities, but it is possible. This forms the basis of meta-programming techniques like writing method decorators.

But the command pattern concerns itself with *invocations*. An invocation is a specific method, invoked on a specific receiver, with specific parameters:

![](/assets/images/command/008.png)

Classes are to instances as methods are to invocations.

![](/assets/images/command/009.png)

If an invocation was a first-class entity, we could store it in a variable or data structure. Let's try it:

![](/assets/images/command/010.png)

{% highlight javascript %}
class Edit {
  constructor (buffer, {replacement, from, to}) {
    this.buffer = buffer;
    Object.assign(this, {replacement, from, to});
  }

  doIt () {
    this.buffer.text =
      this.buffer.text.slice(0, this.from) +
      this.replacement +
      this.buffer.text.slice(this.to);
    return this.buffer;
  }
}

class Buffer {
  constructor (text = '') { this.text = text; }

  replaceWith (replacement, from = 0, to = this.text.length) {
    return new Edit(this, {replacement, from, to});
  }

  toString () { return this.text; }
}

let buffer = new Buffer(), jobQueue = [];

jobQueue.push(
  buffer.replaceWith(
    "The quick brown fox jumped over the lazy dog"
  )
);
jobQueue.push( buffer.replaceWith("fast", 4, 9) );
jobQueue.push( buffer.replaceWith("canine", 40, 43) );

while (jobQueue.length > 0) {
  jobQueue.shift().doIt();
}
 //=> The fast brown fox jumped over the lazy canine
{% endhighlight %}

Since we're taking an OO approach, we've created an `Edit` class that represents invocations. Each instance is an invocation, and thus we can create new invocations with `new Edit(...)` and actually perform the invocation with `.doIt()`.

In this example, we've create a job queue, deferring a number of invocations until we pop them off the queue and perform them. Note that "invoking" methods on a buffer no longer does anything: Instead, they return invocations we manipulate explicitly.[^promises]

[^promises]: This is vaguely related to working with promises in JavaScript, although we won't explore that as this is decidedly **not** a talk about JavaScript, it's a talk *in* JavaScript.

This is the canonical way to "do commands" in OOP: Make them instances of a class and perform them with a method. There are other ways to implement the command pattern, and it can be implemented in FP as well, but for our purposes this is enough to explore its applications.

![](/assets/images/command/012.png)

We can also query commends. Naturally, we do this by implementing methods that report on some critical characteristic, like a command's scope. For simplicity, we won't implement a `.scope()` method that reports the extent of an edit's election, since JavaScript encourages unencapsulated direct property access.

But we can report on the amount by which an edit lengthens or shortens a buffer:

{% highlight javascript %}
class Edit {

  netChange () {
    return this.from - this.to + this.replacement.length;
  }

}

let buffer = new Buffer();

buffer.replaceWith(
    "The quick brown fox jumped over the lazy dog"
).netChange();
 //=> 44

buffer.replaceWith("fast", 4, 9).netChange();
 //=> -1
{% endhighlight %}

This can be useful.

![](/assets/images/command/013.png)

First-class entities can also be *transformed*. And here we come to the most interesting application of commands. Here's a `.reversed()` method that returns the inverse of any edit:

{% highlight javascript %}
class Edit {

  reversed () {
    let replacement = this.buffer.text.slice(this.from, this.to),
        from = this.from,
        to = from + this.replacement.length;
    return new Edit(buffer, {replacement, from, to});
  }
}

let buffer = new Buffer(
  "The quick brown fox jumped over the lazy dog"
);

let doer = buffer.replaceWith("fast", 4, 9),
    undoer = doer.reversed();

doer.doIt();
  //=> The fast brown fox jumped over the lazy dog

undoer.doIt();
  //=> The quick brown fox jumped over the lazy dog
{% endhighlight %}

![](/assets/images/command/014.png)

Let's put our storing and transforming together. Instead of returning a command from the `replaceWith` method, we'll create a `doer` command, and push its reverse onto a `history` stack. We'll then invoke `doer.doIt()` to actually perform the replacement on the buffer:

{% highlight javascript %}
class Buffer {

  constructor (text = '') {
    this.text = text;
    this.history = [];
    this.future = [];
  }

}

class Buffer {

  replaceWith (replacement, from = 0, to = this.length()) {
    let doer = new Edit(this, {replacement, from, to}),
        undoer = doer.reversed();

    this.history.push(undoer);
    this.future = [];
    return doer.doIt();
  }

}
{% endhighlight %}

![](/assets/images/command/015.png)

Implementing `undo` is straightforward: Pop an `undoer` from the stack, create a `redoer` for later, push the `redoer` onto a `future` stack, and invoke the undoer:

{% highlight javascript %}
class Buffer {

  undo () {
    let undoer = this.history.pop(),
        redoer = undoer.reversed();

    this.future.unshift(redoer);
    return undoer.doIt();
  }

}

let buffer = new Buffer(
  "The quick brown fox jumped over the lazy dog"
);

buffer.replaceWith("fast", 4, 9)
  //=> The fast brown fox jumped over the lazy dog

buffer.replaceWith("canine", 40, 43)
  //=> The fast brown fox jumped over the lazy canine

buffer.undo()
  //=> The fast brown fox jumped over the lazy dog

buffer.undo()
  //=> The quick brown fox jumped over the lazy dog
{% endhighlight %}

![](/assets/images/command/016.png)

Redoing something we've undone is now simple:

{% highlight javascript %}
class Buffer {

  redo () {
    let redoer = this.future.shift(),
        undoer = redoer.reversed();

    this.history.push(undoer);
    return redoer.doIt();
  }

}

buffer.redo()
  //=> The fast brown fox jumped over the lazy dog

buffer.redo()
  //=> The fast brown fox jumped over the lazy canine
{% endhighlight %}

And again, its reverse goes onto the history so we can toggle back and forth between undoing and redoing.

![](/assets/images/command/017.png)

Like the slide says, this is the basic idea you'll find in the GoF book as well as in 1980s tomes on OO programming. I recall an Object Pascal book using this pattern to implement undo within the [MacApp](https://en.wikipedia.org/wiki/MacApp) framework in the late 1980s.

![](/assets/images/command/018.png)

Our example hits all three of the characteristics of invocations as first-class entities. But that isn't really enough to "provoke our intellectual curiosity." So let's consider a more interesting direction.

### part ii: playing with time

We begin by asking a question.

![](/assets/images/command/019.png)

Recall this code for replacing text in a buffer:

{% highlight javascript %}
class Buffer {

  replaceWith (replacement, from = 0, to = this.length()) {
    let doer = new Edit(this, {replacement, from, to}),
        undoer = doer.reversed();

    this.history.push(undoer);
    this.future = [];
    return doer.doIt();
  }

}
{% endhighlight %}

Note that when we perform a replacement, we execute `this.future = []`, throwing away any "redoers" we may have accumulated by undoing edits.

![](/assets/images/command/020.png)

Let's try not throwing it away:

{% highlight javascript %}
class Buffer {

  replaceWith (replacement, from = 0, to = this.length()) {
    let doer = new Edit(this, {replacement, from, to}),
        undoer = doer.reversed();

    this.history.push(undoer);
    // this.future = [];
    return doer.doIt();
  }

}

let buffer = new Buffer(
  "The quick brown fox jumped over the lazy dog"
);

buffer.replaceWith("fast", 4, 9);
  //=> The fast brown fox jumped over the lazy dog

buffer.undo();
  //=> The quick brown fox jumped over the lazy dog

buffer.replaceWith("My", 0, 3);
  //=> My quick brown fox jumped over the lazy dog
{% endhighlight %}

We've performed a replacement, then we've undone the replacement, restoring the buffer to its original state. Then we performed a different replacement. But since our code no longer discards the future, a `redoer` is still in `this.future`.

![](/assets/images/command/021.png)

Unfortunately, the result is not what we expect semantically:

![](/assets/images/command/022.png)

What went wrong?

![](/assets/images/command/023.png)

As the illustration shows, when we first performed `.replaceWith('fast', 4, 9)`, it replaced the characters `q`, `u`, `i`, `c`, and `k`, because those were in the selection between `4` and `9` of the buffer.

Our `redoer` in the `future` performs this same replacement, but now that we've invoked `.replaceWith('My', 0, 3)`, the characters in the selection between `4` and `9` are now `u`, `i`, `c`, `k`, and a blank space.

Invoking `.replaceWith('My', 0, 3)` has moved the part of the buffer we semantically want to replace.

![](/assets/images/command/024.png)

If we step through the invocations, we can see that when we first invoke `.replaceWith('fast', 4, 9)`, no other edits were invoked before it.

![](/assets/images/command/025.png)

![](/assets/images/command/026.png)

![](/assets/images/command/027.png)

![](/assets/images/command/028.png)

Then after undoing it and invoking `.replaceWith('My', 0, 3)`, we have created a situation where `.replaceWith('My', 0, 3)` is now *before* `.replaceWith('fast', 4, 9)` in the `future`. If we invoke it, we see this clearly as it moves to the past, but it is now preceded by `.replaceWith('My', 0, 3)`:

![](/assets/images/command/029.png)

![](/assets/images/command/030.png)

The problem is that commands mutating a model have a semantic dependency on all of the commands that have mutated the model in the past. If you change the order of commands, they may no longer be semantically valid. In some cases, they could even become logically invalid.

![](/assets/images/command/031.png)
![](/assets/images/command/032.png)
![](/assets/images/command/033.png)
![](/assets/images/command/034.png)
![](/assets/images/command/035.png)
![](/assets/images/command/036.png)
![](/assets/images/command/037.png)
![](/assets/images/command/038.png)
![](/assets/images/command/039.png)
![](/assets/images/command/040.png)
![](/assets/images/command/041.png)
![](/assets/images/command/042.png)
![](/assets/images/command/043.png)
![](/assets/images/command/044.png)
![](/assets/images/command/045.png)
![](/assets/images/command/046.png)
![](/assets/images/command/047.png)
![](/assets/images/command/048.png)
![](/assets/images/command/049.png)
![](/assets/images/command/050.png)
![](/assets/images/command/051.png)
![](/assets/images/command/052.png)

### afterword

As is often the case, the slides by themselves can only hint at the substance of the presentation: Typically, people read much faster than people speak, so if the slides convey the ideas on their own, the audience will quickly become bored with the speaker's delivery.

Slides are also a poor way to convey detailed information. It is difficult to put a lot of code on a slide, for example, and if you do put a lot of any kind of information on a slide, different people in the audience will process it at different rates, so somebody is bound to grasp it quickly and be bored, while others will still be trying to work it out when the speaker moves along.

For this reason, I prefer to compose talks in a completely different style than blog posts. Blog posts can have longer sections of code, and people can move along at their own pace. Blog posts can convey technical ideas much more efficiently than presentations, so my goal with a presentation is simply to get people interested enough in the subject to seek out blog posts, books, or screen casts for further study.

Which presents me with a dilemma: After giving a talk at a conference, what good are the slides? Even if a video is published online, what good is that compared to rewriting the presentation as a blog post?

### image credits

[https://www.flickr.com/photos/fatedenied/7335413942](https://www.flickr.com/photos/fatedenied/7335413942)
[https://www.flickr.com/photos/fatedenied/7335413942](https://www.flickr.com/photos/fatedenied/7335413942)
[https://www.flickr.com/photos/mwichary/2406482529](https://www.flickr.com/photos/mwichary/2406482529)
[https://www.flickr.com/photos/tompagenet/8580371564](https://www.flickr.com/photos/tompagenet/8580371564)
[https://www.flickr.com/photos/ooocha/2869485136](https://www.flickr.com/photos/ooocha/2869485136)
[https://www.flickr.com/photos/oskay/2550938136](https://www.flickr.com/photos/oskay/2550938136)
[https://www.flickr.com/photos/baccharus/4474584940](https://www.flickr.com/photos/baccharus/4474584940)
[https://www.flickr.com/photos/micurs/4906349993](https://www.flickr.com/photos/micurs/4906349993)
[https://www.flickr.com/photos/purdman1/2875431305](https://www.flickr.com/photos/purdman1/2875431305)
[https://www.flickr.com/photos/daryl_mitchell/15427050433](https://www.flickr.com/photos/daryl_mitchell/15427050433)
[https://www.flickr.com/photos/the00rig/3753005997](https://www.flickr.com/photos/the00rig/3753005997)
[https://www.flickr.com/photos/robbie1/8656027235](https://www.flickr.com/photos/robbie1/8656027235)
[https://www.flickr.com/photos/mwichary/2406489333](https://www.flickr.com/photos/mwichary/2406489333)
[https://www.flickr.com/photos/pedrosimoes7/17386505158](https://www.flickr.com/photos/pedrosimoes7/17386505158)
[https://www.flickr.com/photos/a-barth/2846621384](https://www.flickr.com/photos/a-barth/2846621384)
[https://www.flickr.com/photos/mleung311/9468927282](https://www.flickr.com/photos/mleung311/9468927282)
[https://www.flickr.com/photos/bludgeoner86/5590795033](https://www.flickr.com/photos/bludgeoner86/5590795033)
[https://www.flickr.com/photos/49024304@N00/](https://www.flickr.com/photos/49024304@N00/)
[https://www.flickr.com/photos/29143375@N05/4575806708](https://www.flickr.com/photos/29143375@N05/4575806708)
[https://www.flickr.com/photos/30239838@N04/4268147953](https://www.flickr.com/photos/30239838@N04/4268147953)
[https://www.flickr.com/photos/benetd/4429314827](https://www.flickr.com/photos/benetd/4429314827)
[https://www.flickr.com/photos/shimgray/2811100997](https://www.flickr.com/photos/shimgray/2811100997)
[https://www.flickr.com/photos/wordridden/4308645407](https://www.flickr.com/photos/wordridden/4308645407)
[https://www.flickr.com/photos/sidelong/18620995913](https://www.flickr.com/photos/sidelong/18620995913)
[https://www.flickr.com/photos/stawarz/3848824508](https://www.flickr.com/photos/stawarz/3848824508)
[https://www.flickr.com/photos/mwichary/3338901313](https://www.flickr.com/photos/mwichary/3338901313)

### notes