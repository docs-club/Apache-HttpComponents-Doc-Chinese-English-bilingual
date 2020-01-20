# HttpClient Tutorial

**Oleg Kalnichevski**

**Jonathan Moore**

**Jilles van Gurp**

## 4.5.2

Licensed to the Apache Software Foundation (ASF) under one or more contributor license agreements. See the NOTICE file distributed with this work for additional information regarding copyright ownership. The ASF licenses this file to you under the Apache License, Version 2.0 (the "License"); you may not use this file except in compliance with the License. You may obtain a copy of the License at

http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software distributed under the License is distributed on an "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. See the License for the specific language governing permissions and limitations under the License.

## Contents
- 0 Preface
    - 1 HttpClient scope
    - 2 What HttpClient is NOT
- 1 Fundamentals
    - 1.1 Request execution
        - 1.1.1 HTTP request
        - 1.1.2 HTTP response
        - 1.1.3 Working with message headers
        - 1.1.4 HTTP entity
            - 1.1.4.1 Repeatable entities
            - 1.1.4.2 Using HTTP entities
        - 1.1.5 Ensuring release of low level resources
        - 1.1.6 Consuming entity content
        - 1.1.7 Producing entity content
            - 1.1.7.1 HTML forms
            - 1.1.7.2 Content chunking
            - 1.1.8 Response handlers
    - 1.2 HttpClient interface
        - 1.2.1 HttpClient thread safety
        - 1.2.2 HttpClient resource deallocation
    - 1.3 HTTP execution context
    - 1.4 HTTP protocol interceptors
    - 1.5 Exception handling
        - 1.5.1 HTTP transport safety
        - 1.5.2 Idempotent methods
        - 1.5.3 Automatic exception recovery
        - 1.5.4 Request retry handler
    - 1.6 Aborting requests
    - 1.7 Redirect handling
- 2 Connection management
    - 2.1 Connection persistence
    - 2.2 HTTP connection routing
        - 2.2.1 Route computation
        - 2.2.2 Secure HTTP connections
    - 2.3 HTTP connection managers
        - 2.3.1 Managed connections and connection managers
        - 2.3.2 Simple connection manager
        - 2.3.3 Pooling connection manager
        - 2.3.4 Connection manager shutdown
    - 2.4 Multithreaded request execution
    - 2.5 Connection eviction policy
    - 2.6 Connection keep alive strategy
    - 2.7 Connection socket factories
        - 2.7.1 Secure socket layering
        - 2.7.2 Integration with connection manager
        - 2.7.3 SSL/TLS customization
        - 2.7.4 Hostname verification
    - 2.8 HttpClient proxy configuration
- 3 HTTP state management
    - 3.1 HTTP cookies
    - 3.2 Cookie specifications
    - 3.3 Choosing cookie policy
    - 3.4 Custom cookie policy
    - 3.5 Cookie persistence
    - 3.6 HTTP state management and execution context
- 4 HTTP authentication
    - 4.1 User credentials
    - 4.2 Authentication schemes
    - 4.3 Credentials provider
    - 4.4 HTTP authentication and execution context
    - 4.5 Caching of authentication data
    - 4.6 Preemptive authentication
    - 4.7 NTLM Authentication
        - 4.7.1 NTLM connection persistence
    - 4.8 SPNEGO/Kerberos Authentication
        - 4.8.1 SPNEGO support in HttpClient
        - 4.8.2 GSS/Java Kerberos Setup
        - 4.8.3 login.conf file
        - 4.8.4 krb5.conf / krb5.ini file
        - 4.8.5 Windows Specific configuration
- 5 Fluent API
    - 5.1 Easy to use facade API
        - 5.1.1 Response handling
- 6 HTTP Caching
    - 6.1 General Concepts
    - 6.2 RFC-2616 Compliance
    - 6.3 Example Usage
    - 6.4 Configuration
    - 6.5 Storage Backends
- 7 Advanced topics
    - 7.1 Custom client connections
    - 7.2 Stateful HTTP connections
        - 7.2.1 User token handler
        - 7.2.2 Persistent stateful connections
    - 7.3 Using the FutureRequestExecutionService
        - 7.3.1 Creating the FutureRequestExecutionService
        - 7.3.2 Scheduling requests
        - 7.3.3 Canceling tasks
        - 7.3.4 Callbacks
        - 7.3.5 Metrics
