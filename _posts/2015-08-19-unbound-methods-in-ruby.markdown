---
layout: post
title:  "Keeping a Baby quiet, or unbound methods in Ruby"
date:   2015-08-17 22:13:28
categories: ruby
---
As much as I find writing Python boring I often have to admit it is pretty well
thought out. Even things that used to annoy me to no end when I first got into
Python straight from the cosy Ruby world turn out to make a lot of sense in
practice. Take the seemingly half-assed approach to objects and the necessity
to use the `self` parameter in every method declaration inside Python classes.
What a waste of space, right? And yet it turns out that it can make your code
really expressive.

For example this:

{% highlight python %}
greeting = 'hello'
greeting.upper()
{% endhighlight %}

...can be rewritten as this:

{% highlight python %}
greeting = 'hello'
str.upper(greeting)
{% endhighlight %}

That's because every 'method' in Python is in fact a function which takes an
instance as it's first parameter. In most situations this is irrelevant and only
results in us having to put that annoying first parameter in every instance
method in our Python classes. But where it really shines is with inheritance
chains. Let's assume we have the following hierarchy:

{% highlight python %}
class Animal(object):
  def say(self):
    return '[awkward silence]'

class Human(Animal):
  def say(self):
    return 'I am hungry'

class Baby(Human):
  def say(self):
    return 'aaaaaaa! aaaaaa!! aaaaaaa!!!'
{% endhighlight %}

Now let's assume the worst - we're left with a crying `Baby`.

{% highlight python %}
>>> b = Baby()
>>> b.say()
'aaaaaaa! aaaaaa!! aaaaaaa!!!'
{% endhighlight %}

Turns out that with a bit of effort we can use the power that comes from the
abovementioned annoyance to make our `Baby` a really quiet, generic `Animal`:

{% highlight python %}
>>> b = Baby()
>>> Animal.say(b)
'[awkward silence]'
{% endhighlight %}

See what we've done there? And this is even sorta kinda type-safe! Let's assume
we have an object that implements the same method but is not related to our
`Baby` object. Like, an `Extraterrestrial`:

{% highlight python %}
class Extraterrestrial(object):
  def say(self):
    return 'We come in peace'
{% endhighlight %}

Even though Python is generally color-blind when it comes to type safety, it
does something surprisingly clever here:

{% highlight python %}
>>> b = Baby()
>>> Extraterrestrial.say(b)
TypeError: unbound method say() must be called with Extraterrestrial instance as
first argument (got Baby instance instead)
{% endhighlight %}

Kaboom. So you can't really have your `Baby` come in peace. Oops.

If you're still here you might be wondering if ye olde Ruby can pull the same
trick. Of course it can - the syntax may feel a bit awkward at first but the
concept (unbound methods) is the same. Let's translate our object hierarchy into
Ruby:

{% highlight ruby %}
class Animal
  def say; '[awkward silence]'; end
end

class Human < Animal
  def say; 'I am hungry'; end
end

class Baby < Human
  def say; 'aaaaaaa! aaaaaa!! aaaaaaa!!!'; end
end
{% endhighlight %}

So how do I keep my `Baby` quiet in Ruby?

{% highlight ruby %}
b = Baby.new
 => #<Baby:0x007febec9e9320>
b.say
 => "aaaaaaa! aaaaaa!! aaaaaaa!!!"
Animal.instance_method(:say).bind(b).call
 => "[awkward silence]"
{% endhighlight %}

Let's follow step by step what happened here:

{% highlight ruby %}
b = Baby.new
 => #<Baby:0x007febec980078>
method = Animal.instance_method(:say)
 => #<UnboundMethod: Animal#say>
fn = method.bind(b)
 => #<Method: Baby(Animal)#say>
fn.call
 => "[awkward silence]"
{% endhighlight %}

So there it is. I've created an instance of [`UnboundMethod`][unbound], then
bound an instance of `Baby` to it which resulted in a [`Method`][method] which
in turn presents the same calling interface as a [`Proc`][proc]. The reason I
could do it is the same as in Python - since `Baby` is still a valid `Animal`
there is no reason why I should not be able to run `Animal`'s methods on it,
even if they were overridden in the inheritance chain. An attempt for a `Baby`
to behave like an `Extraterrestrial` will fail here as well:

{% highlight ruby %}
class Extraterrestrial
  def say; 'We come in peace'; end
end
b = Baby.new
Extraterrestrial.instance_method(:say).bind(b)
=> TypeError: bind argument must be an instance of Extraterrestrial
{% endhighlight %}

That's it for today, happy binding!

[unbound]: http://ruby-doc.org/core-2.2.0/UnboundMethod.html
[method]: http://ruby-doc.org/core-2.2.0/Method.html
[proc]: http://ruby-doc.org/core-2.2.0/Proc.html
