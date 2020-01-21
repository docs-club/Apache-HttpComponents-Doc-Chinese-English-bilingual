# Preface

HttpCore is a set of components implementing the most fundamental aspects of the HTTP protocol that are nonetheless sufficient to develop full-featured client-side and server-side HTTP services with a minimal footprint.

HttpCore has the following scope and goals:

1. HttpCore Scope

- A consistent API for building client / proxy / server side HTTP services
- A consistent API for building both synchronous and asynchronous HTTP services
- A set of low level components based on blocking (classic) and non-blocking (NIO) I/O models

2. HttpCore Goals

- Implementation of the most fundamental HTTP transport aspects
- Balance between good performance and the clarity & expressiveness of API
- Small (predictable) memory footprint
- Self-contained library (no external dependencies beyond JRE)

3. What HttpCore is NOT

- A replacement for HttpClient
- A replacement for Servlet APIs
