# Goose

Have you ever been attacked by a goose?

[![crates.io](https://img.shields.io/crates/v/goose.svg)](https://crates.io/crates/goose)
[![Documentation](https://docs.rs/goose/badge.svg)](https://docs.rs/goose)
[![Apache-2.0 licensed](https://img.shields.io/crates/l/goose.svg)](./LICENSE)
[![CI](https://github.com/tag1consulting/goose/workflows/CI/badge.svg)](https://github.com/tag1consulting/goose/actions?query=workflow%3ACI)

## Overview

Goose is a Rust load testing tool inspired by [Locust](https://locust.io/). User behavior is defined with standard Rust code. Load tests are applications that have a dependency on the Goose library. Web requests are made with the [Reqwest](https://docs.rs/reqwest) HTTP Client.

## Getting Started

The [in-line documentation](https://docs.rs/goose/*/goose/#creating-a-simple-goose-load-test) offers much more detail about Goose specifics. For a general background to help you get started with Rust and Goose, read on.

[Cargo](https://doc.rust-lang.org/cargo/) is the Rust package manager. To create a new load test, use Cargo to create a new application (you can name your application anything, we've generically selected `loadtest`):

```bash
$ cargo new loadtest
     Created binary (application) `loadtest` package
$ cd loadtest/
```

This creates a new directory named `loadtest/` containing `loadtest/Cargo.toml` and `loadtest/src/main.rs`. Start by editing `Cargo.toml` adding Goose under the dependencies heading:

```toml
[dependencies]
goose = "^0.10"
```

At this point it's possible to compile all dependencies, though the resulting binary only displays "Hello, world!":

```
$ cargo run
    Updating crates.io index
  Downloaded goose v0.10.0
      ...
   Compiling goose v0.10.0
   Compiling loadtest v0.1.0 (/home/jandrews/devel/rust/loadtest)
    Finished dev [unoptimized + debuginfo] target(s) in 52.97s
     Running `target/debug/loadtest`
Hello, world!
```

To create an actual load test, you first have to add the following boilerplate to the top of `src/main.rs` to make Goose's functionality available to your code:

```rust
use goose::prelude::*;
```

Then create a new load testing function. For our example we're simply going to load the front page of the website we're load-testing. Goose passes all load testing functions a mutable pointer to a GooseUser object, which is used to track metrics and make web requests. Thanks to the Reqwest library, the Goose client manages things like cookies, headers, and sessions for you. Load testing functions must be declared async, which helps ensure that your simulated users don't become CPU-locked.

In load test functions you typically do not set the host, and instead configure the host at run time, so you can easily run your load test against different environments without recompiling. The following `loadtest_index` function simply loads the front page of our web page:

```rust
async fn loadtest_index(user: &GooseUser) -> GooseTaskResult {
    let _goose_metrics = user.get("/").await?;

    Ok(())
}
```

The function is declared `async` so that we don't block a CPU-core while loading web pages. All Goose load test functions are passed in a reference to a `GooseUser` object, and return a `GooseTaskResult` which is either an empty `Ok(())` on success, or a `GooseTaskError` on failure. We use the `GooseUser` object to make requests, in this case we make a `GET` request for the front page, `/`. The `.await` tells frees up the CPU-core while we wait for the web page to respond, and the tailing `?` passes up any unexpected errors that may be returned from this request. When the request completes, Goose returns metrics which we store in `_goose_metrics` variable. The variable is prefixed with an underscore (`_`) to tell the compiler we are intentionally not using the results. Finally, after making a single successful request, we return `Ok(())` to let Goose know this task function completed successfully.

Finally, we have to tell Goose about our new task function. Edit the `main()` function, setting a return type and replacing the hello world text as follows:

```rust
fn main() -> Result<(), GooseError> {
    GooseAttack::initialize()?
        .register_taskset(taskset!("LoadtestTasks")
            .register_task(task!(loadtest_index))
        )
        .execute()?
        .print();
    
    Ok(())
}
```

If you're new to Rust, `main()`'s return type of `Result<(), GooseError>` may look strange. It essentially says that `main` will return nothing (`()`) on success, and will return a `GooseError` on failure. This is helpful as several of `GooseAttack`'s methods can fail, returning an error. In our example, `initialize()` and `execute()` each may fail. The `?` that follows the method's name tells our program to exit and return an error on failure, otherwise continue on. The `print()` method consumes the `GooseMetrics` object returned by `GooseAttack.execute()` and prints a summary if metrics are enabled. The final line, `Ok(())` returns the empty result expected on success.

And that's it, you've created your first load test! Let's run it and see what happens.

```bash
$ cargo run
   Compiling loadtest v0.1.0 (/home/jandrews/devel/rust/loadtest)
    Finished dev [unoptimized + debuginfo] target(s) in 3.56s
     Running `target/debug/loadtest`
Error: InvalidOption { option: "--host", value: "", detail: "A host must be defined via the --host option, the GooseAttack.set_default() function, or the GooseTaskSet.set_host() function (no host defined for LoadtestTasks)." }
```

Goose is unable to run, as it hasn't been told the host you want to load test. So, let's try again, this time passing in the `--host` flag. After running for a few seconds, we then press `ctrl-c` to stop the load test:

```bash
$ cargo run -- --host http://local.dev/
    Finished dev [unoptimized + debuginfo] target(s) in 0.07s
     Running `target/debug/loadtest --host 'http://local.dev/'`

=== PER TASK METRICS ===
------------------------------------------------------------------------------ 
 Name                    | # times run    | # fails        | task/s | fail/s
 ----------------------------------------------------------------------------- 
 1: LoadtestTasks        |
   1:                    | 2,430          | 0 (0%)         | 303    | 0    
-------------------------------------------------------------------------------
 Name                    | Avg (ms)   | Min        | Max        | Median    
 ----------------------------------------------------------------------------- 
 1: LoadtestTasks        |
   1:                    | 14         | 6          | 27         | 14        

=== PER REQUEST METRICS ===
------------------------------------------------------------------------------ 
 Name                    | # reqs         | # fails        | req/s  | fail/s
 ----------------------------------------------------------------------------- 
 GET /                   | 2,430          | 0 (0%)         | 303    | 0    
-------------------------------------------------------------------------------
 Name                    | Avg (ms)   | Min        | Max        | Median    
 ----------------------------------------------------------------------------- 
 GET /                   | 14         | 5          | 27         | 14        

All 8 users hatched, resetting metrics (disable with --no-reset-metrics).

^C06:03:25 [ WARN] caught ctrl-c, stopping...

=== PER TASK METRICS ===
------------------------------------------------------------------------------ 
 Name                    | # times run    | # fails        | task/s | fail/s
 ----------------------------------------------------------------------------- 
 1: LoadtestTasks        |
   1:                    | 1,274          | 0 (0%)         | 424    | 0    
-------------------------------------------------------------------------------
 Name                    | Avg (ms)   | Min        | Max        | Median    
 ----------------------------------------------------------------------------- 
 1: LoadtestTasks        |
   1:                    | 20         | 8          | 213        | 19        

=== PER REQUEST METRICS ===
------------------------------------------------------------------------------ 
 Name                    | # reqs         | # fails        | req/s  | fail/s
 ----------------------------------------------------------------------------- 
 GET /                   | 1,274          | 0 (0%)         | 424    | 0    
-------------------------------------------------------------------------------
 Name                    | Avg (ms)   | Min        | Max        | Median    
 ----------------------------------------------------------------------------- 
 GET /                   | 20         | 8          | 213        | 18        
-------------------------------------------------------------------------------
 Slowest page load within specified percentile of requests (in ms):
 ------------------------------------------------------------------------------
 Name                    | 50%    | 75%    | 98%    | 99%    | 99.9%  | 99.99%
 ----------------------------------------------------------------------------- 
 GET /                   | 18     | 21     | 52     | 58     | 210    |    210
 ```

By default, Goose will hatch 1 GooseUser per second, up to the number of CPU cores available on the server used for load testing. In the above example, the server has 8 CPU cores, so it took 8 seconds to hatch all users. After all users are hatched, Goose flushes all metrics collected during the hatching process so all subsequent metrics are taken with all users running. Before flushing the metrics, they are displayed to the console so the data is not lost.

The same metrics are displayed per-task and per-request. In our simple example, our single task only makes one request, so in this case both metrics show the same results.

The per-task metrics are displayed first, starting with the name of our Task Set, `LoadtestTasks`. Individual tasks in the Task Set are then listed in the order they are defined in our load test. We did not name our task, so it simply shows up as `1: `. All defined tasks will be listed here, even if they did not run, so this can be useful to confirm everything in your load test is running as expected.

Next comes the per-request metrics. Our single task makes a `GET` request for the `/` path, so it shows up in the metrics as `GET /`. Comparing the per-task metrics collected for `1: ` to the per-request metrics collected for `GET /`, you can see that they are the same.

There are two common tables found in each type of metrics. The first shows the total number of requests made (1,274), how many of those failed (0), the average number of requests per second (424), and the average number of failed requests per second (0).

The second table shows the average time required to load a page (20 milliseconds), the minimum time to load a page (8 ms), the maximum time to load a page (213 ms) and the median time to load a page (19 ms).

The per-request metrics include a third table, showing the slowest page load time for a range of percentiles. In our example, in the 50% fastest page loads, the slowest page loaded in 18 ms. In the 75% fastest page loads, the slowest page loaded in 21 ms, etc.

In real load tests, you'll most likely have multiple task sets each with multiple tasks, and Goose will show you metrics for each along with an aggregate of them all together.

Refer to the [examples directory](https://github.com/tag1consulting/goose/tree/master/examples) for more complicated and useful load test examples.

## Tips

* Avoid `unwrap()` in your task functions -- Goose generates a lot of load, and this tends
to trigger errors. Embrace Rust's warnings and properly handle all possible errors, this
will save you time debugging later.
* When running your load test for real, use the cargo `--release` flag to generate
optimized code. This can generate considerably more load test traffic.

## Simple Example

The `-h` flag will show all run-time configuration options available to Goose load tests. For example, pass the `-h` flag to the `simple` example, `cargo run --example simple -- -h`:

```
Usage: target/debug/examples/simple [OPTIONS]

Options available when launching a Goose load test.


Optional arguments:
  -h, --help                 Displays this help
  -V, --version              Prints version information
  -l, --list                 Lists all tasks and exits

  -H, --host HOST            Defines host to load test (ie http://10.21.32.33)
  -u, --users USERS          Sets concurrent users (default: number of CPUs)
  -r, --hatch-rate RATE      Sets per-second user hatch rate (default: 1)
  -t, --run-time TIME        Stops after (30s, 20m, 3h, 1h30m, etc)
  -g, --log-level            Sets log level (-g, -gg, etc)
  -L, --log-file NAME        Enables log file and sets name
  -v, --verbose              Sets debug level (-v, -vv, etc)

Metrics:
  --only-summary             Only prints final summary metrics
  --no-reset-metrics         Doesn't reset metrics after all users have started
  --no-metrics               Doesn't track metrics
  --no-task-metrics          Doesn't track task metrics
  -m, --metrics-file NAME    Sets metrics log file name
  --metrics-format FORMAT    Sets metrics log format (csv, json, raw)
  -d, --debug-file NAME      Sets debug log file name
  --debug-format FORMAT      Sets debug log format (json, raw)
  --status-codes             Tracks additional status code metrics

Advanced:
  --throttle-requests VALUE  Sets maximum requests per second
  --sticky-follow            Follows base_url redirect with subsequent requests

Gaggle:
  --manager                  Enables distributed load test Manager mode
  --expect-workers VALUE     Sets number of Workers to expect
  --no-hash-check            Tells Manager to ignore load test checksum
  --manager-bind-host HOST   Sets host Manager listens on (default: 0.0.0.0)
  --manager-bind-port PORT   Sets port Manager listens on (default: 5115)
  --worker                   Enables distributed load test Worker mode
  --manager-host HOST        Sets host Worker connects to (default: 127.0.0.1)
  --manager-port PORT        Sets port Worker connects to (default: 5115)
```

The `examples/simple.rs` example copies the simple load test documented on the locust.io web page, rewritten in Rust for Goose. It uses minimal advanced functionality, but demonstrates how to GET and POST pages. It defines a single Task Set which has the user log in and then load a couple of pages.

Goose can make use of all available CPU cores. By default, it will launch 1 user per core, and it can be configured to launch many more. The following was configured instead to launch 1,024 users. Each user randomly pauses 5 to 15 seconds after each task is loaded, so it's possible to spin up a large number of users. Here is a snapshot of `top` when running this example on a 1-core VM with 10G of available RAM -- there were ample resources to launch considerably more "users", though `ulimit` had to be resized:

```
top - 06:56:06 up 15 days,  3:13,  2 users,  load average: 0.22, 0.10, 0.04
Tasks: 116 total,   3 running, 113 sleeping,   0 stopped,   0 zombie
%Cpu(s):  1.7 us,  0.7 sy,  0.0 ni, 96.7 id,  0.0 wa,  0.0 hi,  1.0 si,  0.0 st
MiB Mem :   9994.9 total,   7836.8 free,   1101.2 used,   1056.9 buff/cache
MiB Swap:  10237.0 total,  10237.0 free,      0.0 used.   8606.9 avail Mem 

  PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND    
 1339 goose     20   0 1235480 758292   8984 R   3.0   7.4   0:06.56 simple     
```

Here's the output of running the loadtest. The `-v` flag sends `INFO` and more critical messages to stdout (in addition to the log file). The `-u1024` tells Goose to spin up 1,024 users. The `-r32` option tells Goose to hatch 32 users per second. The `-t10m` option tells Goose to run the load test for 10 minutes, or 600 seconds. The `--status-codes` flag tells Goose to track metrics about HTTP Status codes returned by the server, in addition to the default per-task and per-request metrics. The `--no-reset-metrics` flag tells Goose to start tracking the 10m run-time from when the first user starts, instead of the default which is to flush all metrics and start timing after all users have started. And finally, the `--only-summary` flag tells Goose to only display the final metrics after the load test finishes, otherwise it would display running metrics every 15 seconds for the duration of the test.

```
$ cargo run --release --example simple -- --host http://local.dev -v -u1024 -r32 -t10m --status-codes --no-reset-metrics --only-summary
    Finished release [optimized] target(s) in 0.09s
     Running `target/release/examples/simple --host 'http://local.dev' -v -u1024 -r32 -t10m --status-codes --no-reset-metrics --only-summary`
10:55:04 [ INFO] Output verbosity level: INFO
10:55:04 [ INFO] Logfile verbosity level: INFO
10:55:04 [ INFO] Writing to log file: goose.log
10:55:04 [ INFO] run_time = 600
10:55:04 [ INFO] global host configured: http://local.dev
10:55:04 [ INFO] initializing user states...
10:55:09 [ INFO] launching user 1 from WebsiteUser...
10:55:09 [ INFO] launching user 2 from WebsiteUser...
10:55:09 [ INFO] launching user 3 from WebsiteUser...
```
...
```
10:55:42 [ INFO] launching user 1022 from WebsiteUser...
10:55:42 [ INFO] launching user 1023 from WebsiteUser...
10:55:42 [ INFO] launching user 1024 from WebsiteUser...
10:55:42 [ INFO] launched 1024 users...
All 1024 users hatched.

11:05:09 [ INFO] stopping after 600 seconds...
11:05:09 [ INFO] waiting for users to exit
11:05:09 [ INFO] exiting user 879 from WebsiteUser...
11:05:09 [ INFO] exiting user 41 from WebsiteUser...
11:05:09 [ INFO] exiting user 438 from WebsiteUser...
```
...
```
11:05:10 [ INFO] exiting user 268 from WebsiteUser...
11:05:10 [ INFO] exiting user 864 from WebsiteUser...
11:05:10 [ INFO] exiting user 55 from WebsiteUser...
11:05:11 [ INFO] printing metrics after 601 seconds...

=== PER TASK METRICS ===
------------------------------------------------------------------------------ 
 Name                    | # times run    | # fails        | task/s | fail/s
 ----------------------------------------------------------------------------- 
 1: WebsiteUser          |
   1:                    | 1,024          | 0 (0%)         | 1      | 0    
   2:                    | 28,702         | 0 (0%)         | 47     | 0    
   3:                    | 28,678         | 0 (0%)         | 47     | 0    
 ------------------------+----------------+----------------+--------+--------- 
 Aggregated              | 58,404         | 0 (0%)         | 97     | 0    
-------------------------------------------------------------------------------
 Name                    | Avg (ms)   | Min        | Max        | Median    
 ----------------------------------------------------------------------------- 
 1: WebsiteUser          |
   1:                    | 15         | 5          | 9239       | 6         
   2:                    | 21         | 1          | 1680       | 1         
   3:                    | 22         | 1          | 1703       | 1         
 ------------------------+------------+------------+------------+------------- 
 Aggregated              | 22         | 1          | 9239       | 1         

=== PER REQUEST METRICS ===
------------------------------------------------------------------------------ 
 Name                    | # reqs         | # fails        | req/s  | fail/s
 ----------------------------------------------------------------------------- 
 GET /                   | 28,702         | 0 (0%)         | 47     | 0    
 GET /about/             | 28,678         | 0 (0%)         | 47     | 47   
 POST /login             | 1,024          | 0 (0%)         | 1      | 1    
 ------------------------+----------------+----------------+--------+--------- 
 Aggregated              | 58,404         | 0 (0%)         | 97     | 49   
-------------------------------------------------------------------------------
 Name                    | Avg (ms)   | Min        | Max        | Median    
 ----------------------------------------------------------------------------- 
 GET /                   | 21         | 1          | 1680       | 1         
 GET /about/             | 22         | 2          | 1703       | 2         
 POST /login             | 15         | 5          | 9239       | 6         
 ------------------------+------------+------------+------------+------------- 
 Aggregated              | 22         | 1          | 9239       | 1         
-------------------------------------------------------------------------------
 Slowest page load within specified percentile of requests (in ms):
 ------------------------------------------------------------------------------
 Name                    | 50%    | 75%    | 98%    | 99%    | 99.9%  | 99.99%
 ----------------------------------------------------------------------------- 
 GET /                   | 1      | 1      | 390    | 1000   | 1680   |   1680
 GET /about/             | 2      | 2      | 470    | 1000   | 1703   |   1703
 POST /login             | 6      | 7      | 11     | 12     | 28     |     28
 ------------------------+--------+--------+--------+--------+--------+------- 
 Aggregated              | 1      | 1      | 420    | 1000   | 2000   |   2000
-------------------------------------------------------------------------------
 Name                    | Status codes              
 ----------------------------------------------------------------------------- 
 GET /                   | 28,702 [200]             
 GET /about/             | 28,678 [200]             
 POST /login             | 1,024 [200]              
-------------------------------------------------------------------------------
 Aggregated              | 58,404 [200]
```

## Defaults

All run-time options can be configured with custom defaults. For example, you may want to default to the the host name of your local development environment, only requiring that `--host` be set when running against a production environment. Assuming your local development environment is at "http://local.dev/" you can do this as follows:

```
    GooseAttack::initialize()?
        .register_taskset(taskset!("LoadtestTasks")
            .register_task(task!(loadtest_index))
        )
        .set_default(GooseDefault::Host, "http://local.dev/")
        .execute()?
        .print();
    
    Ok(())
```

The following defaults can be configured with a `&str`:
 o host: `GooseDefault::Host`
 o log file name: `GooseDefault::LogFile`
 o metrics log file name: `GooseDefault::MetricsFile`
 o metrics log file format: `GooseDefault::MetricsFormat`
 o debug log file name: `GooseDefault::DebugFile`
 o debug log file format: `GooseDefault::DebugFormat`
 o host to bind Manager to: `GooseDefault::ManagerBindHost`
 o host for Worker to connect to: `GooseDefault::ManagerHost`

The following defaults can be configured with a `usize` integer:
 o total users to start: `GooseDefault::Users`
 o users to start per second: `GooseDefault::HatchRate`
 o number of seconds for test to run: `GooseDefault::RunTime`
 o log level: `GooseDefault::LogLevel`
 o verbosity: `GooseDefault::Verbose`
 o maximum requests per second: `GooseDefault::ThrottleRequests`
 o number of Workers to expect: `GooseDefault::ExpectWorkers`
 o port to bind Manager to: `GooseDefault::ManagerBindPort`
 o port for Worker to connect to: `GooseDefault::ManagerPort`

The following defaults can be configured with a `bool`:
 o only print final summary metrics: `GooseDefault::OnlySummary`
 o do not reset metrics after all users start: `GooseDefault::NoResetMetrics`
 o do not track metrics: `GooseDefault::NoMetrics`
 o do not track task metrics: `GooseDefault::NoTaskMetrics`
 o track status codes: `GooseDefault::StatusCodes`
 o follow redirect of base_url: `GooseDefault::StickyFollow`
 o enable Manager mode: `GooseDefault::Manager`
 o ignore load test checksum: `GooseDefault::NoHashCheck`
 o enable Worker mode: `GooseDefault::Worker`

For example, without any run-time options the following load test would automatically run against `local.dev`, logging metrics to `goose-metrics.log` and debug to `goose-debug.log`. It will automatically launch 20 users in 4 seconds, and run the load test for 15 minutes. Metrics will only be displayed when the load test completes, and will include additional status code metrics. The order the defaults are set is not important.

```
    GooseAttack::initialize()?
        .register_taskset(taskset!("LoadtestTasks")
            .register_task(task!(loadtest_index))
        )
        .set_default(GooseDefault::Host, "local.dev")
        .set_default(GooseDefault::MetricsFile, "goose-metrics.log")
        .set_default(GooseDefault::DebugFile, "goose-debug.log")
        .set_default(GooseDefault::Users, 20)
        .set_default(GooseDefault::HatchRate, 4)
        .set_default(GooseDefault::RunTime, 900)
        .set_default(GooseDefault::OnlySummary, true)
        .set_default(GooseDefault::StatusCodes, true)
        .execute()?
        .print();
    
    Ok(())
```

## Throttling Requests

By default, Goose will generate as much load as it can. If this is not desirable, the throttle allows optionally limiting the maximum number of requests per second made during a load test. This can be helpful to ensure consistency when running a load test from multiple different servers with different available resources.

The throttle is specified as an integer. For example:

```rust
$ cargo run --example simple -- --host http://local.dev/ -u100 -r20 -v --throttle-requests 5
```

In this example, Goose will launch 100 GooseUser threads, but the throttle will prevent them from generating a combined total of more than 5 requests per second. The `--throttle-requests` command line option imposes a maximum number of requests, not a minimum number of requests.

## Logging Load Test Metrics

Goose can optionally log details about all load test requests to a file. To enable, add the `--metrics-log-file=foo` command line option, where `foo` is either a relative or absolute path of the log file to create. Any existing file that may already exist will be overwritten.

When operating in Gaggle-mode, the `--metrics-log-file` option can only be enabled on the Worker processes, configuring Goose to spread out the overhead of writing logs.

By default, logs are written in JSON Lines format. For example:

```json
{"elapsed":30,"final_url":"http://local.dev/user/42","method":"POST","name":"/login","redirected":true,"response_time":220,"status_code":200,"success":true,"update":false,"url":"http://local.dev/login","user":0}
{"elapsed":251,"final_url":"http://local.dev/","method":"GET","name":"/","redirected":false,"response_time":3,"status_code":200,"success":true,"update":false,"url":"http://local.dev/","user":0}
{"elapsed":1027,"final_url":"http://local.dev/user/13","method":"POST","name":"/login","redirected":true,"response_time":266,"status_code":200,"success":true,"update":false,"url":"http://local.dev/login","user":1}
{"elapsed":1294,"final_url":"http://local.dev/","method":"GET","name":"/","redirected":false,"response_time":4,"status_code":200,"success":true,"update":false,"url":"http://local.dev/","user":1}
```

Logs include the entire `GooseRawRequest` object as defined in `src/goose.rs`, which are created on all requests. This object includes the following fields:
 - `elapsed`: total milliseconds between when the `GooseUser` thread started and this
   request was made;
 - `method`: the type of HTTP request made;
 - `name`: the name of the request;
 - `url`: the URL that was requested;
 - `final_url`: the URL that was returned (may be different if the request was redirected);
 - `redirected`: true or false if the request was redirected;
 - `response_time`: how many milliseconds the request took;
 - `status_code`: the HTTP response code returned for this request;
 - `success`: true or false if this was a successful request;
 - `update`: true or false if this is a recurrence of a previous log entry, but with
   `success` toggling between `true` and `false`. This happens when a load test calls
   `set_success()` on a request that Goose previously interpreted as a failure, or
   `set_failure()` on a request previously interpreted as a success;
 - `user`: an integer value indicating which `GooseUser` thread made this request.

In the first line of the above example, `GooseUser` thread 0 made a `POST` request to `/login` and was successfully redirected to `/user/42` in 220 milliseconds. The second line is the same `GooseUser` thread which then made a `GET` request to `/` in 3 milliseconds. The third and fourth lines are a second `GooseUser` thread doing the same thing, first logging in and then loading the front page.

By default Goose logs metrics in JSON Lines format. The `--metrics-log-format` option can be used to log in `csv`, `json` or `raw` format. The `raw` format is Rust's debug output of the entire `GooseRawRequest` object.

For example, `csv` output of the same requests logged above would look like:
```csv
elapsed,method,name,url,final_url,redirected,response_time,status_code,success,update,user
30,POST,"/login","http://local.dev/login","http://local.dev/user/42",true,30,200,true,false,0
251,GET,"/","http://local.dev/","http://local.dev/",false,3,200,true,false,0
1027,POST,"/login","http://local.dev/login","http://local.dev/user/13",true,266,200,true,false,1
1294,GET,"/","http://local.dev/","http://local.dev/",false,4,200,true,false,1
```

## Load Test Debug Logging

Goose can optionally log details about requests and responses for debug purposes. When writing a load test you must invoke `client.log_debug(tag, Option<request>, Option<headers>, Option<body>)` where `tag` is an arbitrary string to identify where in the load test and/or why debug is being written, `request` is a `GooseRawRequest` object, `headers` are the HTTP headers returned by the server, and `body` is the web page body returned by the server.

For an example on how to correctly use `client.log_debug()`, including how to obtain the response headers and body, see `examples/drupal_loadtest`.

If the load test is run with the `--debug-log-file=foo` command line option, where `foo` is either a relative or an absolute path, Goose will log all debug generated by calls to `client.log_debug()` to this file. Debug is logged in JSON Lines format. For example:

```json
{"body":"<!DOCTYPE html>\n<html>\n  <head>\n    <title>503 Backend fetch failed</title>\n  </head>\n  <body>\n    <h1>Error 503 Backend fetch failed</h1>\n    <p>Backend fetch failed</p>\n    <h3>Guru Meditation:</h3>\n    <p>XID: 923425</p>\n    <hr>\n    <p>Varnish cache server</p>\n  </body>\n</html>\n","header":"{\"date\": \"Wed, 01 Jul 2020 10:27:31 GMT\", \"server\": \"Varnish\", \"content-type\": \"text/html; charset=utf-8\", \"retry-after\": \"5\", \"x-varnish\": \"923424\", \"age\": \"0\", \"via\": \"1.1 varnish (Varnish/6.1)\", \"x-varnish-cache\": \"MISS\", \"x-varnish-cookie\": \"SESSd7e04cba6a8ba148c966860632ef3636=hejsW1mQnnsHlua0AicCjEpUjnCRTkOLubwL33UJXRU\", \"content-length\": \"283\", \"connection\": \"keep-alive\"}","request":{"elapsed":4192,"final_url":"http://local.dev/node/3247","method":"GET","name":"(Auth) comment form","redirected":false,"response_time":8,"status_code":503,"success":false,"update":false,"url":"http://local.dev/node/3247","user":4},"tag":"post_comment: no form_build_id found on node/3247"}
```

If `--debug-log-file=foo` is not specified at run time, nothing will be logged.

By default Goose writes debug logs in JSON Lines format. The `--debug-log-format` option can be used to log in `json` or `raw` format. The `raw` format is Rust's debug output of the entire `GooseDebug` object.

## Gaggle: Distributed Load Test

Goose also supports distributed load testing. A Gaggle is one Goose process running in Manager mode, and 1 or more Goose processes running in Worker mode. The Manager coordinates starting and stopping the Workers, and collects aggregated metrics. Gaggle support is a cargo feature that must be enabled at compile-time as documented below. To launch a Gaggle, you must copy your load test application to all servers from which you wish to generate load.

It is strongly recommended that the same load test application be copied to all servers involved in a Gaggle. By default, Goose will verify that the load test is identical by comparing a hash of all load test rules. Telling it to skip this check can cause the load test to panic (for example, if a Worker defines a different number of tasks or task sets than the Manager).

### Gaggle Compile-time Feature

Gaggle support is a compile-time Cargo feature that must be enabled. Goose uses the [`nng`](https://docs.rs/nng/) library to manage network connections, and compiling `nng` requires that `cmake` be available.

The `gaggle` feature can be enabled from the command line by adding `--features gaggle` to your cargo command.

When writing load test applications, you can default to compiling in the Gaggle feature in the `dependencies` section of your `Cargo.toml`, for example:

```toml
[dependencies]
goose = { version = "^0.9", features = ["gaggle"] }
```

### Gaggle Manager

To launch a Gaggle, you first must start a Goose application in Manager mode. All configuration happens in the Manager. To start, add the `--manager` flag and the `--expect-workers` flag, the latter necessary to tell the Manager process how many Worker processes it will be coordinating. For example:

```
cargo run --features gaggle --example simple -- --manager --expect-workers 2 --host http://local.dev/ -v
```

This configures a Goose Manager to listen on all interfaces on the default port (0.0.0.0:5115) for 2 Goose Worker processes.

### Gaggle Worker

At this time, a Goose process can be either a Manager or a Worker, not both. Therefor, it usually makes sense to launch your first Worker on the same server that the Manager is running on. If not otherwise configured, a Goose Worker will try to connect to the Manager on the localhost. This can be done as follows:

```
cargo run --features gaggle --example simple -- --worker -v
```

In our above example, we expected 2 Workers. The second Goose process should be started on a different server. This will require telling it the host where the Goose Manager process is running. For example:

```
cargo run --example simple -- --worker --manager-host 192.168.1.55 -v
```

Once all expected Workers are running, the distributed load test will automatically start. We set the `-v` flag so Goose provides verbose output indicating what is happening. In our example, the load test will run until it is canceled. You can cancel the Manager or either of the Worker processes, and the test will stop on all servers.

### Gaggle Run-time Flags

* `--manager`: starts a Goose process in Manager mode. There currently can only be one Manager per Gaggle.
* `--worker`: starts a Goose process in Worker mode. How many Workers are in a given Gaggle is defined by the `--expect-workers` option, documented below.
* `--no-hash-check`: tells Goose to ignore if the load test application doesn't match between Worker(s) and the Manager. This is not recommended, and can cause the application to panic.

The `--no-metrics`, `--only-summary`, `--no-reset-metrics`, `--status-codes`, and `--no-hash-check` flags must be set on the Manager. Workers inherit these flags from the Manager

### Gaggle Run-time Options

* `--manager-bind-host <manager-bind-host>`: configures the host that the Manager listens on. By default Goose will listen on all interfaces, or `0.0.0.0`.
* `--manager-bind-port <manager-bind-port>`: configures the port that the Manager listens on. By default Goose will listen on port `5115`.
* `--manager-host <manager-host>`: configures the host that the Worker will talk to the Manager on. By default, a Goose Worker will connect to the localhost, or `127.0.0.1`. In a distributed load test, this must be set to the IP of the Goose Manager.
* `--manager-port <manager-port>`: configures the port that a Worker will talk to the Manager on. By default, a Goose Worker will connect to port `5115`.

The `--users`, `--hatch-rate`, `--host`, and `--run-time` options must be set on the Manager. Workers inherit these options from the Manager.

The `--throttle-requests` option must be configured on each Worker, and can be set to a different value on each Worker if desired.

### Technical Details

Goose uses [`nng`](https://docs.rs/nng/) to send network messages between the Manager and all Workers. [Serde](https://docs.serde.rs/serde/index.html) and [Serde CBOR](https://github.com/pyfisch/cbor) are used to serialize messages into [Concise Binary Object Representation](https://tools.ietf.org/html/rfc7049).

Workers initiate all network connections, and push metrics to the Manager process.

## RustLS

By default Reqwest (and therefore Goose) uses the system-native transport layer security to make HTTPS requests. This means `schannel` on Windows, `Security-Framework` on macOS, and `OpenSSL` on Linux. If you'd prefer to use a [pure Rust TLS implementation](https://github.com/ctz/rustls), disable default features and enable `rustls` in `Cargo.toml` as follows:

```toml
[dependencies]
goose = { version = "^0.9", default-features = false, features = ["rustls"] }
```
