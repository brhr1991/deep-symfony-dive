# Argument Resolver

Coming soon...

Inside of `HttpKernel`. We now have the controller, but we still need to figure out
what arguments to pass when we call the controller. Now to make this a little bit
more a little bit better for now, one of the things we know that we can do with the
controller is that we can type into an entity like `Article` and in something will
automatically query for that `Article` by its slug. I'm going to temporarily remove
that when I put it back later and explain how it works. So in the places where they
slept, arguments also going to add an `ArticleRepository $articleRepository`
argument. So it, it's weird for it and I'll say 
`$article = $articleRepository->findOneBy(['slug' => $slug]);`
And then because we're handling this ourselves, I mean to say if not `$article`,

don't throw `$this->createNotFoundException()`. So this is basically, this is
equivalent to what we just had a second ago. We'll just make for a better example. By
the way, if we know that this line here is going to turn into a 404 if you
hold command or control and click into this, you'll find that this throws an
exception, a `NotFoundHttpException`, which for some reason is going to result in
the page being a 404 response code. By the end of this tutorial, we'll know
exactly why this exception is mapped to a 404 anyways, let's go back to 
`HttpKernel`. Now that we've figured out what the controller is, the next thing that
happens in the Symfony is it dispatches another event. This one's called `ControllerEvent`
so keeping score here we have so far dispatched an event, found the
controller, and dispatched another event, and that's it on this one, nothing really
important happens from the perspective of the framework. If I refresh the article
show page and then click to open the profiler, I'll go back to the events and down
here we can say `kernel.controller`. So there's six events here, but nothing
that's very, very important. A couple of them come from `FrameworkExtraBundle`, that
power. Some of it's magic. We'll talk about those later.

One of the things that a listener can can do, can do on this event, those not very
common is replace the controller. You see down here it says 
`$controller = $event->getController()`. Cause if you hold command and open that 
`ControllerEvent`, it actually has a `setController()`. So it's possible for a listener 
on this event to completely replace the controller for some reason. All right, 
so after this whole point, this is the missing piece here. We need to know what 
arguments we're going to pass to our controller. It does that by calling something 
called the argument resolver, and it's pretty cool. It calls `getArguments()`, it passes 
the `$request`, it passes the `$controller` executable, and it says, Hey, give me all the 
arguments I should pass to this function. So let's open up this argument resolver. 
I'll hit shift shift and it's literally going to be called argument is all 
`ArgumentResolver.php`.

And I'll open that and find the `getArguments()` method. Okay, so interesting. First
thing it does is if for each is over `$this->argumentMetadataFactory->createArgumentMetadata()`
meta-data as a `$metadata`, but that is actually doing is it's looping over all of the
arguments to your controller. So in this case it would have, it would loop over three
times for our show page and then it's looping over something else for each argument.
It's looping over something else called the `argumentValueResolvers`. So let's just
see what's going on here. Inside the first loop, I'm going to `dd()` this `$metadata`. So
DDB thing that we're looping over, let's pick over and let's refresh the article
page. Okay, so apparently it's this `ArgumentMetadata` object, which contains the
name of the argument `slug` because that is the name of the first argument. And then
other information like the `type`, which in this case is `null`, but for the second
argument it would be `SlackClient`.

And then other information like is very attic has `defaultValue` `isNullable`, other
things. So it's kind of cool. It's just a bunch of metadata about that argument. And
in this case, this will loop over three times. It would get a metadata object for
each of our three arguments, so that's the first thing to understand. Now let's
replace this `dd()`. Let's figure out what these argument value resolvers are. So this is
actually an iterator. You look up here, it's an `Iterable`, which is basically a fancy
array. If I'm going to call it `iterator_to_array($this->argumentValueResolvers))`.

So now we go over and refresh, we get seven of them. They're all inside of this
`TraceableValueResolver`, but if you look inside, let me open a couple of these.
You'll find that we have something called a `RequestAttributeValueResolver`
`RequestValueResolver`, a `SessionValueResolver`. These are a bunch of different
classes that help figure out which arguments to pass to our controller. Another way
to see this full list by the way, is to fund your terminal rugby 

```terminal
php bin/console debug:container --tag=controller.argument_value_resolver
```

because the way you add an argument resolver, if you want to do a custom one, we will
do that later is by creating a service and giving it this tag. So this dumps us all,
uh, all of our um, RJ resolvers and they're all decorated in this 
`TraceableValueResolver`

But this gives you a better idea. We have something called the  `RequestAttributeValueResolver` 
`RequestValueResolver`, `SessionValueResolver` over `UserValueResolver`, a whole bunch of 
different things. We're going to walk through some of the most important ones these
next. But before we do that, let's just go back and look how they work. So on high
level, we're looping over each argument here and then we move over every single
argument or resolver and we say we may and we call a `support()` method. So one by one we
call these argument resolvers and we say, Hey, for this argument, you know, for the,
for an argument called a `slug` or an argument called `slack`, Slack with a `SlackClient`
type int, do you support figuring out what value should be past that argument? And if
it returns, if an argument resolver returns false, then it just continues onto the
next one. If it returns true, then it calls `$resolver->resolve()`. And it
basically says, okay, give me the value that she passed to that argument. So at the
end, so hopefully by the end of looping through all of these argument resolvers, one
of them has figured out what value to pass for it. So next, let's look through the
most important argument resolvers and figure out what, what all the possible values
are to the arguments to our controller.
