# Routing Secrets & Request Attributes

This array is the *end* result of the route-matching process. Apparently, the router
returns an array with the wildcard values from the route *plus* keys for the
route and controller.

But... it's a bit more interesting than that. A great way to see how, is by
playing with a route in YAML. Open up `config/routes.yaml`. Uncomment the example
route and change the path to `/playing`. Now, on your browser, open *another*
tab and go to https://localhost:8000/playing.

[[[ code('2371b6fbd8') ]]]

That's exactly what we expected: `_route` set to the route name and `_controller`
set to the controller string for that route.

## Route Defaults

But in reality, the `controller` key in a YAML route is just a shortcut. Before
Symfony 4, there *was* no `controller` key. Nope, to define a controller you
added a `defaults` key and put an `_controller` key below *that*.

[[[ code('cf42138d21') ]]]

Move over and refresh now. Woh! We get the *exact* same array! Yep, the
`controller` key is *really* just a shortcut for setting an `_controller` *default*
value on the route.

This is actually an important point, but to see why, let's go a bit further.
First, add a `{id}` wildcard to the end of the path. Then, at your browser, add
`/5` to the end of the URL. And... yep! The array now has an `id` key: no surprise.

[[[ code('d7cb0937e9') ]]]

*Normally*, the purpose of `defaults` on a route are to give a default value for
a *wildcard*. If we say `id: 10`... and then refresh, the array still contains 5
because *that's* what's in the URL. But *thanks* to the default, *now* we can *just*
go to `/playing` and... the id uses the default value `10`.

[[[ code('7afafe30af') ]]]

Cool. But what if we just... *invent* a new key and put it here? Like
`totally_inventing_this_default` set to `true`.

[[[ code('fee0db031b') ]]]

This won't change how the route *matches*, but it *will* change what we get back
in the array. Refresh. The `totally_inventing_this_default` key is now inside
the returned array!

So here's the *full* story of what the route matching process returns: it returns
an `array_merge` of the route defaults and any wildcard values in the route....
plus the `_route` key... just in case that's handy.

With route annotations, it looks a bit different, but it's exactly the same. We can
add a `defaults` key and set `foo` to `bar`. Back in the browser, close the last
tab and refresh the article show page. We suddenly have a `foo` key! On the
route, remove that `defaults` stuff.

[[[ code('1e03784723') ]]]

## Request Attributes

So why is it *so* important to understand exactly *what* the route-matching process
returns? We'll find out the *full* answer soon. But first... back in `RouterListener`,
what does this class *do* with the `$parameters` array?

Remove the `dd()`... and let's follow the logic. It does some logging and... here
it is: `$request->attributes->add($parameters)`. *This* is important.

Let's back up for a second: the `Request` object has several public properties
and *all* of them - except one! - correspond to something on the HTTP request.
For example, `$request->headers` holds the HTTP request headers, `$request->cookies`
holds the cookies, and there are others like `$request->query` to read the query
parameters. The point is: *all* of these refer to real "parts" of an HTTP request.
You could talk to a Java developer about HTTP headers and they would know what
you're referring to.

The *one* exception is `$request->attributes`. This property does *not* correspond
to any *real* part of the HTTP request. If you ask that *same* Java developer:

> Hey! What are the attributes on your request?

They'll think you're nuts. Nope, the Request attributes are something totally
invented by Symfony. The *purpose* of the request attributes is to be a place
where you can store data about the request that's specific to your application.
So, storing the *controller*, for example, is a perfect fit! That's *completely*
a Symfony concept.

Anyways, the array of `$parameters` from the router is added to the
`$request->attributes()`. What does that... do? Absolutely nothing. Soon,
something *else* will *use* this data, but at this moment, this is *just* data
sitting on the request.

It also sets another attribute `_route_params`... but that's not really important.

## After kernel.request... we have Request Attributes!

Ok! `RouterListener` done! Close that class, high-five your cat - and go back
to `HttpKernel`. As we saw, there are a lot of listeners to the `kernel.request`
event, but by *far*, the most important one is `RouterListener`. So what *changed*
in our system before and after this `dispatch()` line? Basically... *just* the
request attributes.

In fact, let's see this. Above dispatch, `dump($request->attributes->all()`. Then
copy that... dump after, and `die`. Refresh the article show page. Yep! Before
we dispatch the event, the attributes are empty. After? We have `_route`,
`_controller`, `slug` and hey! A few *other* things were added by *other*
listeners related to security. That's not important for us - but still, interesting!

[[[ code('49b38fd4d4') ]]]

Remove all that debug code.

## Seeing the Dumped Route

Before we find out *how* the request attributes are used, I want to show you
something kinda cool. We're going to look at a cache file: `var/cache/dev`...
and then `url_matching_routes.php`.

This file is automatically generated by Symfony and is the *end-result* of
*all* of the routes in our application. This file is *insane*. After reading
our routes, Symfony generates a huge list of regular expressions and which route
should match which part, and dumps them to this file. This is used by the
route-matching process so that it's *blazingly* fast. It's... pretty amazing.

Anyways, next! Let's see the significance of those Request attributes by
continuing to go through the `handleRaw()` method.
