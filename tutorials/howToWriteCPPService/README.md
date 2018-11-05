# Tutorial - How to write a C++ gRPC service

-------------------------------

_Before following this tutorial, make sure you've installed_

* _Docker (https://www.docker.com/)_
* _Metamask (https://metamask.io)_

_You will need a private-public key pair to register your service in SNET. Generate them in Metamask before you start this turorial._

-------------------------------

Run this tutorial from a bash terminal in this git repository folder. For more
details regarding C++ gRPC see https://grpc.io/docs/

In this tutorial we'll create a C++ gRPC service and publish it in SingularityNET.

## Step 1 

Setup and run a docker container. We'll install C++ gRPC stuff in a container
because of this warning from the authors: 

```
"WARNING: After installing with make install there is no easy way to uninstall,
which can cause issues if you later want to remove the grpc and/or protobuf
installation or upgrade to a newer version."
```

If you want to install C++ gRPC in your workstation, look for the section "C++ gRPC" in
'setupContainer.sh' or follow the instructions in https://github.com/grpc/grpc/blob/master/BUILDING.md

In this tutorial we'll develop our service inside the docker container.

```SH
$ ./setupContainer.sh
```

From this point we follow the turorial in the Docker container's prompt.

## Step 2

Create the skeleton structure for your service's project

```SH
$ ./create_project.sh PROJECT_NAME SERVICE_NAME SERVICE_PORT
```

`PROJECT_NAME` is a short tag for your project. It will be used to name
project's directory and as a namespace tag in the .proto file.

`SERVICE_NAME` is...

`SERVICE_PORT` is the port number (in localhost) the service will listen to.

`create_project.sh` will create a directory named `PROJECT_NAME` with a basic
empty implementation of the service.

In this tutorial we'll implement a service with two methods:

* int div(int a, int b)
* string check(int a)

So we'll use this command line to create project's skeleton

```SH
$ ./create_project.sh tutorial math-operations 70468
$ cd tutorial
```

## Step 3

Now we'll customize the skeleton code to actually implement our basic service.
We need to edit `src/service_spec/tutorial.proto` and define

* the data structures used to carry input and output of the methods, and
* the RPC API of the service.

`src/service_spec/tutorial.proto` have two sections we are interested in. First, let's define
the messages, which are the data structures used as input and output of the RPC methods.

```Java
message IntPair {
    int32 a = 1;
    int32 b = 2;
}

message SingleInt {
    int32 v = 1;
}

message SingleString {
    string s = 1;
}
```

Now we define the API:

```C++
service ServiceDefinition {
    rpc div(IntPair) returns (SingleInt) {}
    rpc check(SingleInt) returns (SingleString) {}
}
```

Take a look at https://developers.google.com/protocol-buffers/docs/overview to
understand everything you can do in the `.proto` file.

## Step 4

In order to actually implement our API we need to edit `src/server.cc`.

Look for `PROTO_TYPES` and replace the `using` statements to reflect our data
types defined in the last step.

```C++
using tutorial::ServiceDefinition;
using tutorial::IntPair;
using tutorial::SingleInt;
using tutorial::SingleString;
```


Now look for `SERVICE_API` and replace `doSomething()` by our actual API methods:

```C++
Status div(ServerContext* context, const IntPair* input, SingleInt* output) override {
    output->set_v(input->a() / input->b());
    return Status::OK;
}

Status check(ServerContext* context, const SingleInt* input, SingleString* output) override {
    if (input->v() != 0) {
        output->set_s("OK");
    } else {
        output->set_s("NOK");
    }
    return Status::OK;
}
```
## Step 5

Now we'll write a client to test our server locally (without using the
blockchain). Edit `src/client.cc`.

Look for `PROTO_TYPES` and replace the `using` statements to reflect our data
types defined in the last step.

```C++
using tutorial::ServiceDefinition;
using tutorial::IntPair;
using tutorial::SingleInt;
using tutorial::SingleString;
```

Now look for `TEST_CODE` and Replace `doSomething()` implementation by our
testing code:


```C++
void doSomething(int argc, char** argv) {

    int n1 = atoi(argv[1]);
    int n2 = atoi(argv[2]);

    ClientContext context1;
    SingleInt divisor;
    SingleString checkDivisor;
    divisor.set_v(n2);
    Status status1 = stub_->check(&context1, divisor, &checkDivisor);
    if (! status1.ok()) { 
        std::cout << "doSomething rpc failed." << std::endl;
        return;
    }
    if (checkDivisor.s() != "OK") {
        std::cout << "Check failed." << std::endl;
        return;
    }

    ClientContext context2;
    IntPair input;
    SingleInt result;
    input.set_a(n1);
    input.set_b(n2);
    Status status2 = stub_->div(&context2, input, &result);
    if (status2.ok()) { 
        std::cout << result.v() << std::endl;
    } else {
        std::cout << "doSomething rpc failed." << std::endl;
    }
}
```

## Step 6

To build the service:

```SH
$ cd src
$ make
$ cd ..
```

At this point you should have `server` and `client` in `bin/`

## Step 7

To test our server locally (without using the blockchain)

```SH
$ ./bin/server &
$ ./bin/client 12 4
```

You should have something like the following output:

```
root@1eee79873d63:~/install/tutorial# ./bin/server &
[1] 4217
root@1eee79873d63:~/install/tutorial# Server listening on 0.0.0.0:70468
./bin/client 12 4
3
```

At this point you have successfully built a gRPC C++ service. The executables
in `bin/` can be used from anywhere inside the container (they don't need
anything from the installation directory) or outside the container if you have
C++ gRPC libraries installed.

The next steps in this tutorial will publish the service in SingularityNET.

## Step 8 (optional if you already have enough AGI and ETH tokens)

You need some AGI and ETH tokens. You can get then for free (using your github
account) here:

* AGI: https://faucet.singularitynet.io/
* ETH: https://faucet.kovan.network/

## Step 9

Create an "alias" for your private key.

```SH
$ snet identity create MY_ID_NAME KEY_TYPE
```

Replace `MY_ID_NAME` by an id to identify your key in the `SNET-CLI`. This id
will not be seen by anyone. It's just a way to make it easier for you to refer
to your private key (you may have many, btw) in following 'snet' commands. This
alias is kept locally in the container and will vanish when it's shutdown.
`KEY_TYPE` can be either

* key
* rpc
* mnemonic
* ledger
* trezor

You may find detailed information regarding key types (and other `SNET-CLI`
features) in https://github.com/singnet/snet-cli

In this tutorial we'll use `KEY_TYPE == key`. Enter your private key when prompted.

## Step 10 (optional if you already have an organization) 

Create an organization and add your key to it.

```SH
$ snet organization create ORGANIZATION_NAME PUBLIC_KEY
```

Replace `ORGANIZATION_NAME` by a name of your choice and replace `PUBLIC_KEY`
by the public key associated with the private key you used previously.

If you want to join an existing organization (e.g. SNET), ask the owner to add
your key before proceeding.

## Setp 11

Edit a JSON configuration file for your service.  We already have a valid
`service.json` in prject's folder looking like this:

```JSON
{
    "name": "math-operations",
    "service_spec": "src/service_spec",
    "organization": "SNET",
    "path": "",
    "price": 0,
    "endpoint": "http://localhost:70468",
    "tags": [
        "[]"
    ],
    "metadata": {
        "description": ""
    }
}
``` 

Anyway we'll change it to add some useful information in `tags` and `description`.

```JSON
{
    "name": "math-operations",
    "service_spec": "src/service_spec",
    "organization": "SNET",
    "path": "",
    "price": 0,
    "endpoint": "http://localhost:70468",
    "tags": ["tutorial", "math-operations", "basic"],
    "metadata": {
        "description": "A tutorial C++ service"
    }
}
``` 

You could also use `SNET-CLI` build the JSON configuration file
using `snet service init` and answering the prompted questions.

## Step 12

Publish and start your service

```SH
$ ./publishAndStartService.sh PRIVATE_KEY
```

Replace `PRIVATE_KEY` by your wallet's private key (in `Metamask`: menu ->
details -> export private key).  This will start the SNET Daemon and your
service. If everything goes well you will see the blockchain trasaction logs
and then the following 3 messages (respectively from: SNET-CLI, your service and
SNET Daemon):

```
Service published!
Server listening on 0.0.0.0:70468
DEBU[0001] starting daemon                              
```

You can double check if it has been properly published using

```
$ snet organization list-services SNET
```

Optionally you can un-publish the service

```
$ snet service delete ORGANIZATION_NAME SERVICE_NAME
```

Actually, since this is just a tutorial, you are expected to un-publish your
service as soon as you finish the tests.

Other `snet` commands and options (as well as their documentation) can be found here: 
https://github.com/singnet/snet-cli

## Step 13

You can test your service making requests in command line

```
$ ./testServiceRequest.sh 12 4
3
```

That's it. Remember to delete your service as explained in Step 12.

```
$ snet service delete SNET math-operations
```
