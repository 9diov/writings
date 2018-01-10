# Managing Sidekiq workers

At Holistics, we use [Sidekiq](http://sidekiq.org/), the de-factor standard background processing framework for Ruby. By combining the reliability feature of Sidekiq Pro and our own retry logic, we can be assured that no back-end job get lost or forgotten. Yet for a while, there are issues that we haven't been able to solve effectively.

## Gracefully restarting worker processes

In order to restart a Sidekiq worker, the recommended way is to send SIGTERM to the worker process with a predefined timeout then spawn a new process. This works well if all the jobs are short-lived, surely to complete within the timeout period. Yet not all jobs fit that behavior pattern. For example, we use Sidekiq background workers to send complex queries to customers' databases. Most of these queries running time cannot be predicted beforehand. If a worker hasn't finished its work by the end of the timeout period, it would still be forcefully shut down. With Sidekiq Pro's reliability feature, the job is not lost and would be restarted automatically, but the customer's databases now have two identical queries running, wasting resources.

Luckily, Sidekiq provides another building block for gracefully restart: the USR1 signal. When a Sidekiq process receives this signal, it will stop accepting new jobs but continue to work on the current ones. A Sidekiq process executing in this state is called 'quiet'. By waiting for the quiet Sidekiq process to finish its jobs, we can safely restart it with SIGTERM without worrying about the implications once all jobs completed.

Now the question is how do we know when to send the SIGTERM signal? By either looking at the process name from 'ps' command or getting the status using Sidekiq API, we can see how many jobs are still being executed by a specific worker process. Since we don't know when number of jobs would reach zero, we need a periodical check on these status. Thus we need a Sidekiq process manager script.

## The manager script's requirements

The job of this Sidekiq manager script is to:

* Periodically check the status of the Sidekiq processes. If a project is in quiet state and has no running jobs, terminate it with SIGTERM signal.
* Received graceful restart command from administrator and quiet all currently active processes. It should also spawn new processes to handle new jobs. Number of newly spawn processes should be configurable to ensure that there are enough workers to work on the jobs.

Our requirements are now pretty clear:
* Worker processes are never interrupted (SIGTERM/SIGKILL) when there are still running workers
* There are at least X number of running worker processes at any time to handle the workload. X is configurable.
* Allow for graceful restart of the process during deployment, meaning both condition 1 and 2 are guaranteed, while ensure the new code change is in effect as soon as possible

In our case, there is another requirement.

## Avoiding the dreadful OOM killer - the last requirement

One of the issues that we have faced is the fact that a long-running worker process will not release memory, causing its memory footprint to increase to the point that it got killed by the OS's out-of-memory killer. Thus, our last requirement is:

* A worker process, once reaches a predetermined memory threshold, shouldn't accept new jobs (thus allocating more memory) by becoming quiet

## Designing the management script

### Input/output

The process manager accept the followings configuration options:

* ACTIVE_PROCESS_COUNT: Guaranteed number of active processes. Active here means the worker processes are not stopped or quiet, can accept new jobs.
* MEMORY_THRESHOLD: The threshold at which point the process is quiet and not accepting new jobs.

The process manager also accepts the following command:

* restart: graceful restart of the worker processes, typically used during deployment or hotfix

### Operations

Every x (say 5) minutes, this algorithm will be executed:

* Scan data from Sidekiq API, 'ps' command or from the process manager (Upstart/Supervisor/Systemd)
* Each process can be in one of the three states: running/stopped/quiet
* Go through all quiet processes and stop all with 0 running workers
* Go through all running processes that exceed memory threshold and quiet them
* If number of running workers < ACTIVE_PROCESS_COUNT, start the stopped ones to ensure number of running workers >= ACTIVE_PROCESS_COUNT

Every time restart command is triggered:

* Start ACTIVE_PROCESS_COUNT processes to handle new jobs
* Quiet all running processes

## Conclusion

With the management script now in production, we are now assured that the background system is reliable, no jobs get re-executed and we can gracefully restart the workers during deployment. In other words, no complaint from customers regarding duplicated job execution and good sleep at night!

