fibers
======

[![fibers](http://meritbadge.herokuapp.com/fibers)](https://crates.io/crates/fibers)
[![License: MIT](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)

[Documentation](https://docs.rs/fibers/)

This is a library to execute a number of lightweight asynchronous tasks (a.k.a, fibers).

Note that `fibers` heavily uses [futures](https://github.com/alexcrichton/futures-rs) to
represent asynchronous task.
So it is recommended to see the `README.md` and `TUTORIAL.md` of
[futures](https://github.com/alexcrichton/futures-rs) before reading following
if you are unfamiliar with it.

A fiber is a future which executed by ... .
(expect users familliar with the futures)
(mio is hidden and no prerequirement)

futures is gread ... . but a remaiing problems is
"how exeuted the futures efficiently (and especially) with asynchronous I/O ?"

fiber is an answer about it.

erlang like, green thread, ....

intuitive, can use like ordinay threads(?).

and main concern of this library is "how execute and schedule the fibers(futures)".
and it is recommened to use other crate like handy_io.

asynchronous i/o and other asynchronous primitives.

multithread support ... . thread transparent
(for example future and i/o primitive free to moves between fibers) .

intuitive and easy to use.


Installation
------------

First, add following lines to your `Cargo.toml`:

```toml
[dependencies]
fibers = "0.1"
futures = "0.1"  # In practical, `futures` is mandatory to use `fibers`.
```

Next, add this to your crate:

```rust
extern crate fibers;
extern crate futures;
```

Several runnable examples are given in the next section.


Examples
--------

The following are examples of writing code to perform asynchronous tasks.

Other examples are found in "fibers/examples" directory.
And you can run an example by executing the following command.

```bash
$ cargo run --example ${EXAMPLE_NAME}
```

### Calculation of fibonacci numbers

```rust
// See also: "fibers/examples/fibonacci.rs"
extern crate fibers;
extern crate futures;

use fibers::{Spawn, Executor, ThreadPoolExecutor};
use futures::{Future, BoxFuture};

fn main() {
    // Creates an executor instance.
    let mut executor = ThreadPoolExecutor::new().unwrap();

    // Creates a future which will calculate the fibonacchi number of `10`.
    let input_number = 10;
    let future = fibonacci(input_number, executor.handle());

    // Spawns and executes the future (fiber).
    let monitor = executor.spawn_monitor(future);
    let answer = executor.run_fiber(monitor).unwrap();

    // Checkes the answer.
    assert_eq!(answer, Ok(55));
}

fn fibonacci<H: Spawn + Clone>(n: usize, handle: H) -> BoxFuture<usize, ()> {
    if n < 2 {
        futures::finished(n).boxed()
    } else {
        /// Spawns a new fiber per recursive call.
        let f0 = handle.spawn_monitor(fibonacci(n - 1, handle.clone()));
        let f1 = handle.spawn_monitor(fibonacci(n - 2, handle.clone()));
        f0.join(f1).map(|(a0, a1)| a0 + a1).map_err(|_| ()).boxed()
    }
}
```

### TCP Echo Server and Client

An example of TCP echo server listening at the address "127.0.0.1:3000":

```rust
// See also: "fibers/examples/tcp_echo_srv.rs"
extern crate fibers;
extern crate futures;
extern crate handy_io;

use std::io;
use fibers::{Spawn, Executor, ThreadPoolExecutor};
use fibers::net::TcpListener;
use futures::{Future, Stream};
use handy_io::io::{AsyncWrite, AsyncRead};
use handy_io::pattern::{Pattern, AllowPartial};

fn main() {
    let server_addr = "127.0.0.1:3000".parse().expect("Invalid TCP bind address");

    let mut executor = ThreadPoolExecutor::new().expect("Cannot create Executor");
    let handle0 = executor.handle();
    let monitor = executor.spawn_monitor(TcpListener::bind(server_addr)
        .and_then(move |listener| {
            println!("# Start listening: {}: ", server_addr);

            // Creates a stream of incoming TCP client sockets
            listener.incoming().for_each(move |(client, addr)| {
                // New client is connected.
                println!("# CONNECTED: {}", addr);
                let handle1 = handle0.clone();

                // Spawns a fiber to handle the client.
                handle0.spawn(client.and_then(move |client| {
                        // For simplicity, splits reading process and
                        // writing process into differrent fibers.
                        let (reader, writer) = (client.clone(), client);
                        let (tx, rx) = fibers::sync::mpsc::channel();

                        // Spawns a fiber for the writer side.
                        // When a message is arrived in `rx`,
                        // this fiber sends it back to the client.
                        handle1.spawn(rx.map_err(|_| -> io::Error { unreachable!() })
                            .fold(writer, |writer, buf: Vec<u8>| {
                                println!("# SEND: {} bytes", buf.len());
                                writer.async_write_all(buf).map(|(w, _)| w).map_err(|(_, _, e)| e)
                            })
                            .then(|r| {
                                println!("# Writer finished: {:?}", r);
                                Ok(())
                            }));

                        // The reader side is executed in the current fiber.
                        let stream = vec![0;1024].allow_partial().repeat();
                        reader.async_read_stream(stream)
                            .map_err(|(_, e)| e)
                            .fold(tx, |tx, (mut buf, len)| {
                                buf.truncate(len);
                                println!("# RECV: {} bytes", buf.len());

                                // Sends received  to the writer half.
                                tx.send(buf).expect("Cannot send");
                                Ok(tx) as io::Result<_>
                            })
                    })
                    .then(|r| {
                        println!("# Client finished: {:?}", r);
                        Ok(())
                    }));
                Ok(())
            })
        }));
    let result = executor.run_fiber(monitor).expect("Execution failed");
    println!("# Listener finished: {:?}", result);
}
```

And the code of the client side:

```rust
// See also: "fibers/examples/tcp_echo_cli.rs"
extern crate fibers;
extern crate futures;
extern crate handy_io;

use fibers::{Spawn, Executor, InPlaceExecutor};
use fibers::net::TcpStream;
use futures::{Future, Stream};
use handy_io::io::{AsyncWrite, AsyncRead};
use handy_io::pattern::{Pattern, AllowPartial};

fn main() {
    let server_addr = "127.0.0.1:3000".parse().unwrap();

    // `InPlaceExecutor` is suitable to execute a few fibers.
    // It does not create any background threads,
    // so the overhead to manage fibers is lower than `ThreadPoolExecutor`.
    let mut executor = InPlaceExecutor::new().expect("Cannot create Executor");
    let handle = executor.handle();

    // Spawns a fiber for echo client.
    let monitor = executor.spawn_monitor(TcpStream::connect(addr).and_then(move |stream| {
        println!("# CONNECTED: {}", addr);
        let (reader, writer) = (stream.clone(), stream);

        // Writer: It sends data read from the standard input stream to the connected server.
        let stdin_stream = fibers::io::stdin()
            .async_read_stream(vec![0; 256].allow_partial().repeat());
        handle.spawn(stdin_stream.map_err(|(_, e)| e)
            .fold(writer, |writer, (mut buf, size)| {
                buf.truncate(size);
                writer.async_write_all(buf).map(|(w, _)| w).map_err(|(_, _, e)| e)
            })
            .then(|r| {
                println!("# Writer finished: {:?}", r);
                Ok(())
            }));

        // Reader: It outputs data received from the server to the standard output stream.
        reader.async_read_stream(vec![0; 256].allow_partial().repeat())
            .map_err(|(_, e)| e)
            .for_each(|(mut buf, len)| {
                buf.truncate(len);
                println!("{}", String::from_utf8(buf).expect("Invalid UTF-8"));
                Ok(())
            })
    }));

    // Runs until the above fiber is terminated (i.e., The TCP stream is disconnected).
    let result = executor.run_fiber(monitor).expect("Execution failed");
    println!("# Disconnected: {:?}", result);
}
```

License
-------

This library is released under the MIT License.

See the [LICENSE](LICENSE) file for full license information.

Copyright (c) 2016 DWANGO Co., Ltd. All Rights Reserved.
