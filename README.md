## Docker container pool

A Node.JS module to manage a pool of docker containers.


### Usage

```javascript

Pool = require('docker-pool')

var pool = new Pool({

  // name of pool
  name: 'my pool',

  // maximum number of containers to create at any given time
  max: 1,

  // minimum number of containers to keep in pool at any given time
  min: 1,

  // boolean that specifies whether idle containers at or below the min threshold
  // should be destroyed/re-created
  refreshIdle: true,

  // max milliseconds a container can go unused before it should be destroyed
  idleTimeoutMillis: 30000,

  // frequency to check for idle containers
  reapIntervalMillis: 1000,

  // int between 1 and x - if set, acquirers can specify their
  // relative priority in the queue if no containers are available
  priorityRange: 1,

  // int number of containers that should be created concurrently - this setting
  // throttles requests to the docker daemon for container creation
  createConcurrency: 1,

  // int number of containers that can by disposed of concurrently - this setting
  // throttles requests to the docker daemon for container stop and remove
  disposeConcurrency: 1,

  // int milliseconds after container start that Docker Pool should
  // wait before returning the container in the acquire callback - this setting
  // allows some time for the docker container process to start
  readyCheck: function(container, callback) {
    // perform test on container to ensure that it is ready to be used,
    // such as connecting to a port. then invoke the callback to indicate
    // readiness. if ready == false, use the retry interval so the pool
    // can test readiness again later. If error != null, then retries will stop and
    // the container acquisition will fail.

    callback(null /* error */, true /* ready? */, 0 /* retry interval in milliseconds */);
  },

  // options for containers
  container: {

    // every container will run using this image (required)
    image: 'ubuntu',

    // string command to run in the container (required)
    command: 'ncat -e /bin/cat -k -t -l 5555',

    // array of volume paths (optional)
    volumes: [ '/container/path/to/expose' ],

    // object of volume paths to bind to host filesystem (optional)
    binds: {
      '/host/path/to/bind': '/container/path/to/expose'
    },

    // ports to expose on the container
    ports: [ 5555 ],

    // RAM allowed to container in bytes (optional, default 10MB)
    ramBytes: 10485760,

    // share of CPU given to container relative to all other containers (optional, default 1)
    cpuShare: 1,

    // int seconds to be allowed on stop before container will be killed (optional, default 10)
    stopTimeout: 10
  }
});


// function that can be used to check a container to determine whether it is OK to use,
// or if it should be destroyed. This function is called by acquire() before returning a
// container from the pool.

pool.validate = function(container) {
  // containers older than 60s should not be used
  return new Date() - container.startedAt > 60000;
};


// use the acquire function to use a container in the pool. containers that have been
// acquired cannot be re-acquired until after they are released.
pool.acquire(function(err, container) {
  if (err)
    throw new Error(err);

  // use the container however you want
  console.log('Connect to ncat running in this container at ' + container.ip + ':5555');

  // then release the container back to the pool
  pool.release(container);
});


process.on('SIGINT', function() {
  // to clean up your pool's containers, use the drain function
  // this will stop and remove all containers created by the pool
  pool.drain(function() {
    exit();
  });
});

```


### Testing

Docker cannot be run on Mac OS X. For this reason, you must use Vagrant which runs an instance of Ubuntu under a VM.

* Use `vagrant up` to build provision and start the Vagrant instance
* Use `vagrant reload` to reboot the Vagrant instance (so that the Linux 3.8 kernel will be loaded)
* SSH to vagrant using `vagrant ssh`
* Change to the directory containing this module using `cd /vagrant`
* Install dependencies using `npm install`
* Run the tests using `npm test`
* After the tests run, there should be no containers listed with `docker ps -a`
* To stop docker containers running in the vagrant instance, use `docker stop $(docker ps -a -q)`
* To remove docker containers running in the vagrant instance, use `docker rm $(docker ps -a -q)`