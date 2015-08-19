---
layout: post
title:  "Unbound methods in Go"
date:   2015-08-19 22:13:28
categories: golang
---

<img src='/assets/gopher.png' style='width: 80px; float: left; margin: 0 20px 20px' />
In my previous post I wrote about unbound methods in Python and Ruby and showed
how these can be used to go arbitrarily high up the inheritance chain. As an
example I called a grandparent's method on a class and it worked like a charm.

A somewhat similar technique is allowed - although not widely known - in Go.
Obviously since Go does not support inheritance it can't really be useful to
travel up the inheritance chain but you can still apply it to DRY-up your code.

As an example, here is a [POP3 library][popart] I've been writing for a side
project of mine that will most likely never see the light of day. POP3 is a
simple, TCP-based text protocol with just a handful of operations. The whole
conversation with the sever is a call-and-response affair so you can handle it
very simply using the wonderful [`net/textproto`][textproto] library.

The library consists of two main parts - a [`Server`][popart-server] which
handles new connections and a [`session`][popart-session] (non-public struct)
which takes over and handles each connection. Each `session` in turn responds to
a handful of commands, each of which corresponds to a single method inside the
`session` object.

Each line received from the client is parsed to determine which method to run
in response. This is where unbound methods come in. Instead of handling it using
a giant ugly `switch` expression I decided to turn it into a map of
operation (string) to handler (function) external to the `session` itself.
The problem is that handlers are not functions but methods - here
is an example:

{% highlight go %}
func (s *session) handleNOOP(args []string) error {
  return s.respondOK("doing nothing")
}
{% endhighlight %}

Turns out though that this method has a function signature, too - just like
with unbound methods in Python and Ruby:

{% highlight go %}
type operationHandler func(s *session, args []string) error
{% endhighlight %}

And you can address these like so:

{% highlight go %}
operationHandlers = map[string]operationHandler{
  "APOP": (*session).handleAPOP,
  "CAPA": (*session).handleCAPA,
  "DELE": (*session).handleDELE,
  // etc...
}
{% endhighlight %}

Mind the strange syntax here - each of these former methods (and now functions)
has a pointer receiver which is reflected in its name. You can use them as if
you were using normal functions. There is nothing special about them now, except
that they take a `*session` (variable `s` in the example below) as their first
argument:

{% highlight go %}
operationHandlers[command](s, args)
{% endhighlight %}

So as you can see each method in Go can be turned into a function with a
sprinkle of fairy dust. Due to the language's extreme simplicity it's not
terribly magical (which is arguably a _good thing_) but it's still a useful tool
to have in your shed.

If you found this interesting you may want to browse through the source code of
the [popart library source code][popart-source] since there is a few neat tricks
I'm using there that I'd like to post about in the future.

[popart]: http://godoc.org/github.com/slowmail-io/popart
[textproto]: https://golang.org/pkg/net/textproto/
[popart-server]: http://godoc.org/github.com/slowmail-io/popart#Server
[popart-session]: https://github.com/slowmail-io/popart/blob/master/session.go
[popart-source]: https://github.com/slowmail-io/popart/
