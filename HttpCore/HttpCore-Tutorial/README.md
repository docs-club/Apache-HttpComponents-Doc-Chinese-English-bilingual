# HttpCore Tutorial

## Oleg Kalnichevski

## 4.4.5

Licensed to the Apache Software Foundation (ASF) under one or more contributor license agreements. See the NOTICE file distributed with this work for additional information regarding copyright ownership. The ASF licenses this file to you under the Apache License, Version 2.0 (the "License"); you may not use this file except in compliance with the License. You may obtain a copy of the License at

http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software distributed under the License is distributed on an "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. See the License for the specific language governing permissions and limitations under the License.

## Contents

- 0 Preface
  - 1 HttpCore Scope
  - 2 HttpCore Goals
  - 3 What HttpCore is NOT
- 1 Fundamentals
  - 1.1 HTTP messages
    - 1.1.1 Structure
    - 1.1.2 Basic operations
      - 1.1.2.1 HTTP request message
      - 1.1.2.2 HTTP response message
      - 1.1.2.3 HTTP message common properties and methods
    - 1.1.3 HTTP entity
      - 1.1.3.1 Repeatable entities
      - 1.1.3.2 Using HTTP entities
      - 1.1.3.3 Ensuring release of system resources
    - 1.1.4 Creating entities
      - 1.1.4.1 BasicHttpEntity
      - 1.1.4.2 ByteArrayEntity
      - 1.1.4.3 StringEntity
      - 1.1.4.4 InputStreamEntity
      - 1.1.4.5 FileEntity
      - 1.1.4.6 HttpEntityWrapper
      - 1.1.4.7 BufferedHttpEntity
  - 1.2 HTTP protocol processors
    - 1.2.1 Standard protocol interceptors
      - 1.2.1.1 RequestContent
      - 1.2.1.2 ResponseContent
      - 1.2.1.3 RequestConnControl
      - 1.2.1.4 ResponseConnControl
      - 1.2.1.5 RequestDate
      - 1.2.1.6 ResponseDate
      - 1.2.1.7 RequestExpectContinue
      - 1.2.1.8 RequestTargetHost
      - 1.2.1.9 RequestUserAgent
      - 1.2.1.10 ResponseServer
    - 1.2.2 Working with protocol processors
  - 1.3 HTTP execution context
    - 1.3.1 Context sharing
- 2 Blocking I/O model
  - 2.1 Blocking HTTP connections
    - 2.1.1 Working with blocking HTTP connections
    - 2.1.2 Content transfer with blocking I/O
    - 2.1.3 Supported content transfer mechanisms
    - 2.1.4 Terminating HTTP connections
  - 2.2 HTTP exception handling
    - 2.2.1 Protocol exception
  - 2.3 Blocking HTTP protocol handlers
    - 2.3.1 HTTP service
      - 2.3.1.1 HTTP request handlers
      - 2.3.1.2 Request handler resolver
      - 2.3.1.3 Using HTTP service to handle requests
    - 2.3.2 HTTP request executor
    - 2.3.3 Connection persistence / re-use
  - 2.4 Connection pools
  - 2.5 TLS/SSL support
  - 2.6 Embedded HTTP server
- 3 Asynchronous I/O based on NIO
  - 3.1 Differences from other I/O frameworks
  - 3.2 I/O reactor
    - 3.2.1 I/O dispatchers
    - 3.2.2 I/O reactor shutdown
    - 3.2.3 I/O sessions
    - 3.2.4 I/O session state management
    - 3.2.5 I/O session event mask
    - 3.2.6 I/O session buffers
    - 3.2.7 I/O session shutdown
    - 3.2.8 Listening I/O reactors
    - 3.2.9 Connecting I/O reactors
  - 3.3 I/O reactor configuration
    - 3.3.1 Queuing of I/O interest set operations
  - 3.4 I/O reactor exception handling
    - 3.4.1 I/O reactor audit log
  - 3.5 Non-blocking HTTP connections
    - 3.5.1 Execution context of non-blocking HTTP connections
    - 3.5.2 Working with non-blocking HTTP connections
    - 3.5.3 HTTP I/O control
    - 3.5.4 Non-blocking content transfer
    - 3.5.5 Supported non-blocking content transfer mechanisms
    - 3.5.6 Direct channel I/O
  - 3.6 HTTP I/O event dispatchers
  - 3.7 Non-blocking HTTP content producers
    - 3.7.1 Creating non-blocking entities
      - 3.7.1.1 NByteArrayEntity
      - 3.7.1.2 NStringEntity
      - 3.7.1.3 NFileEntity
  - 3.8 Non-blocking HTTP protocol handlers
    - 3.8.1 Asynchronous HTTP service
      - 3.8.1.1 Non-blocking HTTP request handlers
      - 3.8.1.2 Asynchronous HTTP exchange
      - 3.8.1.3 Asynchronous HTTP request consumer
      - 3.8.1.4 Asynchronous HTTP response producer
      - 3.8.1.5 Non-blocking request handler resolver
    - 3.8.2 Asynchronous HTTP request executor
      - 3.8.2.1 Asynchronous HTTP request producer
      - 3.8.2.2 Asynchronous HTTP response consumer
  - 3.9 Non-blocking connection pools
  - 3.10. Pipelined request execution
  - 3.11 Non-blocking TLS/SSL
    - 3.11.1 SSL I/O session
      - 3.11.1.1 SSL setup handler
    - 3.11.2 TLS/SSL aware I/O event dispatches
  - 3.12 Embedded non-blocking HTTP server
- 4 Advanced topics
  - 4.1 HTTP message parsing and formatting framework
    - 4.1.1 HTTP line parsing and formatting
    - 4.1.2 HTTP message streams and session I/O buffers
    - 4.1.3 HTTP message parsers and formatters
    - 4.1.4 HTTP header parsing on demand
