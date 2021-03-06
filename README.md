resque-retry
============

A [Resque][rq] plugin. Requires Resque 1.8.0 & [resque-scheduler][rqs].

resque-retry provides retry, delay and exponential backoff support for
resque jobs.

  * Redis backed retry count/limit.
  * Retry on all or specific exceptions.
  * Exponential backoff (varying the delay between retrys).
  * Multiple failure backend with retry suppression & resque-web tab.
  * Small & Extendable - plenty of places to override retry logic/settings.

Install & Quick Start
---------------------

To install:

    $ gem install resque-retry

You'll want add this to your `Rakefile`:

    require 'resque/tasks'
    require 'resque_scheduler/tasks'

The delay between retry attempts is provided by [resque-scheduler][rqs].
You'll want to run the scheduler process, otherwise delayed retry attempts
will never perform:

    $ rake resque:scheduler

Use the plugin:

    require 'resque-retry'

    class ExampleRetryJob
      extend Resque::Plugins::Retry
      @queue = :example_queue

      @retry_limit = 3
      @retry_delay = 60

      def self.perform(*args)
        # your magic/heavy lifting goes here.
      end
    end

Then start up a resque worker as normal:

    $ QUEUE=* rake resque:work

Now if you ExampleRetryJob fails, it will be retried 3 times, with a 60 second
delay between attempts.

For more explanation and examples, please see the remaining documentation.

Failure Backend & Resque Web Additions
--------------------------------------

Lets say your using the Redis failure backend of resque (the default).
Every time a job fails, the failure queue is populated with the job and
exception details.

Normally this is useful, but if your jobs retry... it can cause a bit of a mess.

For example: given a job that retried 4 times before completing successful.
You'll have a lot of failures for the same job and you wont be sure if it
actually completed successfully just by just using the resque-web interface.

### Failure Backend

`MultipleWithRetrySuppression` is a multiple failure backend, with retry suppression.

Here's an example, using the Redis failure backend:

    require 'resque-retry'
    require 'resque/failure/redis'

    # require your jobs & application code.

    Resque::Failure::MultipleWithRetrySuppression.classes = [Resque::Failure::Redis]
    Resque::Failure.backend = Resque::Failure::MultipleWithRetrySuppression

If a job fails, but **can and will** retry, the failure details wont be
logged in the Redis failed queue *(visible via resque-web)*.

If the job fails, but **can't or won't** retry, the failure will be logged in
the Redis failed queue, like a normal failure *(without retry)* would.

### Resque Web Additions

If your using the `MultipleWithRetrySuppression` failure backend, you should
also checkout the resque-web additions!

The new Retry tab displays delayed jobs with retry information; the number of
attempts and the exception details from the last failure.

Make sure you include this in your `config.ru` or similar file:

    require 'resque-retry'
    require 'resque-retry/server'

    # require your jobs & application code.

    run Resque::Server.new

Retry Options & Logic
---------------------

Please take a look at the yardoc/code for more details on methods you may
wish to override.

Customisation is pretty easy, the below examples should give you
some ideas =), adapt for your own usage and feel free to pick and mix!

### Retry Defaults

Retry the job **once** on failure, with zero delay.

    require 'require-retry'

    class DeliverWebHook
      extend Resque::Plugins::Retry
      @queue = :web_hooks

      def self.perform(url, hook_id, hmac_key)
        heavy_lifting
      end
    end

When a job runs, the number of retry attempts is checked and incremented
in Redis. If your job fails, the number of retry attempts is used to
determine if we can requeue the job for another go.

### Custom Retry

    class DeliverWebHook
      extend Resque::Plugins::Retry
      @queue = :web_hooks

      @retry_limit = 10
      @retry_delay = 120

      def self.perform(url, hook_id, hmac_key)
        heavy_lifting
      end
    end

The above modification will allow your job to retry upto 10 times, with
a delay of 120 seconds, or 2 minutes between retry attempts.

Alternatively you could override the `retry_delay` method to do something
more special.

### Exponential Backoff

Use this if you wish to vary the delay between retry attempts:

    class DeliverSMS
      extend Resque::Plugins::ExponentialBackoff
      @queue = :mt_messages

      def self.perform(mt_id, mobile_number, message)
        heavy_lifting
      end
    end

**Default Settings**

    key: m = minutes, h = hours

                  no delay, 1m, 10m,   1h,    3h,    6h
    @backoff_strategy = [0, 60, 600, 3600, 10800, 21600]

The first delay will be 0 seconds, the 2nd will be 60 seconds, etc...
Again, tweak to your own needs.

The number if retrys is equal to the size of the `backoff_strategy`
array, unless you set `retry_limit` yourself.

### Retry Specific Exceptions

The default will allow a retry for any type of exception. You may change
it so only specific exceptions are retried using `retry_exceptions`:

    class DeliverSMS
      extend Resque::Plugins::Retry
      @queue = :mt_messages

      @retry_exceptions = [NetworkError]

      def self.perform(mt_id, mobile_number, message)
        heavy_lifting
      end
    end

The above modification will **only** retry if a `NetworkError` (or subclass)
exception is thrown.

### Custom Retry Criteria Check Callbacks

You may define custom retry criteria callbacks:

    class TurkWorker
      extend Resque::Plugins::Retry
      @queue = :turk_job_processor

      @retry_exceptions = [NetworkError]

      retry_criteria_check do |exception, *args|
        if exception.message =~ /InvalidJobId/
          false # don't retry if we got passed a invalid job id.
        else
          true  # its okay for a retry attempt to continue.
        end
      end

      def self.perform(job_id)
        heavy_lifting
      end
    end

Similar to the previous example, this job will retry if either a
`NetworkError` (or subclass) exception is thrown **or** any of the callbacks
return true.

Use `@retry_exceptions = []` to **only** use callbacks, to determine if the
job should retry.

### Retry Arguments

You may override `args_for_retry`, which is passed the current
job arguments, to modify the arguments for the next retry attempt.

    class DeliverViaSMSC
      extend Resque::Plugins::Retry
      @queue = :mt_smsc_messages

      # retry using the emergency SMSC.
      def self.args_for_retry(smsc_id, mt_message)
        [999, mt_message]
      end

      self.perform(smsc_id, mt_message)
        heavy_lifting
      end
    end

### Job Identifier/Key

The retry attempt is incremented and stored in a Redis key. The key is
built using the `identifier`. If you have a lot of arguments or really long
ones, you should consider overriding `identifier` to define a more precise
or loose custom identifier.

The default identifier is just your job arguments joined with a dash `-`.

By default the key uses this format: 
`resque-retry:<job class name>:<identifier>`.

Or you can define the entire key by overriding `redis_retry_key`.

    class DeliverSMS
      extend Resque::Plugins::Retry
      @queue = :mt_messages

      def self.identifier(mt_id, mobile_number, message)
        "#{mobile_number}:#{mt_id}"
      end

      self.perform(mt_id, mobile_number, message)
        heavy_lifting
      end
    end

Contributing/Pull Requests
--------------------------

  * Yes please!
  * Fork the project.
  * Make your feature addition or bug fix.
  * Add tests for it.
  * Commit.
  * Send me a pull request. Bonus points for topic branches.
  * If you edit the gemspec/version etc, do it in another commit please.

[rq]: http://github.com/defunkt/resque
[rqs]: http://github.com/bvandenbos/resque-scheduler
