---
layout: post
title: End-to-end testing is essential
date: '2012-07-18'
categories: articles
author: Gregory Brown
permalink: articles/end-to-end-testing-is-essential
summary: Build acceptance tests that reflect application behavior, rather than implementation
  details.
issue_number: 4.12.1
---

> **NOTE:** This is one of [four lessons
> learned](http://practicingruby.com/articles/65) from my 90 day [self-study on
> test-driven development](http://practicingruby.com/articles/28). 
> If this topic interests you, be sure to check out the other lessons!

Perhaps the most significant thing I have noticed about my own TDD habits 
is that I frequently defer end-to-end testing or skip it entirely, and that 
always comes at a huge cost. Now that I have had a chance to watch 
myself get caught in that trap several times, I have a better understanding
of what triggers it.

Most of the time when I work on application development, I start out by 
attempting to treat its delivery mechanism as an implementation detail. 
Thinking this way makes me feel that testing code through the UI 
isn't especially important, provided that I test-drive my domain objects 
and keep their surface as narrow as possible. My first iteration on the
Blind game provides a good example of how I tend to apply this strategy.

My first complete feature in the game was a simple proof of concept: 
I dropped the player into the center of a map, and then allowed them to
move around using the WASD keys on their keyboard. When the player 
reached the edge of the map, the game would play a beeping sound
and then terminate itself. You can check out the [full source for 
this feature](https://github.com/elm-city-craftworks/blind/compare/1f6a...4345)
to see its implementation details, but the important thing to note
is that its delivery mechanism was tiny and almost completely logic-free:

```ruby
Ray.game "Blind" do
  register { add_hook :quit, method(:exit!) }

  scene :main do
    self.frames_per_second = 10

    @game = Blind::Game.new
    @game.on_event(:out_of_bounds) do
      beep = sound("#{File.dirname(__FILE__)}/../data/beep.wav")
      beep.play
      sleep beep.duration

      exit!
    end

    always do
      puts @game.player_position

      @game.move_player(0,-1) if holding?(:w)
      @game.move_player(0,1) if holding?(:s)
      @game.move_player(-1,0) if holding?(:a)
      @game.move_player(1,0) if holding?(:d)
    end
  end

  scenes << :main
end
```

Based on this single code example, it is easy to make the case that end-to-end
testing can be deferred until later, or that perhaps it is not needed at all.
Thinking this way is very tempting, because it frees you from having to think
about how to dig down into the delivery mechanism and run your tests through it.
Already burdened by the idea of writing more tests than I usually do, I was
quick to take that bargain and felt like it was a reasonable tradeoff at the
time.

I couldn't have been more wrong. I encountered my first trivial UI bug 
within 24 hours of shipping the first feature. Several dozen patches 
later when I had a playable game, I had already sunk several hours into 
finding and fixing UI defects that I discovered through manual play testing.
The wheels finally came off the wagon when I realized that I could not
even safely rename methods without playing through the entire game and
triggering each of its edge cases. The following example shows one
of the many "oops" changesets that the projects accumulated in a very short
period of time:

```diff
sounds[:siren].volume = 
-  ((world.distance(world.center_position) - min) / max.to_f) * 100
+  ((world.distance(world.center_point) - min) / max.to_f) * 100
```

By the time I had finally felt the pain of not having any tests running from
end-to-end, the delivery mechanism was no longer a trivial script that could
be scribbled on the back of a napkin. Over the period of a week or so, it had
grown into a [couple hundred lines of code](https://github.com/elm-city-craftworks/blind/tree/776f3462c2244634ccddc22a5473916d6439872c/lib/blind/ui) 
spread across several significant features. The surface of
the domain model also needed to expand to support these new
features, and so the critical path through the system became difficult to 
keep in mind while working on the codebase. This made it much harder
to introduce a change without accidentally breaking something. 

Fed up with chasing down trivial bugs and spending so much time on manual
testing, I finally decided that I needed to implement a player simulator 
which would allow me write tests similar to the one shown below:

```ruby
it "should result in a loss on entering deep space" do
  world  = Blind::Worlds.original(0)
  levels = [Blind::Level.new(world, "test")]

  game  = Blind::UI::GamePresenter.new(levels)

  sim   = Blind::UI::Simulator.new(game)

  sim.move(500, 500)
  
  sim.status.must_equal "You drifted off into deep space! YOU LOSE!"
end
```

As predicted, the `Blind::UI::Simulator` object was [not especially easy to
implement](https://github.com/elm-city-craftworks/blind/blob/2fa2d75216077bdafa556be3c560b3f7c205e672/lib/blind/ui/simulator.rb). 
To get it to work, I had to experiment with several undocumented features in the Ray
framework and cobble them together through a messy trial and error process. This
reminded me of previous projects I had worked on where I had to do the same
thing to introduce end-to-end tests in Rails applications, and is quite possibly
one of my least favorite programming tasks; all this work just feels so
tangential to the task at hand.

Still, it is hard to argue with results. After introducing this little simulator
object, the number of trivial errors I introduced into the system rapidly
declined, even though I was still actively changing things. Occasionally, I'd
make a change which broke the simulator in weird and confusing ways, but all the
time spent working on those issues was less than the total time I spent chasing
down dumb mistakes before making this change. 

As I continued on with my study, I experienced similar situations with both a 
Sinatra application and a command line application, and that is when I realized
that you simply can't get away from paying this tax one way or another. If
nothing else, working on acceptance tests first helps balance out the illusion
of progress in the early stages of a project, and makes it easier to sustain
an even pace of development over time.

At the end of my study, I read the first few chapters of [Growing Object
Oriented Software, Guided by Tests](http://www.growing-object-oriented-software.com/), 
and it gave similar advice to what I had found out the hard way. The authors
presented a somewhat more radical idea about how to build application runners, 
suggesting that they should completely hide the implementation details of the 
underlying application and its delivery mechanism. To try out these ideas, 
I built a small [tic-tac-toe game](https://github.com/elm-city-craftworks/ruby-examples/tree/master/tic_tac_toe) 
using Ray, writing my first end-to-end test before writing any other code: 

```ruby
describe "A tic tac toe game" do
  it "alternates between X and O" do
    run_game do |runner|
      runner.message.must_equal("It's your turn, X")
      runner.move(5)
      runner.message.must_equal("It's your turn, O")
      runner.move(3)
      runner.message.must_equal("It's your turn, X")
    end
  end

  def run_game(&block)
    GameRunner.run(&block)
  end
end
```

Because this test does all of its work through the
[GameRunner
object](https://github.com/elm-city-craftworks/ruby-examples/blob/master/tic_tac_toe/test/helpers/game_runner.rb),
it is both easier to read and more maintainable than the tests that I built for
Blind. Furthermore, I feel like it is much easier write test-first code this
way, as it doesn't require as many decisions to be made up front.

I've been talking about a rather weird domain throughout this article (game
programming in Ray), but I could easily imagine how I might apply what I've
learned to a traditional Rails application. For example, if I were to build a
blog and wanted to write my first test for it, I might start with something like
this:

```ruby
describe "A post" do
  let(:blogger) { SimulatedBlogger.new }

  it "can be created by a logged in blogger" do
    blogger.log_in("user", "password")
    blogger.create_post("Hello World")
  end

  it "cannot be created by a blogger that has not logged in" do
    assert_raises(SimulatedBlogger::AccessDeniedError) do
      blogger.create_post("Hello World")
    end
  end
end
```

I would then move on to implement the `SimulatedBlogger` using something like
capybara or some other web automation tool. On the surface, this at least 
seems like a good idea; in practice it may be more trouble than it's worth for a number 
of reasons.

Since I'm still relatively new to end-to-end testing in general, I am definitely
curious to hear what you think of these ideas. This article summarizes what
I learned from my experiences during my study, but I am not yet confident in my 
own applications of these techniques. If you have an interesting story to share, 
please do so!
