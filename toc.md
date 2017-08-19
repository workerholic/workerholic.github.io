## Introduction
We are three software engineers..... Yup..

Workerholic is an open source background job processor. Workerholic is a play on the word "workaholic". We built this because we have popular and mature solutions out there like Sidekiq which is widely used, which is great. Libraries are great, but it abstracts the lower level details like threads and processes, concurrency and parallelism, the details that we often take for granted. Even we use libraries a lot, and we thought this would be a good chance to dig deeper and share with you all, give back to the community, what abstractions we learned.

### Why use a background job processor?
So, why do we want to use a background job processor? Well, it's because sometimes, certain methods or tasks can take a long time. We're talking about anywhere from a few hundred milliseconds to sometimes several seconds or even minutes or hours. While your application is performing this time-consuming task, your application becomes blocked, unresponsive, and cannot do anything else. From a user standpoint, that's not good.

![Request-Response](/images/popular_features_example_1.png){:width="400"}

For example, say that you as a user wanted to click a button to send yourself an email with more information about a product you're interested in. This will take a while because this happens over the wire, the web server has to process your request, send out the email, possibly wait for a response from the email service, and then give you back a response from the web server. All during this time, the application won't be able to process anymore requests from you, or otherwise interact with your current page in a responsive manner. This is the classic HTTP request-response cycle, except the request is blocked waiting for a response from the email service, and cannot give you, the client, back a response.

![Example BJP](/images/popular_features_example_2.png){:width="500"}

This is where background job processors come in. Background job processors run as a separate process decoupled from your application. As a developer of your application, you can delegate these time-consuming tasks to a background job processor to handle these time-consuming tasks while allowing the user to still interact with your website. These tasks are performed asynchronously when you have resources available.

## Popular Features of BJPs
Before we get into building a background job processor, let's first introduce some of the common features we see in other background job processors:
##### Core Features:
These are features that all background job processors *should* have.
* **asynchrony**: background job processors should handle a task in a separate process away from the main application, so that your main web application is free to continue handling more requests from the user.
* **reliability**: background job processors should be able to handle any errors that occur gracefully.
* **efficiency:** background job processors should perform their tasks in a timely manner so that queues do not get a backlog of things that still needs to be done.
* **scalability**: background job processors should run fine and scale in the context of a distributed system, such as multiple web servers.

##### Bonus Features:
These are features that are not necessary for background job processors, but these can be added for robustness.
* **configurability**: allows a developer to tweak options that best fits their own application's needs.
* **ease of use**: simple to use out-of-the-box with rails.
* **reporting**: track job statistics for information about background jobs for decision making.

## Introducing Workerholic: Overall Architecture
![Workerholic Overvall Architecture](/images/workerholic_overall_architecture.png)

Above is a diagram of the overall architecture of our take on a background job processor. On the left is a web application that includes our library, the jobs are defined in there, the jobs are serialized and pushed into Redis into "Job Queues". On the right, Workerholic workers poll from the job queues to see if there are any jobs that need to be done; if there is, then the workers will use the "Job Processor" to do the jobs. Regardless of whether the job is completed successfully or not, we store the job back into Redis as "Stats" that we will show on our web UI (not shown here). If a job did fail, we use "Job Retry" together with the "Job Scheduler" to attempt to retry a job sometime in the future. A future timestamp is placed on the job, it gets pushed into Redis into a sorted set as "Scheduled Jobs", the Job Scheduler will peek the sorted set and compare timestamps to see if there is a job due. If there is then the job will be enqueued into a Job Queue and the cycle continues.

## Building Workerholic
Let's start diving into the numerous features we wanted in Workerholic!

### Reliability
A common feature developers want out of background job processors is to be reliable through a number of different reasons. It could be a network issue that prevents email sending, or maybe the job wasn't configured properly, or maybe the background job processor itself crashes. Regardless of the reason, we want to make sure that jobs are aren't just getting lost. Our challenge here is how do we make Workerholic reliable?

#### Jobs Persistence
![Jobs Persistence Diagram](/images/jobs_persistence_redis.png)

In order to make our jobs persistent even through some of those failures, we rely on Redis's robustness to make our library more reliable. Redis helps solve the problem of when either the web application crashes or if the background job processor itself crashes, the jobs that are already stored on Redis will be preserved. But what if Redis itself crashes? This is part of why Redis is considered robust because it takes snapshots of your database every five minutes by default. As an added bonus, you can configure Redis to take snapshots more or less as needed.

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
Concurrency alone is good enough for IO-blocking jobs, but as you saw in a previous chart, it does nothing for CPU bound blocking jobs. Why? And what do we do?

##### Image Processing Jobs Calculations
A common CPU-blocking job is image processing. So now let's say an image processing job takes 4 seconds on average, and recall from earlier, we said that we have a large Rails application with 1000 QPS and 1% of that is image processing, which means we have 864,000 image processing jobs per day, multiply that by 4s and you have 960 hrs worth of processing lined up in a period of 24 hrs, giving us a 1:40 enqueue:processing ratio. Similar to the email example, as the this gets backlogged, we will start to run out of memory. So same challenge: evening out enqueuing and processing throughput, but our previous solution won't work. Why?

##### Parallelism & Processes
![efficiency_parallelism_processes](/images/efficiency_parallelism_processes.png){:width="300"}

As we've said, image processing is CPU-bound, because this type of job requires CPU time only, there is no idle time in which the thread can be put to sleep. This means that having more threads will not help the situation because your CPU core is already working full-time to process this job, and we cannot take advantage of concurrency and multithreading for this type of job in the context of MRI. Since our machine has multiple cores, we can take advantage of this fact by running multiple processes in parallel. When the CPU has multiple cores, the OS scheduler can allocate some computational resources from the different cores to different processes.

![benchmark_2_processes](/images/benchmark_2_processes.png)

Here's we benchmarked how Workerholic performed when we ran 2 processes vs. 1 process. We can see that having more processes mean increase in performance, but at the cost of more memory. In each of these cases, we can see that the time is around half. However, that is not always guaranteed because of the way your CPU cores are assigned by your operating system.

##### Parallelism in Workerholic
![parallelism_workeholic_diagram](/images/parallelism_workeholic_diagram.png)

The diagram above shows how Workerholic uses multiple processes if we had the CPU cores available to do so. The OS scheduler schedules the cores to each process, each process has its own worker threads which can poll jobs from Redis and work on them. So if we have four CPU cores we can have computational resources allocated to four different processes potentially at the same time, effectively allowing them to run in parallel, reducing the enqueuing:processing ratio from 1:40 to 1:10.

##### Processes and Memory consumption
We're able to create more processes, but surely that comes with a cost like everything else right?

![efficiency_processes_memory_design](/images/efficiency_processes_memory_design.png)

Before we get into that, let's talk a little bit more about processes. Each process has its own address space, stack, and heap. When you fork a process in Ruby, you create a child process which will get its own stack and share the same heap initially. This is called copy-on-write, meaning that the child process will share the same resources with the parent process, until modifications are made to that resource, in which case, that resource will be written into the child's own heap.

![memory_usage_processes](/images/memory_usage_processes.png)

Here, we benchmarked both having one process and two processes. Having one process takes up 125MB of memory, but having two processes don't take twice as much memory. This is the copy-on-write mechanism at work.

As we mentioned previously, using our four CPU cores, we can fork to a total of four processes and reduce our image processing enqueuing:processing ratio down to 1:10. But that is still not good enough. After an extended period of time, we will eventually end up with a backlog and a huge memory footprint. Where do we go from here?

### Scalability

#### Scaling in the context of our scenario
![scalibility_image](/images/scalibility_image.png){:width="380" height="350"}

Currently, we are using all of our resources on hand to evenly processing throughput, but with an enqueuing:processing ratio of 1:10. It is still not good enough. At this point, we have to scale our system. We have two options:

##### Scaling vertically
We can get more cores on a single machine. If we get 20 cores, we still have a 1:2 enqueuing:processing ratio, so we will need 40 cores. This does not sound very plausible. Have you ever heard of a single machine having this many cores?

##### Scaling horizontally
We can instead buy more servers with less cores on each. We can buy 10 servers, each with four cores for a total of 40 cores and that will get the job done. This is way more realistic, as it would be akin to buying more worker dynos on Heroku for example.

#### Workerholic: a scalable BJP
Now we know that we want to scale horizontally. The question is how?

![scalibility_workerholic](/images/scalibility_workerholic.png)

Well, that's easy. Workerholic is already scalable because we use Redis as a central data store for your jobs. Workerholic will still need access to the source code of your application, but Workerholic is not tied to a specific instance of your application, so you can have distributed web servers or a single web server and Workerholic will still work just fine. Workerholic can do this because its workers only care about the queue they're polling from, which is centralized with Redis.

### Optimizations
Once we had a fairly featured solution, we decided to compare against Sidekiq.

#### Serialization
![optimizations_serialization_benchmark_yaml](/images/optimizations_serialization_benchmark_yaml.png)

On our first iteration, we found that there was a great difference between Workerholic and Sidekiq; ours took much longer both on the enqueuing side and the processing side. Why was that? We looked into Sidekiq and found that it was using JSON serialization while we were using YAML, and so we decided to change our code to use JSON and see if that was really where the bottleneck was.

![optimizations_serialization_benchmark_json](/images/optimizations_serialization_benchmark_json.png)

And voila! Here are the results above. By changing our serialization strategy to JSON, we were able to improve our enqueuing duration by 70% and our processing duration by 55% compared to our YAML iteration. We're now slightly faster than Sidekiq, but that's only because Sidekiq is more robust.

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
