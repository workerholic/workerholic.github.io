## Introduction
![Request-Response](/images/popular_features_example_1.png){:width="400"}

![Example BJP](/images/popular_features_example_2.png){:width="500"}

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

### Efficiency

#### An Example Scenario

##### Scenario

##### Challenge

#### Concurrency

##### Email Jobs calculations
![serialized_job_length_redis](/images/serialized_job_length_redis.png)

##### Concurrency & Threads
![efficiency_OS_scheduler_threads](/images/efficiency_OS_scheduler_threads.png){:width="400"}

![efficiency_mri_gil](/images/efficiency_mri_gil.png){:width="450" height="200"}

![benchmark_workers_count](/images/benchmark_workers_count.png)

##### Concurrency in Workerholic
![efficiency_concurrency_workerholic](/images/efficiency_concurrency_workerholic.png){:width="700"}

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

##### Threads and Memory Consumption
![efficiency_concurrency_threads_memory](/images/efficiency_concurrency_threads_memory.png){:width="450"}

![memory_usage_threads](/images/memory_usage_threads.png){:width="600"}

##### Concurrency Issues & Thread-Safety
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

    $ ruby concurrency_issues_example.rb
    $ Final balance: 3000000
    $ Final balance: 6000000
    $ Final balance: 7000000

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
    100_000.times { PaymentJob.new.perform }
  end
end.each(&:join)

puts "Final balance: #{$balance}"
```

    $ ruby concurrency_issues_example.rb
    $ Final balance: 10000000
    $ Final balance: 10000000
    $ Final balance: 10000000

#### Parallelism
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
