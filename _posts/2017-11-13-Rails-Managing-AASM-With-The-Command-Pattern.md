---
layout: post
title: Rails -- Managing AASM with the Command Pattern
tags:
  - Software Engineering
---

How to reign in your AASM code.

[AASM](https://github.com/aasm/aasm) is one of the most powerful Ruby gems for Rails development. We use it extensively at Square Capital for components such as the payment plans and application flows. Normally, we could get away with a fairly standard configuration of AASM because the state machine complexity was relatively simple to understand. Here’s a game of basketball:

```ruby
class BasketballGame < ApplicationRecord
  include AASM

  aasm(column: :status) do
    state :pending, initial: true
    state :playing
    state :timeout

    event :start do
      transitions from: :pending, to: :playing
    end

    event :timeout do
      transitions from: :playing, to: :timeout
    end

    event :resume do
      transitions from: :timeout, to: :playing
    end
  end

  # A method to have automatic state transitions.
  def process_game!
    # Good Practice: Modify non-status fields once and only once before transitioning AASM states.
    update_stats!
    populate_timeout_reasons!

    resume! if may_resume?
    timeout! if may_timeout?
  end
end
```

Things are rarely *this* simple, but you get the picture. Let’s add some more post-transition actions and guards.

```ruby
class BasketballGame < ApplicationRecord
  include AASM

  aasm(column: :status) do
    state :pending, initial: true
    state :playing
    state :timeout

    event :start do
      after { on_start }
      transitions from: :pending, to: :playing, guard: startable?
    end

    event :timeout do
      after { on_timeout }
      transitions from: :playing, to: :timeout, guard: timeoutable?
    end

    event :resume do
      after { on_resume }
      transitions from: :timeout, to: :playing, guard: resumable?
    end
  end

  # A method to have automatic state transitions.
  def process_game!
    # Good Practice: Modify non-status fields once and only once before transitioning AASM states.
    update_stats!
    populate_timeout_reasons!

    resume! if may_resume?
    timeout! if may_timeout?
  end

  private

  def timeoutable?
    timeout_reasons.present?
  end

  def on_timeout
    play_commercial
    put_me_in_coach!
    stop_stat_track!
  end

  def play_commercial
     ...
end
```

We’re going to run out of room here, but at least our code is still readable. It’s easy to see what our possibilities are with automatic and manual state transitions. One could even learn to play my never-ending game of basketball just by reading the `aasm` and `#process_game!` definitions. The callback and guard methods are more difficult to grok without breaking your mouse-wheel.

## The Command Pattern
We could delegate the functionality around the state transitions to their individual classes. This results in a less greppable but more comprehensive overview of what is happening around each transition. Let’s take the functionality outside of the main `BasketBall` model and move them into a lib class, `BasketballGame::Operation::Timeout`.

```ruby
class BasketballGame::Operation::Timeout
  class << self
    def guard(basketball_game)
      basketball_game.errors.clear
      timeout_reasons_must_exist(basketball_game)
      basketball_game.errors.empty?
    end

    private

    def timeout_reasons_must_exist(basketball_game)
      if basketball_game.timeout_reasons.empty?
        basketball_game.errors.add(:timeout_reasons, 'cannot go to timeout: must not be empty')
      end
    end
  end

  attr_reader :basketball_game

  def initialize(basketball_game)
    @basketball_game = basketball_game
  end

  def call
    play_commercial
    put_me_in_coach!
    stop_stat_track!
  end

  private

  def play_commercial; end
  def put_me_in_coach!; end
  def stop_stat_track!; end
end
```

Change your AASM event to invoke the `.guard` and `#call` methods.

```ruby
event :timeout do
  transitions from: :playing,
              to: :timeout,
              guard: lambda { Basketball::Operation::Timeout.guard(self) },
              after: Basketball::Operation::Timeout
end
```

Our operation class is acts as a glorified Proc that is called by AASM, and our main model is back to a manageable size.
