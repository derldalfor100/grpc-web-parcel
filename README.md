# gRPC-Web-Parcel Hello World Guide

This guide is intended to help you get started with gRPC-Web with a simple
Hello World example. For more information about the gRPC-Web project as a
whole, please visit the [main repo](https://github.com/grpc/grpc-web).

All the code for this example can be found in this current directory.

```sh
$ cd net/grpc/gateway/examples/helloworld
```

## Define the Service

First, let's define a gRPC service using
[protocol buffers](https://developers.google.com/protocol-buffers/). Put this
in the `helloworld.proto` file. Here we define a request message, a response
message, and a service with one RPC method: `SayHello`.

```protobuf
syntax = "proto3";

package helloworld;

service Greeter {
  rpc SayHello (HelloRequest) returns (HelloReply);
}

message HelloRequest {
  string name = 1;
}

message HelloReply {
  string message = 1;
}
```

## Implement the Service

Then, we need to implement the gRPC Service. In this example, we will use
NodeJS. Put this in a `server.js` file. Here, we receive the client request,
and we can access the message field via `call.request.name`. Then we construct
a nice response and send it back to the client via `callback(null, response)`.

```js
var PROTO_PATH = __dirname + '/helloworld.proto';

var assert = require('assert');
var grpc = require('@grpc/grpc-js');
var protoLoader = require('@grpc/proto-loader');
var packageDefinition = protoLoader.loadSync(
    PROTO_PATH,
    {keepCase: true,
     longs: String,
     enums: String,
     defaults: true,
     oneofs: true
    });
var protoDescriptor = grpc.loadPackageDefinition(packageDefinition);
var helloworld = protoDescriptor.helloworld;

function doSayHello(call, callback) {
  callback(null, {
    message: 'Hello! ' + call.request.name
  });
}

function getServer() {
  var server = new grpc.Server();
  server.addService(helloworld.Greeter.service, {
    sayHello: doSayHello,
  });
  return server;
}

if (require.main === module) {
  var server = getServer();
  server.bindAsync(
    '0.0.0.0:6555', grpc.ServerCredentials.createInsecure(), (err, port) => {
      assert.ifError(err);
      server.start();
  });
}

exports.getServer = getServer;
```

## Configure the Proxy

Next up, we need to configure the Envoy proxy to forward the browser's gRPC-Web
requests to the backend. Put this in an `envoy.yaml` file. Here we configure
Envoy to listen at port `:8080`, and forward any gRPC-Web requests to a
cluster at port `:6555`.

```yaml
static_resources:
  listeners:
  - name: listener_0
    address:
      socket_address: { address: 0.0.0.0, port_value: 8080 }
    filter_chains:
    - filters:
      - name: envoy.filters.network.http_connection_manager
        typed_config:
          "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
          codec_type: auto
          stat_prefix: ingress_http
          route_config:
            name: local_route
            virtual_hosts:
            - name: local_service
              domains: ["*"]
              routes:
              - match: { prefix: "/" }
                route:
                  cluster: greeter_service
                  max_stream_duration:
                    grpc_timeout_header_max: 0s
              cors:
                allow_origin_string_match:
                - prefix: "*"
                allow_methods: GET, PUT, DELETE, POST, OPTIONS
                allow_headers: keep-alive,user-agent,cache-control,content-type,content-transfer-encoding,custom-header-1,x-accept-content-transfer-encoding,x-accept-response-streaming,x-user-agent,x-grpc-web,grpc-timeout
                max_age: "1728000"
                expose_headers: custom-header-1,grpc-status,grpc-message
          http_filters:
          - name: envoy.filters.http.grpc_web
          - name: envoy.filters.http.cors
          - name: envoy.filters.http.router
  clusters:
  - name: greeter_service
    connect_timeout: 0.25s
    type: logical_dns
    http2_protocol_options: {}
    lb_policy: round_robin
    load_assignment:
      cluster_name: cluster_0
      endpoints:
        - lb_endpoints:
            - endpoint:
                address:
                  socket_address:
                    address: 0.0.0.0
                    port_value: 6555
```

> NOTE: As per [this issue](https://github.com/grpc/grpc-web/issues/436): if
> you are running Docker on Mac/Windows/WSL, change the last `address: 0.0.0.0` to
>
> ```yaml
>     ...
>     socket_address:
>         address: host.docker.internal
> ```
>
> or if your version of Docker on Mac older then v18.03.0, change it to:
>
> ```yaml
>     ...
>     socket_address:
>         address: docker.for.mac.localhost
> ```

## Write Client Code

Now, we are ready to write some client code! Put this in a `client.js` file.

```js
const {HelloRequest, RepeatHelloRequest,
       HelloReply} = require('./helloworld_pb.js');
const {GreeterClient} = require('./helloworld_grpc_web_pb.js');

var client = new GreeterClient('http://' + window.location.hostname + ':8080',
                               null, null);

// simple unary call
var request = new HelloRequest();
request.setName('World');

client.sayHello(request, {}, (err, response) => {
  if (err) {
    console.log(`Unexpected error for sayHello: code = ${err.code}` +
                `, message = "${err.message}"`);
  } else {
    console.log(response.getMessage());
  }
});


// server streaming call
var streamRequest = new RepeatHelloRequest();
streamRequest.setName('World');
streamRequest.setCount(5);

var stream = client.sayRepeatHello(streamRequest, {});
stream.on('data', (response) => {
  console.log(response.getMessage());
});
stream.on('error', (err) => {
  console.log(`Unexpected stream error: code = ${err.code}` +
              `, message = "${err.message}"`);
});
```

The classes `HelloRequest`, `HelloReply` and `GreeterClient` we import here are
generated for you by the `protoc` generator utility (which we will cover in the
next section) from the `helloworld.proto` file we defined earlier.

Then we instantiate a `GreeterClient` instance, set the field in the
`HelloRequest` protobuf object, and we can make a gRPC call via
`client.sayHello()`, just like how we defined in the `helloworld.proto` file.


You will need a `package.json` file. This is needed for both the `server.js` and
the `client.js` files.

```json
{
  "name": "grpc-web-parcel",
  "version": "0.1.0",
  "description": "gRPC Web Parcel",
  "devDependencies": {
    "@grpc/grpc-js": "~1.1.8",
    "@grpc/proto-loader": "~0.5.4",
    "async": "~1.5.2",
    "google-protobuf": "~3.14.0",
    "grpc-web": "~1.3.0",
    "lodash": "~4.17.0",
    "parcel": "^2.0.1"
  },
  "scripts": {
    "build": "sh build.sh",
    "startLocal": "parcel index.html",
    "buildParcel": "parcel build index.html"
  }
}
```

And finally a simple `index.html` file.

```html
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<title>gRPC-Web Parcel Example</title>
</head>
<body>
  <p>Open up the developer console and see the logs for the output.</p>
  <script src="./client.js" type="module"></script>
</body>
</html>
```

The `./dist/index.[hash].js` file will be generated by `parcel` (which will be covered
in the next section).


And that's it! We have all the code ready. Let's run the example!

## Generate Protobuf Messages and Client Service Stub

To generate the protobuf messages and client service stub class from your
`.proto` definitions, we need:
 - the `protoc` binary, _and_
 - the `protoc-gen-grpc-web` plugin.

> You can download the `protoc-gen-grpc-web` protoc plugin from our
> [release](https://github.com/grpc/grpc-web/releases) page.
>
> If you don't already have `protoc` installed, you will have to download it
> first from [here](https://github.com/protocolbuffers/protobuf/releases).
>
> Make sure they are both executable and are discoverable from your PATH.
>
> For example, in MacOS, you can do:
>
> ```sh
> $ sudo mv ~/Downloads/protoc-gen-grpc-web-1.3.0-darwin-x86_64 \
>   /usr/local/bin/protoc-gen-grpc-web
> $ sudo chmod +x /usr/local/bin/protoc-gen-grpc-web
> ```

Linux:
`protoc -`
PB_REL="https://github.com/protocolbuffers/protobuf/releases"
curl -LO $PB_REL/download/v3.15.8/protoc-3.15.8-linux-x86_64.zip
unzip protoc-3.15.8-linux-x86_64.zip -d $HOME/.local
export PATH="$PATH:$HOME/.local/bin"

`protoc-gen-grpc-web -`
sudo mv protoc-gen-grpc-web-1.3.0-linux-x86_64 /usr/local/bin/protoc-gen-grpc-web
chmod +x /usr/local/bin/protoc-gen-grpc-web

When you have both `protoc` and `protoc-gen-grpc-web` installed, you can now
run this command:

```sh
$ protoc -I=. helloworld.proto \
  --js_out=import_style=commonjs:. \
  --grpc-web_out=import_style=commonjs,mode=grpcwebtext:.
```

Or add a `build.sh` file and add "build": "sh build.sh" on scipts section in package.json.
Run the sh file by `yarn build`.

After the command runs successfully, you should now see two new files generated
in the current directory:

 - `helloworld_pb.js`: this contains the `HelloRequest` and `HelloReply`
   classes
 - `helloworld_grpc_web_pb.js`: this contains the `GreeterClient` class

These are also the 2 files that our `client.js` file imported earlier in the
example.

## Compile the Client JavaScript Code

Next, we need to compile the client side JavaScript code into something that
can be consumed by the browser.

```sh
$ yarn install
$ yarn buildParcel
```

Here we use `parcel` and give it an entry point `client.js`. You can also use
`browserify`, `webpack` or other similar tools. This will resolve all the `require()`
statements and produce a `./dist/index.[hash].js` file that can be embedded in our
`index.html` file.

## Run the Example!

We are ready to run the Hello World example. The following set of commands will
run the 3 processes all in the background.

 1. Run the NodeJS gRPC Service. This listens at port `:6555`.

 ```sh
 $ node server.js &
 ```

 2. Run the Envoy proxy. The `envoy.yaml` file configures Envoy to listen to
 browser requests at port `:8080`, and forward them to port `:6555` (see
 above).

 ```sh
 $ sudo docker run -d -v "$(pwd)"/envoy.yaml:/etc/envoy/envoy.yaml:ro \
     --network=host envoyproxy/envoy:v1.20.0
 ```

> NOTE: As per [this issue](https://github.com/grpc/grpc-web/issues/436):
> if you are running Docker on Mac/Windows or WSL, remove the `--network=host` option:
>
> ```sh
> $ sudo docker run -d -v "$(pwd)"/envoy.yaml:/etc/envoy/envoy.yaml:ro \
>     -p 8080:8080 -p 9901:9901 envoyproxy/envoy:v1.20.0
>  ```

 3. Run the simple Web Server. Open VsCode from
 `dist` folder, add the extension: `Live Server`.

Click on `Go Live` button, and then the live server will open a browser tab in

```
localhost:5000
```

Open up the developer console and you should see the following printed out:

```
Hello! World
```
