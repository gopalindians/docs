# GRPC — Client SDK

In the previous part of this [article](./service.md), we showed you how to create a gRPC service `Pinger` in PHP with
Spiral and `spiral/roadrunner-bridge` package. In this part, we will show you how to create a client service
`Monitor` for communication with `Pinger` service.

## PHP client

By following these steps, you will be able to create a client SDK for the Pinger service that can be easily used in your
PHP applications. This will make it easier to communicate with the service and integrate it into your codebase.

> **Note**
> Here you can find the [installation instructions](./configuration.md) for the `grpc` PHP extension, `protoc` compiler
> and the `protoc-gen-php-grpc` plugin.

### 1. Generate PHP classes from the `.proto` file

To create the client SDK, we will use the PHP classes generated from the `.proto` file in the
[previous part](./service.md). You can easily generate these classes following the instructions in the previous part.

### 2. Create a client class

We will create a class that implements the `generated/GRPC/PingerInterface` interface and provides a simple interface for
calling the service's methods.

```php
namespace App\Service;

use GRPC\Pinger;
use Spiral\Core\CoreInterface;
use Spiral\RoadRunner\GRPC;

final class PingerClient implements Pinger\PingerInterface extends \Grpc\BaseStub
{
    public function ping(GRPC\ContextInterface $ctx, Pinger\PingRequest $in): Pinger\PingResponse
    {
        [$response, $status] = $this->_simpleRequest(
            '/' . self::NAME . '/ping',
            $in,
            [Pinger\PingResponse::class, 'decode'],
            (array) $ctx->getValue('metadata'),
            (array) $ctx->getValue('options')
        )->wait();

        return $response;
    }
}
```

### 3. Register the client class in the container

To use the client class in your application, you will need to register it in the container. You can do this in the
Application bootloader:

```php app/src/Application/Bootloader/AppBootloader.php
namespace App\Application\Bootloader;

use App\Service\PingerClient;
use GRPC\Pinger\PingerInterface;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Boot\EnvironmentInterface;

final class AppBootloader extends Bootloader
{
    protected const SINGLETONS = [
        PingerInterface::class => [self::class, 'initPingService'],
    ];

    private function initPingService(
        EnvironmentInterface $env
    ): PingerInterface {
        return new PingerClient(
            $env->get('PING_SERVICE_HOST', '127.0.0.1:9001'),
            ['credentials' => \Grpc\ChannelCredentials::createInsecure()],
        );
    }
}
```

And add the bootloader to the list of bootloaders:

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \App\Application\Bootloader\AppBootloader::class,
];
```

Now the client class is registered as a singleton in the container.

### 4. Client usage

Finally, you can inject the client class into your code and use it to call the Pinger service.

Here is an example of how you can use the `PingerClient`:

```php app/src/Endpoint/Console/PingServiceCommand.php
namespace App\Endpoint\Console;

use Spiral\Console\Attribute\Argument;
use Spiral\Console\Attribute\AsCommand;
use Spiral\Console\Attribute\Question;
use GRPC\Pinger\PingerInterface;
use Spiral\Console\Command;
use GRPC\Pinger\PingRequest;
use Spiral\RoadRunner\GRPC\Context;
use Spiral\RoadRunner\GRPC\Exception\GRPCException;

#[AsCommand(name: 'ping')]
final class PingCommand extends Command
{
    #[Argument(description: 'URL to ping')]
    #[Question(question: 'Provide URL to ping')]
    private string $url;

    public function __invoke(
        PingerInterface $client
    ): int {
        try {

            $this->writeln(\sprintf('Sending ping request [%s]...', $this->url));

            $response = $client->ping(
                new Context([]),
                new PingRequest(['url' => $this->url])
            );

            $this->writeln(\sprintf(
                'Response: code - %d',
                $response->getStatusCode()
            ));

        } catch (GRPCException $e) {

            $this->writeln(\sprintf(
                'Error: code - %d, message - %s',
                $e->getCode(),
                $e->getMessage()
            ));

        }

        return self::SUCCESS;
    }
}
```

To use this command, run it from the command line:

```terminal
php app.php ping https://google.com
```

This will call the Pinger service and print the HTTP status code of the response.

## Golang Clients

GRPC allows you to create a client SDK in any supported language. To generate client on Golang, install the GRPC toolkit
first:

### 1. Install the necessary dependencies

To use gRPC in Go, you will need to install the necessary dependencies. You can do this using the Go package manager:

```bash
go get -u google.golang.org/grpc
go get -u github.com/golang/protobuf/protoc-gen-go
```

> **See more**
> Read more about how to create Golang GRPC clients and server [here](https://grpc.io/docs/tutorials/basic/go/).

### 2. Compile the `.proto` file

Next, you will need to compile the `.proto` file into Go code. You can do this using the protoc compiler and the Go
plugin:

```bash
protoc -I proto/ proto/pinger.proto --go_out=plugins=grpc:pinger
```

This will generate a` pinger.pb.go` file, which contains the Go classes for the service and messages defined in the
`.proto` file.

> **Note**
> Notice the `package` name in `pinger.proto`.

### 3. Create the client

Now you can create a client in Go for the Pinger service. Here is an example of a client that calls the `ping()` method 
of the Pinger service and prints the HTTP status code to the console:

```golang
package main

import (
	"context"
	"fmt"
	"log"

	"google.golang.org/grpc"
	"pinger"
)

func main() {
	// Set up a connection to the server.
	conn, err := grpc.Dial("127.0.0.1:9001", grpc.WithInsecure())
	if err != nil {
		log.Fatalf("did not connect: %v", err)
	}
	
	defer conn.Close()

	client := pinger.NewPingerClient(conn)

	// Call the ping method.
	response, err := client.Ping(context.Background(), &pinger.PingRequest{
		Url: "https://google.com",
	})

	if err != nil {
		log.Fatalf("error calling ping: %v", err)
	}

	// Print the HTTP status code.
	fmt.Println(response.StatusCode)
}
```

You can run this client using the Go command:

```bash
go run main.go
```

### Passing Metadata

To pass metadata between server and client:

```golang
package main

import (
	"context"
	"fmt"
	"log"

	"google.golang.org/grpc"
	"google.golang.org/grpc/metadata"
	"pinger"
)

func main() {
	// Set up a connection to the server.
	conn, err := grpc.Dial("127.0.0.1:9001", grpc.WithInsecure())
	if err != nil {
		log.Fatalf("did not connect: %v", err)
	}
	
	defer conn.Close()

	client := pinger.NewPingerClient(conn)
	
	// attach value to the server
	ctx := metadata.AppendToOutgoingContext(context.Background(), "client-key", "client-value")

	var header metadata.MD
	
	// Call the ping method.
	response, err := client.Ping(ctx, &pinger.PingRequest{
		Url: "https://google.com",
	}, grpc.Header(&header))

	if err != nil {
		log.Fatalf("error calling ping: %v", err)
	}

	// Print the HTTP status code.
	fmt.Println(response.StatusCode)
}
```

> **See more**
> Read more about working with metadata in
> Golang [here](https://github.com/grpc/grpc-go/blob/master/Documentation/grpc-metadata.md).
