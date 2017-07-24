# Queue for Kirby

This plugin adds a basic queue to the Kirby CMS, using Cron and Kirby's flat file system.

## Installation

Copy the files of this repo to your `site/plugins` folder. You can also run `kirby plugin:install sebsel/queue-for-kirby` if you have the [CLI tools](https://github.com/getkirby/cli) installed.

You then need to add `site/plugins/queue-for-kirby/worker.php` to your [Cron Jobs](https://en.wikipedia.org/wiki/Cron) or similar software, for once per minute, depending on how fast you want your jobs done. You can even move this file if you want, but make sure to change the `require` path to point at `kirby/bootstrap.php`, so it can load Kirby.

The plugin will try to create the folder `site/queue` and some files and folders within it.

## How to define jobs

You need to define the following things within the base-file of your plugin, not in any lazy loaded classes (with Kirby's `load()` or composer's autoloading). Just put it in `site/plugins/your_plugin/your_plugin.php`.

```php
// Make sure that the Queue plugin is loaded.
kirby()->plugin('queue-for-kirby');
if(!class_exists('queue')) throw new Exception('This plugin requires the Queue for Kirby plugin');

// Define a job by giving it a name and an action
queue::define('send_webmention', function($data) {

    // For example: send a webmention!
    $endpoint = discover_webmention_endpoint($data['target']);

    $r = remote::post($endpoint, ['data' => [
        'target' => $data['target'],
        'source' => $data['source']
    ]]);

    // But it could be anything ofcourse

    if($r->code != 201 and $r->code != 202) {
        // Return false or throw an exception to fail the job
        return false;
    }
});
```

Then, anywhere else in your plugin code, you can call `queue::add()` to schedule a new job. This will also work in the lazy loaded classes, if you added the previous step to your plugin base-file.

```php
// Schedule a job by giving a job-name and the data you need
queue::add('send_webmention', [
    'target' => $target,
    'source' => $source
]);
```

The data you pass in is up to you: you can define your own jobs. The only caveat is that the data needs to be stored in YAML. To access a `Page` object, you can use:

```php
queue::define('some_page_action', function($data) {
    $page = page($data['page']);
});

// and

queue::add('some_page_action', [
    'page' => $page->id()
]);
```

Do not forget to add `site/plugins/queue-for-kirby/worker.php` to a Cron Job, or your jobs will sit in the queue forever!

## Failed jobs

Failed jobs are currently added to `site/queue/.failed`, with some information about the error attached. They will not be retried automatically. To trigger them again, just move them one folder down to `site/queue` and make sure the worker runs.

To fail a job, either throw an exception or return false.

## Available methods

The following static methods are available on this class:

### queue::define($name, $action)

Define a new action that should be executed once the job is handled by the worker.

```php
queue::define('job_name', function($data) {
    // Do something
});
```

### queue::add($name, $data)

Add a new job to the queue. The data you pass in can be anything, as long as it can be stored in YAML.

```php
queue::add('job_name', [
    'param' => 'some data'
]);
```

### queue::jobs()

Returns an array of jobs, which are associated arrays with `added`, `name` and `data`.

Doing something with these jobs does **not** change the queue. Only `queue::work()` removes jobs from the queue.

```php
queue::jobs();
// Returns, for example:
[
    [
        'added' => '2001-01-01T01:01:01+00:00',
        'name' => 'job_name',
        'data' => [
            'param' => 'some data'
        ]
    ],
    [
        'added' => '2001-01-01T01:01:02+00:00',
        'name' => 'another_job',
        'data' => 'some string'
    ]
]
```

### queue::failedJobs()

Returns an array of failed jobs, which are associated arrays with `error` and `tried`, in addition to a normal job's `added`, `name` and `data`.

```php
queue::jobs();
// Returns, for example:
[
    [
        'added' => '2001-01-01T01:01:01+00:00',
        'name' => 'job_name',
        'data' => [
            'param' => 'some data'
        ],
        'error' => 'Job returned false',
        'tried' => '2001-01-01T01:01:02+00:00'
    ]
]
```

### queue::work()

Executes the first job in the queue. Don't call this one outside of `worker.php`, because that would defeat the purpose of the queue.

### queue::hasJobs()

Returns `true` or `false`, depending on wether there are jobs in the queue.

### queue::flush()

Removes all jobs from the queue, **including** failed jobs.

### queue::path()

Returns the full path of `site/queue`.

### queue::failedPath()

Returns the full path of `site/queue/.failed`.
