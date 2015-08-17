---
layout: post
title:  "Array decomposition in Ruby"
date:   2015-08-17 22:13:28
categories: jekyll update
tags:
- ruby
---
One of the things that impress me the most about Haskell is the combination of
expressive power and readability afforded by its' pattern matching. As a simple
example, here is a clever way of reversing a `Data.List` (roughly equivalent to
Ruby's `Array`):

{% highlight haskell linenos %}
reverse' :: [a] -> [a]
reverse' [] = []
reverse' (x:xs) = reverse'(xs) ++ [x]
-- reverse' [1, 2, 3, 4]
-- [4,3,2,1]
{% endhighlight %}

For Haskell infidels let me explain that with the `(x:xs)` syntax the `x`
variable represents the first element of the list while `xs` represents the
rest.

What I really like about this way of writing algorithms is that they read like
human-friendly description of a problem (definition of what it means for an
list to be reversed) rather than a machine-friendly list of steps how solve it.

Thankfully as of Ruby 1.9 I can use at least some of that goodness with the
splat operator and array decomposition. In the example below `head, *tail = arr`
is a rough equivalent of Haskell's `(x:xs)`.

{% highlight ruby linenos %}
def reverse(arr)
  return arr unless arr.length > 1
  head, *tail = arr
  reverse(tail) + [head]
end
reverse([1, 2, 3, 4])
#=> [4, 3, 2, 1]
{% endhighlight %}

That alone is pretty cool even tough there is no pattern matching possible here
so we're using an early return to get a similar boost in readability. And yet it
gets even better since array decomposition with the splat operator can get
fancier than that. Let's take your simple coding interview warm-up question -
checking for palindromes:

{% highlight ruby linenos %}
def palindrome?(word)
  return true if word.length < 2
  first, *middle, last = word.chars
  first == last && palindrome?(middle.join)
end
{% endhighlight %}

What's going on in line #3 above is that `first` is assigned to the first value
of the array, `last` to the last one and `middle` captures the entire space
inbetween, as an `Array`. In case there is nothing between the two extremes
`middle` becomes an empty container and evaluates to `''` after joining.

Once you get a handle on the splay operator and array decomposition you can
start using it to greatly improve the readability of your code, especially that
operating on arrays.
