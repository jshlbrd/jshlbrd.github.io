---
title: 'Remote Packet Retrieval with Stenographer and gRPC'
author: 'Josh Liburdi'
date: 2019-01-20
description: 'Walkthrough of building a secure packet capture (PCAP) retrieval service using Stenographer, gRPC, and Python.'
tags:
   - nsm
   - grpc
   - python
---

This post describes the process of building a secure packet capture (PCAP) retrieval service for [Google's Stenographer](https://github.com/google/stenographer) using [gRPC](https://grpc.io/) and Python. The service allows clients to securely request PCAP from a server running Stenographer and stream the PCAP back to the client.

# Components of Stenographer
Stenographer contains several utilities (`stenotype`, which writes packets to disk; `stenocurl`, which interacts with the Stenographer system; etc.), but for this service we only need to build on top of `stenoread`. `stenoread` is a utility that retrieves packets stored on disk using a Berkley Packet Filter (BPF) string.

# Components of gRPC
For gRPC, we need to build:
- The protobuf that defines the messages used by the service.
- The service utilities (client and server) and example applications.

# High-Level Design
The high-level design of the service is: the client sends an encrypted and authenticated message to a server running Stenographer requesting PCAP based on a `stenoread`-compatible BPF; the server uses the parameters in the client's message to run `stenoread` and streams the returned PCAP back to the client.

# Building the Protobuf
gRPC uses [Protocol Buffers](https://developers.google.com/protocol-buffers/) (protobuf) to define the messages that clients and servers use to communicate, and as a result may unintentionally constrain the service if not designed correctly.

Let's work through some of our options. We'll start with the client-side message(s) because they're the most complex:
```
message StenoRequest {
  Request request = 1;
  string path = 2;
  int32 epoch_start = 3;
  int32 epoch_end = 4;
  string bpf = 5;
}
message Request {
  string uid = 1;
  int32 chunk_size = 2;
  int32 max_size = 3;
}
```

There's a lot to unpack here, so let's begin with the `Request` message. This message contains parameters for correlating requests and responses ("uid") and controlling the flow and amount of data of gRPC responses (`chunk_size` and `max_size`).

To build a Stenographer-specific message, `Request` is embedded along with parameters that query Stenographer -- this includes the location where `stenoread` should temporarily store retrieved PCAP ("path"), the timestamp of the first packet (`epoch_start`), the timestamp of the last packet (`epoch_end`), and the BPF string (`bpf`). 

By building a Stenographer-specific message, we've made an explicit design decision: this service will only work with Stenographer. Alternatively, we could have designed the protobuf to be more generic and compatible with other packet capture tools. For example, instead of putting the BPF into a single field, we could have created a BPF message, like this:

```
message Bpf {
  string protocol = 1;
  string source_ip = 2;
  int32 source_port = 3;
  string destination_ip = 4;
  int32 destintation_port = 5;
}
```

If we did this, then we could build the BPF string on the server, ensuring that it is compatible with whatever packet capture utility we're running (Stenographer, netsniff-ng, etc.). Instead, since this service is specific to Stenographer, we'll use the `StenoRequest` message above.

The server-side message is less complex:

```
message Response {
  string uid = 1;
  bytes pcap = 2;
}
```

This is a minimal server-side message -- it includes the unique ID from the request (`uid`) and a bytes field containing the PCAP data (`pcap`). 

However, there's more we could potentially do here. If you regularly parse PCAP via tcpdump (which is what `stenoread` does), then you may have found that you don't always get the PCAP you expected. For example, tcpdump returns an empty PCAP file of 24 bytes if no parsed PCAP is found; additionally, you can also truncate a PCAP file with minimal data loss (packets at the end of the file are corrupted but the rest of the file is intact). 

What if we looked for these conditions on the server and wrote them into a field so that clients are better equipped to interpret the response? We could expand our message like this:

```
/*
Status Codes
200 == OK, no parsing errors
210 == OK, truncated PCAP
400 == Unknown failure
410 == Empty PCAP (24 bytes)
*/

message Response {
  string uid = 1;
  bytes pcap = 2;
  int32 status = 3;
}
```

For this service, we'll use the simpler version of the `Response` message.

The last piece of design for the protobuf is to create the Stenographer service. This is the easiest part -- we know that the client will make a single request for PCAP and the server will stream back responses of PCAP.

```
service Stenographer {
  rpc FetchPcap(StenoRequest) returns (stream Response) {}
}
```

That leaves us with this complete protobuf:

```
syntax = "proto3";
package stenographer;

service Stenographer {
  rpc FetchPcap(StenoRequest) returns (stream Response) {}
}

message StenoRequest {
  Request request = 1;
  string path = 2;
  int32 epoch_start = 3;
  int32 epoch_end = 4;
  string bpf = 5;
}

message Request {
  string uid = 1;
  int32 chunk_size = 2;
  int32 max_size = 3;
}

message Response {
  string uid = 1;
  bytes pcap = 2;
}
```

Compiling the Protobuf
Now for the easy part -- compile the protobuf in your preferred language. We'll use Python because it's an easy language for prototyping.

```
pip install grpcio grpcio-tools
python -m grpc_tools.protoc -I. --python_out=. --grpc_python_out=. stenogrpc/stenographer.proto
```

(Note: generally you would run the protobuf compilation command from the main directory of a project.)

# Building the Service
With the protobuf designed and compiled, we're now ready to build the service components. The client requires at least four components: configure gRPC SSL channel credentials, setup a gRPC channel with the server, create the `StenoRequest` message, and request/receive the PCAP. The server requires at least three components: configure gRPC SSL server credentials, define the Stenographer service code, and configure/start the gRPC server. Storing many of these components as functions will help with prototyping.

## Certificates? In My gRPC?
While gRPC supports insecure channels, this service should only support secure, client-authenticated channels. The reason for this is straightforward: packet captures may contain sensitive data, so we should use strict access control when allowing clients to access it. The easiest method I've seen for managing SSL infrastructure for a service like this is [Square's certstrap](https://github.com/square/certstrap).

# Client Code
Below are functions for use in the client, including a function to load SSL channel credentials, create a `StenoRequest` message containing the `stenoread` search parameters, and request/write PCAP to local disk.

```python
import os
import uuid

import grpc

from stenogrpc import stenographer_pb2


def load_credentials(root_certificate, private_key, certificate_chain):
    """Creates client-side gRPC SSL credentials.

    Args:
        root_certificate: Path to CA that has signed the client's certificate.
        private_key: Path to client's private key.
        certificate_chain: Path to the client's certificate.
    Returns:
        Valid gRPC SSL channel credentials.
    """
    with open(root_certificate, 'rb') as fin:
        loaded_root_certificate = fin.read()
    with open(private_key, 'rb') as fin:
        loaded_private_key = fin.read()
    with open(certificate_chain, 'rb') as fin:
        loaded_certificate_chain = fin.read()
    return grpc.ssl_channel_credentials(root_certificates=loaded_root_certificate, 
                                        private_key=loaded_private_key, 
                                        certificate_chain=loaded_certificate_chain)
                                        
def stenographer_request(epoch_start, epoch_end, bpf,
                         uid=None, chunk_size=1000, max_size=5000000, path='/tmp/'):
    """Loads Stenographer request parameters into a StenoRequest protobuf.

    Args:
        epoch_start: Epoch representation of the first packet time to fetch.
        epoch_end: Epoch representation of the last packet time to fetch.
        bpf: String containing Berkley Packet Filter parameters to fetch PCAP for.
        uid: Unique ID of the PCAP request.
            Defaults to random UUID-generated string.
        chunk_size: Size of individual gRPC PCAP streams sent back in response.
            Defaults to 1000 (1 MB).
        max_size: Maximum size of the total returned PCAP.
            Defaults to 5000000 (5MB).
        path: Path where the server will store temporarily extracted PCAP.
            Defaults to '/tmp/'.

    Returns:
        Tuple containing the uid and StenoRequest protobuf.
    """
    if not uid:
        uid = str(uuid.uuid4())
    protobuf = stenographer_pb2.StenographerRequest()
    protobuf.request.uid = uid
    protobuf.request.chunk_size = chunk_size
    protobuf.request.max_size = max_size
    protobuf.path = path
    protobuf.epoch_start = epoch_start
    protobuf.epoch_end = epoch_end
    protobuf.bpf = bpf
    return (uid, protobuf)


def write_pcap(uid, protobuf, stub, path='.'):
    """
    Writes fetched PCAP to local disk.

    Args:
        uid: Unique ID of the PCAP request. This is used as the filename for
            the returned PCAP.
        protobuf: Protocol buffer that contains the PCAP request.
        stub: gRPC stub sourced from the stenogrpc library.
        path: Path where returned PCAP will be stored.
            Defaults to local directory.
    """
    pcap_file = os.path.join(path, "{}.pcap".format(uid))
    with open(pcap_file, "wb") as fout:
        for response in stub.FetchPcap(protobuf):
            fout.write(response.pcap)
```

# Server Code
Below are functions and a class for use in the server, including a function to load SSL server credentials, a function to create a Stenographer server, and the gRPC Stenographer service class that queries `stenoread` and returns PCAP back to the client.

```python
from concurrent import futures
from datetime import datetime
import os
import subprocess

import grpc

from stenogrpc import stenographer_pb2
from stenogrpc import stenographer_pb2_grpc


class StenographerServicer(stenographer_pb2_grpc.StenographerServicer):
    """RPC that queries Stenographer and returns extracted PCAP."""
    def FetchPcap(self, request, context):
        # Format timestamps for stenoread compatibility
        epoch_start = request.epoch_start
        epoch_end = request.epoch_end
        start_dt = datetime.utcfromtimestamp(epoch_start)
        start_strf = start_dt.strftime('%Y-%m-%dT%H:%M:%SZ')
        end_dt = datetime.utcfromtimestamp(epoch_end)
        end_strf = end_dt.strftime('%Y-%m-%dT%H:%M:%SZ')
        # Establish temporary PCAP file location and call stenoread
        pcap_file = os.path.join(request.path,
                                 '{}.pcap'.format(request.request.uid))
        subprocess.call(['stenoread',
                         'after {} and before {} and {}'.format(start_strf,
                                                                end_strf,
                                                                request.bpf),
                         '-w',
                         pcap_file],
                        stdout=subprocess.DEVNULL,
                        stderr=subprocess.DEVNULL)
        # Iterate temporary PCAP, stream back to client, and remove it
        pcap_offset = 0
        with open(pcap_file, 'rb') as fin:
            while pcap_offset < request.request.max_size:
                pcap_offset += request.request.chunk_size
                yield stenographer_pb2.Response(uid=request.request.uid,
                                                pcap=fin.read(request.request.chunk_size))
        os.remove(pcap_file)


def load_credentials(root_certificate, private_key, certificate_chain):
    """Creates server-side gRPC SSL credentials.

    Args:
        root_certificate: Path to CA that has signed the server's certificate.
        private_key: Path to server's private key.
        certificate_chain: Path to the server's certificate.
    Returns:
        Valid gRPC SSL server credentials with client authentication required.
    """
    with open(root_certificate, 'rb') as fin:
        loaded_root_certificate = fin.read()
    with open(private_key, 'rb') as fin:
        loaded_private_key = fin.read()
    with open(certificate_chain, 'rb') as fin:
        loaded_certificate_chain = fin.read()
    return grpc.ssl_server_credentials([(loaded_private_key, loaded_certificate_chain)],
                                       root_certificates=loaded_root_certificate,
                                       require_client_auth=True)


def stenographer_server(credentials, port=8443, workers=10):
    """Configures a gRPC server to run the StenographerServicer.

    Args:
        credentials: gRPC SSL server credentials.
        port: Port that the stenogrpc service listens on.
            Defaults to 8443.
        workers: Maximum number of thread workers the stenogrpc service will create.
            Defaults to 10.
    """
    server = grpc.server(futures.ThreadPoolExecutor(max_workers=workers))
    stenographer_pb2_grpc.add_StenographerServicer_to_server(StenographerServicer(),
                                                             server)
    server.add_secure_port('[::]:{}'.format(port),
                           credentials)
    return server
```

# Client and Server Apps
Here's an example client that is configured from YAML, reads PCAP requests from a CSV, and writes the retrieved PCAP to local disk. Recall that PCAP requests can come from anywhere (CSV, database, network socket, etc.) and PCAP responses can be stored/sent anywhere (local disk, database, etc.) as well -- we just need to add functionality for it.

```python
import argparse
import csv
import yaml

import grpc

from stenogrpc import client
from stenogrpc import stenographer_pb2_grpc

if __name__ == '__main__':
    """Toy version of a stenogrpc client.

    This client reads from a CSV containing BPFs, fetches PCAP from a server
    running Google's Stenographer packet capture utility, and writes the PCAP to
    disk. A YAML configuration file is used to load the client's SSL credentials.
    """
    parser = argparse.ArgumentParser(prog="toy-fetcher.py")
    parser.add_argument("-c", required=True,
                        dest="config", type=str)
    parser.add_argument("-r", required=True,
                        dest="requests", type=str)
    parser.add_argument("-s", required=True,
                        dest="server", type=str)
    args = parser.parse_args()
    with open(args.config) as fin:
        config = yaml.load(fin.read())

    credentials = client.load_credentials(root_certificate=config['root_certificate'],
                                          private_key=config['private_key'],
                                          certificate_chain=config['certificate_chain'])

    with grpc.secure_channel(args.server, credentials) as channel:
        stub = stenographer_pb2_grpc.StenographerStub(channel)
        with open(args.requests) as csvfile:
            csvreader = csv.reader(csvfile)
            for row in enumerate(csvreader):
                (uid, protobuf) = client.stenographer_request(epoch_start=int(row[1][0]),
                                                              epoch_end=int(row[1][1]),
                                                              bpf=row[1][2])
                client.write_pcap(uid, protobuf, stub)
```

Similar to the client, here's an example server that is configured from YAML and runs the service. The server-side components don't need as much flexibility as the client-side components -- we want the server to behave the same way (i.e. access `stenoread` identically for every request) no matter what.

```python
import argparse
import time

import yaml

import stenogrpc.server

if __name__ == '__main__':
    """Toy version of a stenogrpc server.

    This server runs a service that integrates with Google's Stenographer packet
    capture utility. A YAML configuration file is used to load the server's SSL
    credentials.
    """
    parser = argparse.ArgumentParser(prog="toy-server.py")
    parser.add_argument("-c", required=True,
                        dest="config", type=str)
    args = parser.parse_args()
    with open(args.config) as fin:
        config = yaml.load(fin.read())

    credentials = stenogrpc.server.load_credentials(root_certificate=config['root_certificate'],
                                                    private_key=config['private_key'],
                                                    certificate_chain=config['certificate_chain'])
    server = stenogrpc.server.stenographer_server(credentials,
                                                  config['port'],
                                                  config['workers'])
    server.start()
    try:
        while True:
            time.sleep(config['sleep'])
    except KeyboardInterrupt:
        server.stop(0)
```

Although these are toy apps, they're tested and fully functional -- these are relatively simple examples of how to securely, remotely query Stenographer with gRPC. A production implementation of this should include logging, better exception handling, resource limits (e.g. set a server-side limit on request size so that someone can't request an absurd amount of PCAP), and PCAP integrity verification.
