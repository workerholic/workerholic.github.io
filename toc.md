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

This solves the problem of the main application crashing or the BJP crashing. What should be done if the jobs themselves crash/raise an exception?

#### Retrying Failed Jobs & Job Scheduler

Jobs can fail for numerous reasons that may or may not be in the developer's control such as temporary network issues, timeouts, invalid job, etc. Regardless of the reason, the BJP needs to handle these errors. In case the error being raised while the job is being performed is a momentary error, then it might be a good idea to have a way to retry this job later in the future.

![Retry Failed Jobs Diagram](/images/job_retry.png)

Workerholic will attempt to retry failed jobs. The way we set up job retrying is to schedule it for some time in the future, effectively turning a failed job into a scheduled job. Here is the implementation of Workerholic's `retry` functionality:

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

Jobs are wrapped with Ruby objects which have an attribute called `retry_count`. As the name suggests, it is used to keep track of how many times a job has been retried. A job will be retried up to five times. At that point, it's more likely that there's a problem with the job itself rather than something wrong with an external component. In which case, Workerholic will log that the job has failed and store statistics data into Redis so that Workerholic's users can figure out what went wrong.

The code snippet above also shows that `JobRetry` enlists the help of `JobScheduler` to schedule a time for a failed job to be executed again, effectively turning it into a scheduled job. Here is how `JobScheduler` schedules jobs and enqueues jobs ready to be executed:

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

When Workerholic first boots up, its `Manager` component is in charge of starting a new `JobScheduler` thread which continuously calls `enqueue_due_jobs`, every two seconds. In `enqueue_due_jobs`, a call to a private method `job_due?` checks if there are any jobs due. If there is, `JobScheduler` takes a `peek` at the scheduled jobs sorted set, deserializes the job, puts it in the correct queue, and removes that job from the sorted set.

With this feature, if a job fails it will be retried, following a specific retry logic. But there is still a scenario during which jobs could be lost. What should be done if the BJP is stopped while some jobs were being executed?

#### Graceful Shutdown

When shutting down the BJP some jobs might still be executing and not yet done. These jobs would no longer be stored in Redis and would, consequently, be lost upon Workerholic shutting down. The way Workerholic handles this specific reliability issue is by performing a **graceful shutdown**:

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

When Workerholic detects an `Interrupt` or a `SystemExit`, it calls `shutdown`, which in turns kills its workers and other internal components. The way a worker is killed is by turning its `alive` status to `false`:

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

This will allow the worker to finish executing its current job. Afterwards, we `join` each of our worker with our main thread of execution. We will touch on why this is important in a later section, for now it is only useful to know that the workers will be waited on before Workerholic exits.

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

And voila! Here are the results above. By changing our serialization strategy to JSON, we were able to improve our enqueuing duration by 70% and our processing duration by 55% compared to our YAML iteration. We're now slightly faster than Sidekiq!

#### Using Custom Algorithms
We have improved our efficiency by changing our serialization strategy. Can we do better? We benchmarked against Sidekiq using 10,000 nonblocking, cpu-blocking, and IO-blocking jobs.

![efficiency_algorithms_benchmark](/images/efficiency_algorithms_benchmark.png){:width="400"}

With Sidekiq's random polling algorithm, it took 242 seconds. Our turn. We benchmarked with our evenly balancing algorithm with an even number of workers for each queue regardless of job types or queue load, and we stand at 357s. Much worse...

Next, we tried auto-balancing workers based on queue load, but still no assumption on jobs. Slightly better, but still not good enough. We went through a second iteration of auto-balancing where we identified IO-bound queues and CPU-bound queues, where we assign only one worker per CPU-bound queue and auto-balance the rest. As you may recall, having more workers on CPU-blocking jobs makes no difference, which is a waste of Workerholic's resources. With that we rang in at 223s, which is great! We call this algorithm the Adaptive and Successive Provisioning (ASP).

*Note: we want to quickly mention that we did not build Workerholic to compete with Sidekiq, and that you should not prefer our library to theirs' or vice-versa. We just chose Sidekiq to benchmark against because it is the leader of background job processing in Ruby and we thought that is the bar we should aim for. Could we do better? Maybe, maybe not. We haven't tried to yet, because we were satisfied with the current results.*

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

When Workerholic starts, so does our `WorkerBalancer`, which will default to evenly balancing workers unless an `auto` option is detected.

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

Here, we're showing what happens when Workerholic auto-balances its workers. It will auto-balance every second.

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

Workerholic will `assign_one_worker_per_queue`, then take the total number of jobs in all the queues, divide that by the number of our remaining workers to get the average number of jobs each remaining worker should be responsible for, and we provision the queues accordingly, only to the IO-bound queues.

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

Once an average is calculated, Workerholic divides each of its queue sizes by the average, get a workers count, and assign that many extra workers to their respective queues.

### Reporting
Let's move on to reports. It is an important feature because it allows a developer to gain insight into the state of overall jobs, the job types, jobs that failed, and how many jobs are completed over time, which the developer can use to make optimized decisions.

![reporting_web_ui](/images/reporting_web_ui.png)

Above is what our web UI looks like. This tracks real-time data, polling every 10 seconds.

#### Real-time Statistics
##### What data?
![reporting_realtime_jobs_per_s](/images/reporting_realtime_jobs_per_s.png)

![reporting_realtime_memory](/images/reporting_realtime_memory.png)

![reporting_real_time_queues](/images/reporting_real_time_queues.png)

We decided to have Workerholic show our users aggregate data for finished jobs, queued jobs, scheduled jobs, failed jobs, current number of queues, and the memory footprint over time, as well as the breakdown of jobs from each class. All this data is updated every 10 seconds, using AJAX on the front-end to query our internal API for the data.

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

Now we have what data we wanted to track and store. Next we needed to ask how we should store our data? We decided to use Redis because there's no need for a new dependency, very efficient writes and reads (in-memory store), automatic persistence to disk, and have the tools available to store serialized jobs, like using a sorted set.

##### How much data?
once we had a foundation for live data, we should think about how much data we should store. Initially, we decided we do not need to store live data, and to just poll new data every 10 seconds. This posed a problem though once we introduced graphs into the web UI; just polling for new data and throwing away stale data was no longer an option. A quick-fix for this was to just store data on the front-end for a certain number of data points to create the graph. This worked, but only if the user stayed on the page. If the user navigated away from the page. The data would've been lost. Instead, we looked at Redis to store this data, up to 1000 seconds, for a total of 100 data points. Currently, our graphs only show up to 240 seconds, so this many data points is unnecessary, but it may become necessary if we decided to cover more time with our graphs.

#### Historical Statistics
Live data, check! Next, we want to display historical, and first we had to think about what type of data to store?

##### What data?
![reporting_historical_charts](/images/reporting_historical_charts.png)

Since historical data is looking into the past, for now we decided that we'll just store aggregated data of completed and failed jobs, as well as the breakdown for each class, up to 365 days.

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

In our first iteration, we used a sorted set using the beginning of the day as a timestamp to use as scores, which would be very easy to retrieve from Redis - using a range of scores to display data for 7 or 30 days for example. However, we ran into concurrency issues because we have three ways to update aggregated data: getting the count, removing the count, and incrementing the count.

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

In our second iteration, we decided to use a hash instead. Same as before, using the beginning of the day timestamps as hash fields, and this way we push the computation logic down to the Redis level when we retrieve our data. Redis also has a very nice convenient hash method for incrementing fields.

##### How much data?
Once we have figured out how we wanted to store the data, we need to think about how many data points to store? The goal for us here is to be able to store data for up to a year.

```ruby
require 'redis'

redis = Redis.new

100_000.times do |i|
  redis.hset('my_key', Time.now.to_i + i, 1000)
end
```

![reporting_historical_redis_size](/images/reporting_historical_redis_size.png)

 We set 10,000 and 100,000 hash fields in Redis to get an average of how much memory each field would take, which comes to an average of 8 bytes. We went through iterations of how much memory each would take:

![reporting_historical_estimations](/images/reporting_historical_estimations.png)

We make an assumption that there would be 25 different job classes, and from there if we took one data point a day, that would give us 9000 fields which translates to 0.1MB, once an hour for 219,000 fields which translate to 1.7MB, and once per minute for 13M fields which translates to 100MB. From this, we realized that once/day is the only viable solution to be able transfer information over the wire quickly.

### Configurability
Moving on to our first bonus feature: configurability. We wanted Workerholic to be versatile and satisfy the needs of the developer. Background job processors are powerful in what they accomplish. But all applications are different. Some may have a million jobs per day, while some maybe only have 10. In which case, we want our background job processor to have the option for the developer to change what they want to best suit their application's needs.

![configurability_CLI](/images/configurability_CLI.png)

The configurability options we included are auto-balancing workers, an option to set the number of workers based on your application's needs, an option to load your application by supplying a path, an option to specify the number of processes you want to spin up, and the number of connections in the Redis connection pool. All those options are packaged up into a simple and intuitive API. And like all other command-line tools you've experienced, we have the `--help` flag to show you how to use these options.

### Ease of Use
Next bonus feature: ease of use. We wanted to make Workerholic easy to use and work right our of the box, as well as make it friendly with the popular frameworks in the Ruby ecosystem like Rails.

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

To make it work out of the box, Workerholic has default options set up already so you don't need to supply any of the options we mentioned previously. Our default is 25 workers, and the default number of Redis connections is the number of workers + 3, in this case, 28. This is the three additional connections we need for the job scheduler, worker balancer, and the memory trackers.

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

Workerholic also has a default for 1 process and evenly balancing workers. If `options[:processes]` is defined, we fork processes. Otherwise, we just start the manager for one process.

#### Rails Integration
A library built for web applications written in Ruby would be quite useless, or at best very unpopular, if it did not work with Rails.

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

When workerholic starts, it'll load the app which load rails if it detects a specific file in Rails, and then we require our own active job adapter and load that in along with the rest of the rails application.

### Testing
As we developed our features, we needed to tests for our code.

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

To set up, we set up redis with a different port if the environment is testing, to separate it from our development environment. And also, we wanted to flush redis after each run to ensure that it is a valid state for every spec.

#### Testing and Threads
What we found along the way is testing threaded code is not trivial. We spent quite some time trying to figure this out, and this is because having multiple threads means that there is naturally asynchronously execution, meaning that we cannot expect the results immediately. Additionally, there is potential dependency on other threaded components.
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

In order to get around the asynchronous nature of threads, instead of asserting the final state of the system, we expect a certain state of the system to be mutated within a specified timeframe.

### Benchmarking Workerholic
#### Workerholic compared to the Gold Standard: Sidekiq
![benchmark_workerholic_sidekiq](/images/benchmark_workerholic_sidekiq.png)

Finally, we wanted to compare with Sidekiq one last time with each types of jobs individually. We're on par with Sidekiq, and only slightly faster than Sidekiq each time, and as we mentioned before this is because Sidekiq is a more mature and robust solution with many more features and handles more use cases.

#### JRuby
![benchmark_jruby](/images/benchmark_jruby.png)

We also decided to compare the results of jRuby vs MRI. Because jRuby can run in parallel without the need of spinning up multiple processes, we found that CPU blocking jobs were much faster in jRuby than in MRI, which is what we would expect.

## Conclusion
N/A.
