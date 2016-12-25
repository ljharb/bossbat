# bossman

Distributed job scheduling in node, backed by redis.

Bossman combines schedulers and workers into a single concept, which aligns better with most node applications.
All of the jobs run with a [redis lock](https://redis.io/topics/distlock), preventing more than once instance from performing a given job at a time.

## Usage


```shell
npm install bossman --save
```

Or if you use Yarn in your project:

```shell
yarn add bossman
```

You can then import the module:

```js
import Bossman from 'bossman';

// Make sure you name your variables clever things:
const fred = new Bossman({
  connection: {
    host: '127.0.0.1',
    port: 1234,
  },
  // Set the redis key prefix:
  prefix: 'bossman:',
});

// Hire for a job.
fred.hire('engineers', {
  every: '10 minutes',
  work: () => {
    // Do something...
  },
});

// You can also "qa" work:
fred.qa((jobName, jobDefinition, next) => {
  return newrelic.createBackgroundTransaction(`job:${jobName}`, () => {
    const response = next();

    const end = () => {
      newrelic.endTransaction();
    };

    return response.then(end, end);
  })();
});

// Fire a job, this will cancel any pending jobs:
fred.fire('engineers');

// Shut down our instance. This does not stop any already-scheduled work from firing in other instances.
fred.quit();
```

## How it works

When you `hire` for a new job, Bossman sets a key in Redis that expire when the first run should occur.
When the key expires, Bossman does the following:

1. Attempt to get a lock to perform the work. Only one instance of bossman will acquire the lock.
  1. If the lock is acquired, then perform the work and move on.
  2. If the lock is not acquired, then move on.
2. Schedule the next run of the job by setting a key that expires when the next run should occur.

It's worth noting that every instance of Bossman attempts to schedule the next run of the job. This is okay because Redis will only schedule the first one that it receives, thanks to the `NX` option in `SET`.