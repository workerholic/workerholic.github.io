## Introduction
Why would you need to use a Background Job Processor (BJP)?

Let's look at a simple example of a user, identified as the `Client`, registering on a web site `awesome-website.com`. When users register on `awesome-website.com` they receive an email in order for them to verify their email address. What does this mean on the Web Server side?
In the case of `awesome-website.com`, sending an email requires to send an HTTP request to an Email Service Provider (ESP) and wait for a response, depending on the latency of the ESP and the geographical distance between the Web Server and the ESP's server this could take from 2ms to 500ms or more. Let's assume the worst and consider each trip over the wire to the ESP takes 500ms. This means that if the Web Server's usual response time for this kind of HTTP request is 150ms then it will take 650ms total to send a response back to the client.

![Request-Response](/images/popular_features_example_1.png){:width="400"}

650 ms is not a good response time. How can we do better? Does the user really need to receive a response a be able to interact with the rest of the website only once the email has been correctly sent? The answer is no, which means that we could send the email asynchronously and, in the meantime, let the user interact with the website, maybe even send another HTTP request to view a different page. This is where a **Background Job Processor** comes in. Its sole purpose is to be delegated some jobs/work so it can be performed asynchronously and unburden the web server from *blocking* work.

![Example BJP](/images/popular_features_example_2.png){:width="500"}

## Background Job Feature Overview

Before we get into how to build a Background Job Processor, let's take a quick run at the features a BJP needs to have, based on research done on other popular BJPs:

##### Core Features:
* **asynchrony**: a BJP should handle a task in a different process than the one used for the main application, so that the main application is free to continue handling more requests
* **reliability**: a BJP should be able to handle errors gracefully and be resilient in case of crashing
* **efficiency:** a BJP should perform the jobs that it is handed in a timely manner so that it does not get a constantly increasing backlog of jobs
* **scalability**: a BJP should run fine and scale in the context of a distributed system
* **reporting**: a BJP should track statistics about jobs, errors and other information about background jobs for better decision making

##### Additional Features:
* **configurability**: a BJP should be configurable in order to allow developers to tweak options so that the BJP can best fits their own application's needs
* **ease of use**: a BJP should be simple to use out-of-the-box and integrate with rails in the context of building it in Ruby

## Introducing Workerholic: Overall Architecture

![Workerholic Overvall Architecture](/images/workerholic_overall_architecture.png)

Above is a diagram of the overall architecture of our BJP, Workerholic.
- On the left is a stack of instances of a web application, each including Workerholic. Jobs are defined and enqueued in these web application instances.
- In order for the jobs to be enqueued, they are first serialized and then stored in Redis into Lists, serving as `Job Queues`.
- On the right, Workerholic workers poll from the job queues and, if there are any jobs to be done, process the jobs using the `Job Processor` component of Workerholic.
- Regardless of whether the job is completed successfully or not, we store the job back in Redis inside a data structure serving as `Stats` storage that we will then use to show on our web UI.
- If a job failed, we use the `Job Retry` component along with the `Job Scheduler` component in order to attempt to retry a job sometime in the future. To do so, a future timestamp is placed on the job, it is then stored into Redis into a `Scheduled Jobs` sorted set (a Redis data structure that we will expand on later in this post).
- The `Job Scheduler` will peek the sorted set and compare timestamps to see if there is a job due. If there is then the job will be enqueued into a Job Queue and the cycle continues.

## Building Workerholic

The core of this project was to build a BJP from scratch and sharing our findings, what we learned and the challenges we faced. Next, we will dive into each feature that we deemed belonged into a BJP and how they are incorporated in Workerholic.

### Reliability
A common feature developers want out of a BJP is to be reliable. When performing a job, a network issue that prevents email sending could occur, or the job could be misconfigured. The main application could crash or the BJP could crash. Regardless of the reason, we want to make sure that jobs are not lost.

How can we make our BJP reliable?

#### Jobs Persistence

As mentioned above, a question that needs to be answered is: how can we make sure our jobs don't get lost if the BJP or the main application crashes?
To solve this problem we introduced a data store. This data store is used to persist the serialized jobs that have been enqueued by the main application.
For Workerholic, we decided to use Redis because of its following features:
- convenient data structures for the problems we needed to solve (lists, sorted sets, hashes)
- persistence to disk (every 5 minutes by default, configurable)
- key:value data store, which grants easy access to the data
- in memory data store, which allows for very efficient reads and writes

![Jobs Persistence Diagram](/images/jobs_persistence_redis.png)

By relying on Redis and its robustness we made Workerholic reliable. Redis helps solve the problem of when either the web application crashes or BJP itself crashes, the jobs stored in Redis will be persisted. In case of Redis crashing, we will also have the jobs persisted to disk thanks to the database snapshots taken by Redis.

#### Retrying Failed Jobs & Job Scheduler

![Retry Failed Jobs Diagram](/images/job_retry.png)

Jobs can also fail for numerous reasons that may or may not be in the developer's control such as temporary network issues, timeouts, invalid job, etc. Regardless of the reason, Workerholic will attempt to retry a job. The way we set up job retrying is to schedule it for some time in the future, effectively turning a failed job into a scheduled job. Here is a little bit of code to show you what that looks like:

```ruby
module Workerholic
  class JobRetry
    # ...

    def retry
      return if job.retry_count >= 5

      increment_retry_count
      schedule_job_for_retry
      Workerholic.manager
                .scheduler
                .schedule(JobSerializer.serialize(job), job.execute_at)
    end

    # ...
  end
end
```

Our jobs are simply Ruby objects with an attribute called `retry_count`, as the name suggests, keeps track of how many times a job has been retried. We will retry a job up to five times, if it ends up failing that many times. At that point, it's more likely that there's a problem with the job itself than something wrong with a component that's not in your control. In which case, we log that the job has failed and store that data into our stats and you as the developer can figure out what went wrong.

As we mentioned earlier, we `JobRetry` enlists the help of `JobScheduler` to schedule a time for a failed job to be executed again, effectively turning it into a scheduled job:

```ruby
module Workerholic
  class JobScheduler
    # ...

    def start
      @scheduler_thread = Thread.new do
        enqueue_due_jobs while alive
      end
    end

    #...

    def enqueue_due_jobs
      if job_due?
        while job_due?
          serialized_job, job_execution_time = sorted_set.peek
          job = JobSerializer.deserialize(serialized_job)
          queue = job.queue ? Queue.new(job.queue) : Queue.new

          queue.enqueue(serialized_job)

          sorted_set.remove(job_execution_time)
        end
      else
        sleep(2)
      end
    end

    # ...
  end
end
```

When Workerholic first boots up, we have a manager that `start`s a new scheduler thread which continuously calls `enqueue_due_jobs`. In `enqueue_due_jobs`, we have a private method that checks if there are any jobs due. If there is, we take a `peek` at our sorted set, deserialize the job, put it in the correct queue, and remove that job from the sorted set.

#### Graceful Shutdown
So now that we have solved the issue of what to do in terms of application failure or job failures, we also want to handle what happens if you decide to shut down our Workerholic manually. We of course want to handle that gracefully to prevent future complications from occurring when you start up Workerholic again.

```ruby
module Workerholic
  class Manager
    # ...

    def start
      worker_balancer.start
      workers.each(&:work)
      scheduler.start

      sleep
    rescue SystemExit, Interrupt
      logger.info("Workerholic's process #{Process.pid} is gracefully\
       shutting down, letting workers finish their current jobs...")
      shutdown

      exit
    end

    # ...

    def shutdown
      workers.each(&:kill)
      worker_balancer.kill
      scheduler.kill
      Starter.kill_memory_tracker_thread

      workers.each(&:join)
      scheduler.join
    end

    # ...
  end
end
```

When Workerholic does detect some kind of interruption, we call `shutdown` where we kill our workers and other internal components. We have not discussed threads yet, but for those of you who are familiar with threads, keep in mind that our `kill` methods here are not the same as `Thread.kill`:

```ruby
module Workerholic
  class Worker
    # ...

    def kill
      self.alive = false
    end

    # ...
  end
end
```

Our internal `kill` method simply sets the `alive` instance variable from `true` to `false`. Afterwards, we join each of our worker threads with the main thread. This way, we ensure that the workers can have a chance to finish their final jobs before actually shutting down.

### Efficiency
Let's shift gears a little bit and start talking about how Workerholic is efficient.

#### An Example Scenario

##### Scenario
Suppose we have a huge Rails application with the follow statistics:
* 1000 average queries per second (QPS)
* 10% of those is email sending
* 1% image processing

Also suppose that we a machine with the follow specs:
* 4GB RAM
* 4 CPU cores.

##### Challenge
The challenge we have here is how do we maximize the use of available resources?

#### Concurrency

##### Email Jobs calculations
To begin braeking down our example scenario, let's make the assumption that it would take an average of 50ms to send an email; this is the time it takes between making the request and getting a response from the email service. If we take a 24 hour window, at 86400 seconds per day and 1000 QPS, we will get 86.4 million requests per day, 10% of which are email which means 8.64 million email sending background jobs. If each takes an average of 50ms, it means that in a span of 24 hours, we have a list of jobs that will take 120 hours to process, which gives us an enqueuing:processing ratio of 1:5. If we left this unchecked and allowed these to be performed synchronously, then over longer periods of time, we will get a backlog of background jobs.

![serialized_job_length_redis](/images/serialized_job_length_redis.png)

We pushed 100,000 jobs into our main queue. We found that on average, each serialized job in Workerholic takes up 26 bytes in Redis (`serializedlength` / 100,000). After a week, we'd have a backlog of 48M jobs that still needs to be processed which is equivalent to 1.18GB of memory stored. So the challenge for us here is how do we even out enqueuing throughput and processing throughput? We should not slow down the enqueuing throughput or else your web application will face the similar unresponsive issues again waiting for the jobs to enqueue before moving on, so let's focus on increasing the processing throughput.

##### Concurrency & Threads
So we found that because our jobs would backlog to 120h in a span of 24h, that gives us a 1:5 enqueuing:processing ratio, meaning that we need 5 workers working concurrently in order to enqueue and process on a 1:1 ratio.

![efficiency_OS_scheduler_threads](/images/efficiency_OS_scheduler_threads.png){:width="400"}

Threads in general are controlled by your OS schedule and invokes context switching to switch between threads.

![efficiency_mri_gil](/images/efficiency_mri_gil.png){:width="450" height="200"}

In MRI (Matz Ruby Interpreter, aka CRuby, the main Ruby implementation that we are all used to), threads enable concurrency but do not execute in parallel. This is because of the global interpreter lock (GIL) that currently exists in MRI, which means only a single thread can be running at any given time. Each thread can be scheduled to receive some CPU time by the OS scheduler, but thanks to the GIL, a Ruby process running in MRI cannot receive computational resources from more than one core.

Concurrency and parallelism are often mixed up, as it did for us. Let us offer an example to hopefully clarify that if these two concepts are still fuzzy: say that you are a single developer working on two projects, A and B. You can only work on a single project at any given time. Maybe you're working on project A now, and you want to work on project B after project A is 20% complete. You can do that, but you can never focus on both project A and B at the same time. It would take you A + B time to complete both projects. That is *concurrency* - you are working concurrently to complete both projects. Now let's say there are two developers, you and a colleague; you work on project A while your colleague works on project B, you can swap projects anytime in the middle or both complete your respective projects. This would take you (A + B) / 2 time. That is *parallelism* - you and your colleague are working in parallel to complete both projects.

With that said, going back to the limitations of MRI only being able to run concurrently, it does not seem like it would make much of a difference to use more threads, if only one thread can run at a time. However, because email sending is considered an IO-blocking job, a job where you have idle time, the OS scheduler can schedule another thread to be active while the current active thread is idly waiting for a response. The point here is if a job has 99% of its execution time sitting idly, more threads will help reduce the total execution time than if you were to only have a single thread working sequentially. That's because even if a thread is not active, the idle time is still being "worked on".

![benchmark_workers_count](/images/benchmark_workers_count.png)

And as you can see from the results here: for non-blocking and CPU-blocking jobs, having threads don't help the situation. In fact, it makes it worse due to the overhead incurred from switching between threads. But if you look at the IO-blocking jobs: with one worker, it would've taken very long, 5034 seconds in fact and way off the chart; the y-axis has been capped to give you a better representation of the rest of the data. With 25 workers, the tasks perform almost 25x as fast. At 100 workers, it's almost 4 times as fast as 25 workers.

##### Concurrency in Workerholic
![efficiency_concurrency_workerholic](/images/efficiency_concurrency_workerholic.png){:width="700"}

In the context of Workerholic, we introduced concurrency in order to improve performance by having our workers poll and perform the jobs from within a thread. This way, as shown earlier, if jobs are IO bound we can make use of concurrency in order to maximize the dequeuing and processing throughput and bring that enqueueing:processing ratio down.

```ruby
module Workerholic
  class Worker
    # ...

    def work
      @thread = Thread.new do
        while alive
          serialized_job = poll
          JobProcessor.new(serialized_job).process if serialized_job
        end
      end
    rescue ThreadError => e
      @logger.info(e.message)
      raise Interrupt
    end

    # ...
  end
end
```

We have our workers `poll` Redis. If `poll` returns something, we process that job using `JobProcessor`.

##### Threads and Memory Consumption
![efficiency_concurrency_threads_memory](/images/efficiency_concurrency_threads_memory.png){:width="450"}

Processes have threads, and these threads can spawn more threads. These threads have their independent stacks, but they share a common heap belonging to the process that the threads belong to. For our email-sending job example above, we can spawn as many threads as we'd like to bring down our enqueuing:processing ratio. But what happens to our memory footprint?

![memory_usage_threads](/images/memory_usage_threads.png){:width="600"}

Not a problem! As you can see in the graph above, having 24 more threads do not increase your memory consumption significantly. That is because threads are cheap, and most of the memory footprint comes from the heap of the process.

##### Concurrency Issues & Thread-Safety
While spawning more threads is cheap and significantly increases processing throughput, multi-threading introduces a new concern called *thread-safety*. When code is "thread-safe", it means that the state of the resources behave correctly when multiple threads are using and modifying those resources.

In MRI, core methods are thread-safe, so we don't need to worry about them. However, user-spaced code is not thread-safe, because it may introduce race conditions. Race conditions occur when two or more threads are competing to modify the same resource. This happens because the atomicity of the operation is not guaranteed and the OS scheduler can interrupt the execution of code at any time and schedule another thread. Let's take a look at the `PaymentJob` class:

```ruby
$balance = 0

class PaymentJob
  def perform
    current_balance = $balance

    new_balance = current_balance
    1_000_000.times { new_balance += 1 }

    $balance = new_balance
  end
end

10.times.map do
  Thread.new do
    PaymentJob.new.perform
  end
end.each(&:join)

puts "Final balance: #{$balance}"
```

Above we have a global variable `$balance` and a `PaymentJob` class with a instance method `perform` which modifies `$balance`. Then we create 10 threads, and have each of those threads create a new instance of `PaymentJob` and calls `perform` on the instance, each instance trying to increment `$balance` to 1,000,000. At the end, we should end up with 10,000,000, right?

```
    $ ruby concurrency_issues_example.rb
    $ Final balance: 3000000
    $ ruby concurrency_issues_example.rb
    $ Final balance: 6000000
    $ ruby concurrency_issues_example.rb
    $ Final balance: 5000000

```

As you can see here, that is definitely not the case. Why?

Because the code above is not thread-safe. As mentioned earlier, when we have multiple threads trying to access and modify the same resource, `$balance` in this case, we have a race condition. A thread can enter the `perform` method which first sets `current_balance = $balance`, and then the OS scheduler can pause that thread and run another thread to do the same thing. So now you have two threads (and potentially more) starting its `current_balance` from 0 rather than a stacking multiple of 1,000,000. In the end, your final balance can be any multiple of 1,000,000 between 1,000,000 and 10,000,000. In other words, you cannot guarantee that the code will work as expected, and results may be different each time you run this program. So how do we prevent this?


```ruby
$balance = 0

class PaymentJob
  def perform
    current_balance = $balance

    new_balance = current_balance
    new_balance = current_balance + 1

    $balance = new_balance
  end
end

10.times.map do
  Thread.new do
    1_000_000.times { PaymentJob.new.perform }
  end
end.each(&:join)

puts "Final balance: #{$balance}"
```

```
    $ ruby concurrency_issues_example.rb
    $ Final balance: 10000000
    $ ruby concurrency_issues_example.rb
    $ Final balance: 10000000
    $ ruby concurrency_issues_example.rb
    $ Final balance: 10000000

```

{Insert text here later}

#### Parallelism
Concurrency alone is good enough for IO-blocking jobs, but as you saw in a previous chart, it does nothing for CPU bound blocking jobs. So what do we do?

##### Image Processing Jobs Calculations

##### Parallelism & Processes
![efficiency_parallelism_processes](/images/efficiency_parallelism_processes.png){:width="300"}

![benchmark_2_processes](/images/benchmark_2_processes.png)

##### Parallelism in Workerholic
![parallelism_workeholic_diagram](/images/parallelism_workeholic_diagram.png)

##### Processes and Memory consumption
![efficiency_processes_memory_design](/images/efficiency_processes_memory_design.png)

![memory_usage_processes](/images/memory_usage_processes.png)
### Scalability
#### Scaling in the context of our scenario
![scalibility_image](/images/scalibility_image.png){:width="380" height="350"}

##### Scaling vertically
##### Scaling horizontally
#### Workerholic: a scalable BJP
![scalibility_workerholic](/images/scalibility_workerholic.png)

### Optimizations
#### Serialization
![optimizations_serialization_benchmark_yaml](/images/optimizations_serialization_benchmark_yaml.png)

![optimizations_serialization_benchmark_json](/images/optimizations_serialization_benchmark_json.png)

#### Using Custom Algorithms
![efficiency_algorithms_benchmark](/images/efficiency_algorithms_benchmark.png){:width="400"}

##### Evenly balanced workers
##### Adaptive and Successive Algorithm (ASP)
```ruby
module Workerholic
  class WorkerBalancer
    # ...

    def start
      if auto
        auto_balance_workers
      else
        evenly_balance_workers
      end
    end

    # ...
  end
end
```

```ruby
module Workerholic
  class WorkerBalancer
    # ...

    def auto_balance_workers
      @thread = Thread.new do
        while alive
          auto_balanced_workers_distribution
          output_balancer_stats

          sleep 1
        end
      end
    end

    # ...
  end
end
```

```ruby
module Workerholic
  class WorkerBalancer
    # ...

    def auto_balanced_workers_distribution
      self.queues = fetch_queues

      total_workers_count = assign_one_worker_per_queue

      remaining_workers_count = workers.size - total_workers_count

      average_jobs_count_per_worker =
                            total_jobs / remaining_workers_count.to_f

      total_workers_count = provision_queues(
                              io_queues,
                              average_jobs_count_per_worker,
                              total_workers_count
                            )

      distribute_unassigned_worker(total_workers_count)
    end

    # ...

    def io_queues
      io_qs = queues.select { |q| q.name.match(/.*-io$/) }

      if io_qs.empty?
        queues
      else
        io_qs
      end
    end

    # ...
  end
end
```

![ASP_diagram_1](/images/ASP_diagram_1.png){:width="350"}

```ruby
module Workerholic
  class WorkerBalancer
    # ...

    def provision_queues(qs, average_jobs_count_per_worker, total_workers_count)
      qs.each do |q|
        workers_count = q.size / average_jobs_count_per_worker
        workers_count = round(workers_count)

        assign_workers_to_queue(q, workers_count, total_workers_count)

        total_workers_count += workers_count
      end

      total_workers_count
    end

    # ...
  end
end
```

![ASP_diagram_2](/images/ASP_diagram_2.png){:width="350"}

### Reporting
![reporting_web_ui](/images/reporting_web_ui.png)

#### Real-time Statistics
##### What data?
![reporting_realtime_jobs_per_s](/images/reporting_realtime_jobs_per_s.png)

![reporting_realtime_memory](/images/reporting_realtime_memory.png)

![reporting_real_time_queues](/images/reporting_real_time_queues.png)

##### How to store the data?
```ruby
module Workerholic
  class StatsStorage
    # ...

    def self.save_job(category, job)
      job_hash = job.to_hash
      serialized_job_stats = JobSerializer.serialize(job_hash)

      namespace = "workerholic:stats:#{category}:#{job.klass}"
      storage.add_to_set(namespace, job.statistics.completed_at, serialized_job_stats)
    end

    # ...
  end
end
```

##### How much data?

#### Historical Statistics
##### What data?
![reporting_historical_charts](/images/reporting_historical_charts.png)

##### How to store the data?
###### First Iteration
```ruby
module Workerholic
  class Storage
    # ...

    def self.save_job(category, job)
      job_hash = job.to_hash
      serialized_job_stats = JobSerializer.serialize(job_hash)

      namespace = "workerholic:stats:#{category}:#{job.klass}"
      storage.add_to_set(namespace, job.statistics.completed_at, serialized_job_stats)
    end

    # ...
  end
end
```

###### Second Iteration
```ruby
module Workerholic
  class Storage
    # ...

      def sorted_set_range_members(key, minscore, maxscore, retry_delay = 5)
        execute(retry_delay) { |conn| conn.zrangebyscore(key, minscore, maxscore, with_scores: true) }
      end

    # ...
  end
end
```

```ruby
module Workerholic
  class Storage
    # ...

      def hash_increment_field(key, field, increment, retry_delay = 5)
        execute(retry_delay) { |conn| conn.hincrby(key, field, increment) }
      end

    # ...
  end
end
```

##### How much data?
```ruby
require 'redis'

redis = Redis.new

100_000.times do |i|
  redis.hset('my_key', Time.now.to_i + i, 1000)
end
```

![reporting_historical_redis_size](/images/reporting_historical_redis_size.png)

![reporting_historical_estimations](/images/reporting_historical_estimations.png)

### Configurability
![configurability_CLI](/images/configurability_CLI.png)

### Ease of Use
#### Default Configuration
```ruby
module Workerholic
  # ...

  def self.workers_count
    @workers_count || 25
  end

  # ...

  def self.redis_connections_count
    @redis_connections_count || (workers_count + 3)
  end

  # ...
end
```

```ruby
module Workerholic
  class Starter
    # ...

    def self.launch
      fork_processes if options[:processes] && options[:processes] > 1

      Workerholic.manager = Manager.new(auto_balance: options[:auto_balance])
      Workerholic.manager.start
    end

    # ...
  end
end
```

#### Rails Integration


```ruby
module Workerholic
  class Starter
    # ...

    def self.start
      apply_options
      load_app
      track_memory_usage_and_expire_job_stats
      launch
    end

    # ...

    def self.load_app
      if File.exist?('./config/environment.rb')
        load_rails
      elsif options[:require]
        load_specified_file
      else
        display_app_load_info
      end
    end

    # ...

    def self.load_rails
      require File.expand_path('./config/environment.rb')

      require 'workerholic/adapters/active_job_adapter'

      ActiveSupport.run_load_hooks(:before_eager_load, Rails.application)
      Rails.application.config.eager_load_namespaces.each(&:eager_load!)
    end

    # ...
  end
end
```

### Testing
#### Testing Setup
```ruby
module Workerholic
  # ...

  REDIS_URL = ENV['REDIS_URL'] || 'redis://localhost:' + ($TESTING ? '1234' : '6379')

  # ...
end
```

```ruby
# spec/spec_helper.rb

RSpec.configure do |config|
  # ...

  config.before do
    Redis.new(url: Workerholic::REDIS_URL).flushdb
  end

  # ...
end
```

#### Testing and Threads
```ruby
# spec/worker_spec.rb

it 'processes a job from a thread' do
  queue = Workerholic::Queue.new(TEST_QUEUE)
  worker = Workerholic::Worker.new(queue)

  serialized_job = Workerholic::JobSerializer.serialize(job)
  redis.rpush(TEST_QUEUE, serialized_job)

  worker.work

  expect_during(1, 1) { WorkerJobTest.check }
end
```

```ruby
# spec/helper_methods.rb

def expect_during(duration_in_secs, target)
  timeout = Time.now.to_f + duration_in_secs

  while Time.now.to_f <= timeout
    result = yield
    return if result == target

    sleep(0.001)
  end

  expect(result).to eq(target)
end
```

### Benchmarking Workerholic
#### Workerholic compared to the Gold Standard: Sidekiq
![benchmark_workerholic_sidekiq](/images/benchmark_workerholic_sidekiq.png)

#### JRuby
![benchmark_jruby](/images/benchmark_jruby.png)

## Conclusion
