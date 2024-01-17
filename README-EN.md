Communication between microservices using Redis and WebSocket

This guide provides instructions on how to communicate between microservices using Redis and WebSocket. This method is simple but efficient and is suitable for sending and receiving messages between applications without using HTTP.
What is a microservice?

The concept of a microservice refers to the idea of managing a large system by dividing it into smaller, independent components, typically called microservices. These microservices run independently and have their own data, dependency layer, and business logic.

There are a number of ways to communicate between microservices internally without using HTTP, such as:

    Message queue (MQ): In this approach, microservices communicate with each other through a message queue. Each microservice can send a message to the queue and the other microservice can read and process it. This communication method is well-suited for managing asynchronous connections.
    Event-driven architecture (EDA): In this case, microservices communicate with each other by sending and receiving events. Each microservice can listen for the occurrence of an event and react to it. This approach is typically implemented using a messaging system.
    RPC (Remote Procedure Call): In this case, microservices are called as remote procedures for each other. This method involves using protocols such as Protocol Buffers or Apache Thrift.
    Database replication: It is possible that some microservices may directly access other microservices and extract data from a shared database. This approach should be managed carefully to ensure proper coordination and security settings.

Each of these methods has its own advantages and disadvantages, and the best approach depends on the needs and specifications of your specific project.
Installation and setup
Prerequisites

    Node.js
    Laravel
    Redis

Step 1: Start Redis

It is recommended to start Redis first. You can download and install the version you want from the official Redis website: https://redis.io/download.
Step 2: Start LaravelService in Node.js

    Go to the Node.js project folder you want to use.
    Make sure Redis is installed.
    Add the LaravelService.js file to your project.
    Use the LaravelService class to communicate with Laravel.

```JavaScript

const EventEmitter = require("events");
const Redis = require("ioredis");
const Sub = new Redis();
const Pub = new Redis();
const { promisify } = require("util");
const uuid = require("uuid");
class LaravelService extends EventEmitter {
  constructor(action, data) {
    super();
    console.log("new LaravelService ...");
    this.response = null;
    this.channelName =
      process.env.LARAVEL_REDIS_CHANNEL ?? "laravel_database_general";
    this.data = data;
    this.action = action ?? "subscribed";
    this.configs = {};
    // Subscribe to Redis channel
    Sub.subscribe(this.channelName, (err) => {
      if (err) {
        console.error("Subscription error:", err);
        return;
      }

      // Subscribed to channel: ${this.channelName}
    });

    // Event for receiving data from Redis channel
    Sub.on("message", (channel, message) => {
      this.parseResponse(message);
    });
    this.call(this.action, this.data);
  }

  async call2(action, data) {
    await Pub.publish(
      this.channelName + "_sender",
      JSON.stringify({ action, data, source: "nodejs" })
    );
  }

  call(action, data) {
    return new Promise((resolve, reject) => {
      const requestId = uuid.v4(); // Create unique ID
      const requestData = { action, data, requestId, source: "nodejs" };
      const requestChannel = this.channelName + "_sender";

      const responseCallback = (channel, message) => {
        // console.log("Response received:", message);

        const response = this.parseResponse(message);
        if (response.requestId === requestId) {
          // Check for unique ID match
          Sub.removeListener("message", responseCallback);
          resolve({ response, channel });
        }
      };
      Sub.on("message", responseCallback);
Pub.publish(requestChannel, JSON.stringify(requestData), (err) => {
                if (err) {
                    reject(err);
                }
            });

            // تنظیم timeout برای 5 ثانیه
            const timeoutDuration = 5000; // 5 second
            setTimeout(() => {
                Sub.removeListener("message", responseCallback);
                reject(new Error("Timeout exceeded")); // send timeout expection
            }, timeoutDuration);
        });
    }

    parseResponse(message) {
        const data = JSON.parse(message);
        if (data && data.source && data.source !== "nodejs") {
            this.response = data;
            if (data.action == "configs") this.configs = data;
            this.emit("message", data);
        }
        return data;
    }
}

module.exports = LaravelService;
```
Step 3: Start SubscribeToGeneralChannel in Laravel

  Create a new Artisan command:

    php artisan make:command SubscribeToGeneralChannel

  Edit the created command file (SubscribeToGeneralChannel.php) and add the following code:

```PHP

// ...
use Illuminate\Support\Facades\Redis;

class SubscribeToGeneralChannel extends Command
{
  // ...

  public function handle()
  {
    Redis::connection('pubsub')->psubscribe(['*_sender'], function ($message) {
      // Process message received from Node.js
      echo $message;

      $messageArray = json_decode($message, true);

      if ($messageArray && isset($messageArray["action"])) {
        // Perform actions related to requests
        $this->performActions($messageArray);
      }
    });
  }

  private function performActions($messageArray)
  {
    $response = ["source" => "laravel", "action" => $messageArray["action"], "requestId" => $messageArray["requestId"] ?? null];

    switch ($messageArray["action"]) {
      case "configs":
        // Perform actions related to "configs" request
        $mainConfig = new MainConfigsController();
        $configs = $mainConfig->index(true);
        $response = [...$configs, ...$response];
        Redis::connection('default')->publish('general', json_encode($response));
        break;
      // Actions related to other requests
      // ...
    }
  }
}
```
3.Run the command to start listening to the channel:

    php artisan redis:subscribe-general

Usage

Now, by running the commands and adding actions related to requests, your microservices will be able to exchange information and commands between each other.

For example, sending a command from Node.js to Laravel:
JavaScript

    const LaravelService = require("./path/to/LaravelService.js");
    const LaravelServiceInstance = new LaravelService("commandName", { foo: "bar" });
Note that you need to change the path of the ```LaravelService.js``` file and other items to the appropriate real location of your project.

Also, how to send a command from Laravel to nodejs:
```PHP

$resp = ["source" => "laravel", "action" => "configs", "requestId" => null, ...$configs];
Redis::connection('default')->publish('general', json_encode($resp));
```
Contribution

If the explanations or scripts need improvement, we would be happy to help with this project. Please submit issue reports or merge requests. If you got it working, please give me a star.

License

MIT License

Additional notes

The ```LaravelService``` class in Node.js uses a publish/subscribe pattern to communicate with Laravel. The ```SubscribeToGeneralChannel``` command in Laravel uses a pub/sub pattern to listen to messages from Node.js.

The ```call()``` method in the LaravelService class is used to send a command to Laravel. The ```call2()``` method is a simplified version of the call() method that does not require a request ID.

The ```performActions()``` method in the ```SubscribeToGeneralChannel``` class is used to handle requests from Node.js.

The ```configs``` action is used to send a list of configurations from Laravel to Node.js. The ```mainConfig``` controller is used to get the list of configurations.

You can customize the communication between your microservices by adding your own actions to the ```performActions()``` method.
