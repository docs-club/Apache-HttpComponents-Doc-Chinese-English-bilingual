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