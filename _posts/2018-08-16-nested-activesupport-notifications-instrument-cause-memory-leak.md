---
layout: post
title:  "Memorandum: Nested ActiveSupport::Notifications.instrument may cause memory leak"
date:   2018-08-16 20:00:00 +09:00
categories:
- Ruby
---


```ruby
source 'https://rubygems.org'

gem 'activesupport', '5.2.1'
gem 'activejob', '5.2.1'
```


I learned nested `.instrument` cause memory leak
----

Look this.

```ruby
require 'active_support'
require 'active_support/subscriber'

class Subscriber < ActiveSupport::Subscriber
  def event(event)
  end
end
Subscriber.attach_to :sample

puts "### Not nested instrument\n\n"
puts `ps u #{Process.pid}`
1_000_001.times { ActiveSupport::Notifications.instrument('event.sample') }
puts `ps --no-headers u #{Process.pid}`
puts ''

puts "### Nested instrument\n\n"
puts `ps u #{Process.pid}`
ActiveSupport::Notifications.instrument('event.sample') do
  1_000_000.times { ActiveSupport::Notifications.instrument('event.sample') }
end
puts `ps --no-headers u #{Process.pid}`
puts ''

__END__

ruby 2.5.1p57 (2018-03-29 revision 63029) [x86_64-linux-musl]

### Not nested instrument

USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root        72 33.0  0.0  33680 16800 pts/0    Sl+  11:03   0:00 ruby 01_nested_instrument.rb
root        72 94.3  0.0  33680 16996 pts/0    Sl+  11:03   0:05 ruby 01_nested_instrument.rb

### Nested instrument

USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root        72 94.3  0.0  33680 16996 pts/0    Sl+  11:03   0:05 ruby 01_nested_instrument.rb
root        72 98.5  3.0 853040 747976 pts/0   Sl+  11:03   0:13 ruby 01_nested_instrument.rb

```


Why nested `.instrument` cause memory leak?
----

Because, the parent subscriber stack child events.

```ruby
require 'active_support'
require 'active_support/subscriber'

class Subscriber < ActiveSupport::Subscriber
  def event(event)
  end
end
Subscriber.attach_to :sample

subscriber = ActiveSupport::Subscriber.subscribers.last

puts "### Not nested instrument\n\n"
10.times do
  ActiveSupport::Notifications.instrument('event.sample')
end
pp subscriber.send(:event_stack).last&.children&.map(&:class)
puts ''

puts "### Nested instrument\n\n"
ActiveSupport::Notifications.instrument('event.sample') do
  10.times do
    ActiveSupport::Notifications.instrument('event.sample')
  end
  pp subscriber.send(:event_stack).last&.children&.map(&:class)
end
puts ''

__END__

ruby 2.5.1p57 (2018-03-29 revision 63029) [x86_64-linux-musl]

### Not nested instrument

nil

### Nested instrument

[ActiveSupport::Notifications::Event,
 ActiveSupport::Notifications::Event,
 ActiveSupport::Notifications::Event,
 ActiveSupport::Notifications::Event,
 ActiveSupport::Notifications::Event,
 ActiveSupport::Notifications::Event,
 ActiveSupport::Notifications::Event,
 ActiveSupport::Notifications::Event,
 ActiveSupport::Notifications::Event,
 ActiveSupport::Notifications::Event]

```


Should we warry about this behavior?
----

I don't know. If you call many `.instrument` in ActiveJob's `#perform` like this:

```ruby
require 'active_support'
require 'active_support/subscriber'
require 'active_job'

class Job < ActiveJob::Base
  def perform(n, subscriber)
    n.times do
      ActiveSupport::Notifications.instrument('event.active_job')
    end
    pp subscriber.send(:event_stack).last&.children&.map(&:class)
  end
end

class Subscriber < ActiveSupport::Subscriber
  def perform(event)
  end

  def event(event)
  end
end
Subscriber.attach_to :active_job

Job.perform_now(10, ActiveSupport::Subscriber.subscribers.last)

__END__

ruby 2.5.1p57 (2018-03-29 revision 63029) [x86_64-linux-musl]

[ActiveJob] [Job] [b1ad34a0-1084-403c-8f16-21fac79465a3] Performing Job (Job ID: b1ad34a0-1084-403c-8f16-21fac79465a3) from Async(default) with arguments: 10, #<Subscriber:0x000055fc5d1603c8 @queue_key="Subscriber-47271190921700", @patterns=["perform.active_job", "event.active_job"]>
[ActiveSupport::Notifications::Event,
 ActiveSupport::Notifications::Event,
 ActiveSupport::Notifications::Event,
 ActiveSupport::Notifications::Event,
 ActiveSupport::Notifications::Event,
 ActiveSupport::Notifications::Event,
 ActiveSupport::Notifications::Event,
 ActiveSupport::Notifications::Event,
 ActiveSupport::Notifications::Event,
 ActiveSupport::Notifications::Event]
[ActiveJob] [Job] [b1ad34a0-1084-403c-8f16-21fac79465a3] Performed Job (Job ID: b1ad34a0-1084-403c-8f16-21fac79465a3) from Async(default) in 4.35ms
```

The ActiveJob make parent-child relationship.

```ruby
# https://github.com/rails/rails/blob/fc5dd0b85189811062c85520fd70de8389b55aeb/activejob/lib/active_job/logging.rb#L21-L29
      around_perform do |job, block, _|
        tag_logger(job.class.name, job.job_id) do
          payload = { adapter: job.class.queue_adapter, job: job }
          ActiveSupport::Notifications.instrument("perform_start.active_job", payload.dup)
          ActiveSupport::Notifications.instrument("perform.active_job", payload) do
            block.call
          end
        end
      end
```