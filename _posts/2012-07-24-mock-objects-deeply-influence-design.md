---
layout: post
title: Mock objects deeply influence design
date: '2012-07-24'
categories: articles
author: Gregory Brown
permalink: articles/mock-objects-deeply-influence-design
summary: Learn the rules that you need to follow in order to use mock objects as an
  effective design tool.
issue_number: 4.12.3
---

> **NOTE:** This is one of [four lessons
> learned](http://practicingruby.com/articles/65) from my 90 day [self-study on
> test-driven development](http://practicingruby.com/articles/28). 
> If this topic interests you, be sure to check out the other lessons!

Before this study, I knew that I [rarely used mock objects](http://practicingruby.com/articles/49) 
in my tests, but I didn't clearly understand why that was the case. When asked to explain my 
preferences, I typically would offer some vague argument about keeping things
simple, and then go on to complain about test brittleness. Because I knew many
other people who shared the same view, I assumed my line of reasoning was 
mostly coherent. This left me with no desire to dig any deeper than what my own 
experiences had taught me.

After years of somewhat blissful ignorance, I finally started to second guess
myself after watching Greg Moeck's excellent talk at RubyConf 2011, which was
aptly named [Why You Don't Get Mock Objects](http://www.confreaks.com/videos/659-rubyconf2011-why-you-don-t-get-mock-objects). 
This talk pointed out that the reason why most Rubyists tend to dislike mock
objects is because they try to shoehorn them into existing workflows rather 
than adopting the form of TDD that mocks are meant to promote. I remember being
easily convinced by this talk when I first watched it, but because old habits
die hard, I never ended up changing my way of doing things.

Throughout the entire 90 day period of my study, I found myself [using mock
objects only
once](https://github.com/elm-city-craftworks/broken_record/blob/5c9287e0c6d8211c4a91aee43b26181dfbcc1992/test/record_test.rb), 
even though I had thought about using them in many places. Towards the end, I realized that I
still didn't quite understand how mocks were meant to be used, and so I
decided to study them properly. This inevitably lead me to the excellent [Mock
Roles, Not Objects](http://www.jmock.org/oopsla2004.pdf) paper, which was written in 
2004 by the developers who had pioneered the concept of mock-based testing. In
addition to being a solid introduction to the topic in general, the paper lays
out a number of practical guidelines for avoiding the common problems that can
arise from using mocks incorrectly. In particular, the authors proposed the
following rules:

* Only mock types you own.
* Don't use getters.
* Be explicit about what should not happen.
* Specify as little as possible in a test.
* Don't use mocks to test boundary objects.
* Don't add behavior to mocks.
* Only mock your immediate neighbors.
* Don't create too many mocks.
* Inject all dependencies.

By programming in this style, the promise is that the benefits of mock objects
will be maximized and their drawbacks minimized. The interesting thing is that
while several of these heuristics are meant to improve the testability of code,
nearly as many have a direct influence on software design in general. Taken
together, the following four points strongly favor [responsibility-centric
design](http://practicingruby.com/articles/64):

* Don't use getters.
* Only mock your immediate neighbors.
* Don't create too many mocks.
* Inject all dependencies.

These guidelines will almost certainly lead to code that is more testable, 
and should also lead to code that is easier to change. If you think about
these heuristics a little bit, you'll find they conveniently map onto
the following software design principles:

* [Tell, don't ask](http://robots.thoughtbot.com/post/27572137956/tell-dont-ask)
* [The law of Demeter](http://en.wikipedia.org/wiki/Law_of_Demeter)
* [Single responsibility](http://en.wikipedia.org/wiki/Single_responsibility_principle)
* [Dependency inversion](http://en.wikipedia.org/wiki/Dependency_inversion_principle)

Testing a codebase via mock objects is easy when these design principles are
followed, and challenging when they are not. In that sense, mock objects can be
used as a smoke test for the overall design of a project, which is useful in its
own right. However, most mockists claim that the technique actually inspires 
better design, rather than simply helping you find areas in your code that
suffer from bad design. This is a much broader statement, and isn't nearly as
obvious to those who have not had this experience themselves.

Because I used mock objects so infrequently during my study, I am unable to tell
you whether or not they can actually help improve software design. However, now
that I have a clearer sense of what my own workflow is like, I understand why I
have had so few opportunities to make good use of mock objects. It all boils
down to the fact that I don't practice disciplined outside-in design.

The way I tend to approach design is to choose a very small vertical slice of
functionality and develop an imaginary example of how I expect that feature to
work. This technique is consistent with the outside-in way of doing things, 
but my next steps bring me in a completely different direction. Rather than
starting with my interface and then using mock objects to allow me to discover
collaborators iteratively until I reach the lowest-level objects in my system, 
I build things bottom up instead.

Taking a look back at the projects I worked on during this study, I was able to
see this trend in action. For example, in BrokenRecord, my first test
was for a struct-like object that would be used for storing field
data:

```ruby
describe BrokenRecord::Row do
  it "must create readers for all attributes" do
    row = BrokenRecord::Row.new(:a => 1, :b => 2)

    row.a.must_equal(1)
    row.b.must_equal(2)
  end
end
```

Similarly, when I was working on the Blind game, my first test was for a `Map` object
that allowed you to place named objects at specific coordinates:

```ruby
describe Blind::Map do
  it "must be able to store elements at a position" do
    map = Blind::Map.new
    map.add_object("player", 10, 25)

    pos = map.locate("player")
    [pos.x, pos.y].must_equal([10,25])
  end
end
```

Even though each of these objects were designed with a single external feature
in mind, they are clearly boundary objects; concrete implementation code with 
no collaborators within the system. As I built on top of them, I found no
need for mocks, because using these objects directly was easy enough to do. The
benefit of building things this way is that you can think in terms of concrete
objects at all times, but that is also the drawback: you can't use mock objects
to discover the protocols of your collaborators if those details have already
been locked down. I don't know enough about mock-based TDD to know
whether this is a trade worth making, but this does explain to me why I've
failed to experience some of its benefits.

After I realized that I haven't been working in a way that would support the
effective use of mock objects, I took an interest in figuring out what kind of
workflow mockists tend to follow. Digging back to one of my [favorite articles on
mock objects](http://martinfowler.com/articles/mocksArentStubs.html), I found that 
this is what Martin Fowler had to say:

> Mock objects came out of the XP community, and one of the principal features of XP is its emphasis on Test Driven Development - where a system design is evolved through iteration driven by writing tests.

> Thus it's no surprise that the mockists particularly talk about the effect of mockist testing on a design. In particular they advocate a style called need-driven development. With this style you begin developing a story by writing your first test for the outside of your system, making some interface object your SUT. By thinking through the expectations upon the collaborators, you explore the interaction between the SUT and its neighbors - effectively designing the outbound interface of the SUT.

> Once you have your first test running, the expectations on the mocks provide a specification for the next step and a starting point for the tests. You turn each expectation into a test on a collaborator and repeat the process working your way into the system one SUT at a time. This style is also referred to as outside-in, which is a very descriptive name for it. It works well with layered systems. You first start by programming the UI using mock layers underneath. Then you write tests for the lower layer, gradually stepping through the system one layer at a time. This is a very structured and controlled approach, one that many people believe is helpful to guide newcomers to OO and TDD.

Based on what I learned about mock objects, this style of development does
appear to be a natural way of developing responsibility-centric code that abides
by all the guidelines laid out in the [Mock Roles, Not
Objects](http://www.jmock.org/oopsla2004.pdf) paper. While it sounds intriguing
to me and worth trying out, I doubt that I am smart enough to apply this
style of development effectively. The reason I tend to use a divide-and-conquer, 
think-in-concrete-objects strategy in my projects is that I don't have much
faith in my own abilities to understand the current and future relations 
between my objects. In other words, the disciplined outside-in approach seems 
to require more design confidence than what I typically am able to muster up. 

To make matters worse, I have not yet come across an example that clearly shows how this
technique can be applied throughout an entire project. I think that in addition
to my own experimentation, I'll need to see something like that for these ideas
to finally click. If you know of a source of good large-scale examples of these
techniques, please let me know!

To sum up the overall point of this lesson: mock objects facilitate
a particular design style, and if you're not using that approach in
your projects, you probably will not experience their benefits. I'd love to hear
your thoughts on that conclusion, whether or not you agree with it; I clearly
have a lot more to learn in this area.
