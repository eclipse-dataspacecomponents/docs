# Contributor Documentation

<!-- TOC -->
* [Contributor Documentation](#contributor-documentation)
  * [0. Intended audience](#0-intended-audience)
  * [1. Getting started](#1-getting-started)
    * [1.1 Prerequisites](#11-prerequisites)
    * [1.2 Terminology](#12-terminology)
    * [1.3 Architectural and coding principles](#13-architectural-and-coding-principles)
  * [2. The control plane](#2-the-control-plane)
    * [2.1 Entities](#21-entities)
    * [2.2 Programming Primitives](#22-programming-primitives)
    * [2.3 Serialization via JSON-LD](#23-serialization-via-json-ld)
    * [2.4 Extension model](#24-extension-model)
    * [2.5 Dependency injection deep dive](#25-dependency-injection-deep-dive)
    * [2.6 Service layers](#26-service-layers)
    * [2.7 Policy Monitor](#27-policy-monitor)
    * [2.8 Protocol extensions (DSP)](#28-protocol-extensions-dsp)
  * [3. (Postgre-)SQL persistence](#3-postgre-sql-persistence)
  * [4. The data plane](#4-the-data-plane)
    * [4.1 Data plane signaling](#41-data-plane-signaling)
    * [4.2 Data plane self-registration](#42-data-plane-self-registration)
    * [4.3 Public API authentication](#43-public-api-authentication)
    * [4.4 Writing a custom data plane extension (sink/source)](#44-writing-a-custom-data-plane-extension-sinksource)
    * [4.5 Writing a custom data plane (using only DPS)](#45-writing-a-custom-data-plane-using-only-dps)
  * [5. Development best practices](#5-development-best-practices)
    * [5.1 Writing Unit-, Component-, Integration-, Api-, EndToEnd-Tests](#51-writing-unit--component--integration--api--endtoend-tests)
    * [5.1 Other best practices](#51-other-best-practices)
  * [6. Further concepts](#6-further-concepts)
    * [6.2 Autodoc](#62-autodoc)
    * [6.3 Adapting the Gradle build](#63-adapting-the-gradle-build)
<!-- TOC -->

## 0. Intended audience

This document is aimed at software developers who have already read the [adopter documentation](../for-adopters) and
want to contribute code to the Eclipse Dataspace Components project.

Its purpose is to explain in greater detail the core concepts of EDC. After reading through it, readers should have a
good understanding of EDCs inner workings, implementation details and some of the advanced concepts.

So if you are a solution architect looking for a high-level description on how to integrate EDC, or a software engineer
who wants to use EDC in their project, then this guide is not for you. More suitable resources can be found
[here](https://eclipse-edc.github.io/docs/#/README) and [here](../for-adopters) respectively.

## 1. Getting started

### 1.1 Prerequisites

This document presumes a good understanding and proficiency in the following technical areas:

- JSON and [JSON-LD](https://json-ld.org)
- HTTP/REST
- relational databases (PostgreSQL) and transaction management
- git and git workflows

Further, the following tools are required:

- Java Development Kit 17+
- Gradle 8+
- a POSIX compliant shell (bash, zsh,...)
- a text editor
- CLI tools like `curl` and `git`

This guide will use CLI tools as common denominator, but in many cases graphical alternatives exist (e.g. Postman,
Insomnia, some database client, etc.), and most developers will likely use IDEs like IntelliJ or VSCode. We are of
course aware of them and absolutely recommend their use, but we simply cannot cover and explain every possible
combination of OS, tool and tool version.

> Note that Windows is not a supported OS at the moment. If Windows is a must, we recommend using WSL2 or a setting up a
> Linux VM.

### 1.2 Terminology

- runtime: a Java process executing code written in the EDC programming model (e.g. a control plane)
- distribution: a specific combination of modules, compiled into a runnable form, e.g. a fat JAR file, a Docker image
  etc.
- launcher: a runnable Java module, that pulls in other modules to form a distribution. "Launcher" and "distribution"
  are sometimes used synonymously
- connector: a control plane runtime and 1...N data plane runtimes. Sometimes used interchangeably with "distribution".
- consumer: a dataspace participant who wants to ingest data under the access rules imposed by the provider
- provider: a dataspace participant who offers data assets under a set of access rules

### 1.3 Architectural and coding principles

When EDC was originally created, there were a few fundamental architectural principles around which we designed and
implemented all dataspace components. These include:

- **asynchrony**: all external mutations of internal data structures happen in an asynchronous fashion. While the REST
  requests to trigger the mutations may still be synchronous, the actual state changes happen in an asynchronous and
  persistent way. For example starting a contract negotiation through the API will only return the negotiation's ID, and
  the control plane will cyclically advance the negotiation's state.
- **single-thread processing**: the control plane is designed around a set of sequential [state
  machines](./control-plane/programming-primitives.md#1-state-machines), that employ pessimistic locking to guard
  against race conditions and other problems.
- **idempotency**: requests, that do not trigger a mutation, are idempotent. The same is true when provisioning external
  resources.
- **error-tolerance**: the design goal of the control plane was to favor correctness and reliability over (low) latency.
  That means, even if a communication partner may not be reachable due to a transient error, it is designed to cope with
  that error and attempt to overcome it.

Prospective contributors to the Eclipse Dataspace Components are well-advised to follow these principles and build their
applications around them.

There are other, less technical principles of EDC such as simplicity and self-contained-ness. We are extremely careful
when adding third-party libraries or technologies to maintain a simple, fast and un-opinionated platform.

Take a look at our [coding principles](../../contributing/coding-principles.md) and our
[styleguide](../../contributing/styleguide.md).

## 2. The control plane

Simply put, the control plane is the brains of a connector. Its tasks include handling protocol and API requests,
managing various internal asynchronous processes, validating policies, performing participant authentication and
delegating the data transfer to a data plane. Its job is to handle (almost) all business logic. For that, it is designed
to favor _reliability_ over _low latency_. It does **not** directly transfer data from source to destination.

The primary way to interact with a connector's control plane is through the Management API, all relevant Java modules
are located at `extensions/control-plane/api/management-api`.

### 2.1 Entities

Detailed documentation about entities can be found [here](./control-plane/entities.md)

### 2.2 Programming Primitives

This chapter describes the fundamental architectural and programming paradigms that are used in EDC. Typically, they
are not related to one single extension or feature area, they are of overarching character.

Detailed documentation about programming primitives can be found [here](./control-plane/programming-primitives.md)

### 2.3 Serialization via JSON-LD

JSON-LD is a JSON-based format for serializing [Linked Data](https://www.w3.org/wiki/LinkedData), and allows adding
specific "context" to the data expressed as JSON format.
It is a [W3C](https://www.w3.org/TR/json-ld/) standard since 2010.

Detailed information about how JSON-LD is used in EDC can be found [here](./control-plane/json-ld.md)

### 2.4 Extension model

One of the principles EDC is built around is _extensibility_. This means that by simply putting a Java module on the
classpath, the code in it will be used to enrich and influence the runtime behaviour of EDC. For instance, contributing
additional data persistence implementations can be achieved this way. This is sometimes also referred to as "plugin".

Detailed documentation about the EDC extension model can be found [here](./control-plane/extension-model.md)

### 2.5 Dependency injection deep dive

In EDC, dependency injection is available to inject services into extension classes (implementors of the
`ServiceExtension` interface). The `ServiceExtensionContext` acts as service registry, and since it's not _quite_ an IoC
container, we'll refer to it simple as the "context" in this chapter.

Detailed documentation about the EDC dependency injection mechanism can be
found [here](./control-plane/dependency-injection.md)

### 2.6 Service layers

Like many other applications and application frameworks, EDC is built upon a vertically oriented set of different layers
that we call "service layers".

Detailed documentation about the EDC service layers can be found [here](./control-plane/service-layers.md)

### 2.7 Policy Monitor

The policy monitor is a component that watches over on-going transfers and ensures that the policies associated with the transfer are still valid.

Detailed documentation about the policy monitor can be found [here](./control-plane/policy-monitor.md)

### 2.8 Protocol extensions (DSP)

This chapter describes how EDC abstracts the interaction between connectors in a Dataspace through protocol extensions and introduces the current default implementation which follows the [Dataspace protocol](https://docs.internationaldataspaces.org/ids-knowledgebase/v/dataspace-protocol) specification.

Detailed documentation about protocol extensions can be found [here](./control-plane/protocol-extensions.md)

### 3. (Postgre-)SQL persistence

PostgreSQL is a very popular open-source database and it has a large community and vendor adoption. It is also EDCs data
persistence technology of choice.

Every [store](./control-plane/service-layers.md#5-data-persistence) in the EDC, intended to persist state, comes out of
the box with two implementations:

- in-memory
- sql (PostgreSQL dialect)

By default, the [in-memory stores](./control-plane/service-layers.md#51-in-memory-stores) are provided by the dependency
injection, the SQL variants can be used by simply adding the relevant extensions (e.g. `asset-index-sql`,
`contract-negotiation-store-sql`, ...) to the classpath.

Detailed documentation about EDCs PostgreSQL implementations can be found [here](./postgres-persistence.md)

## 4. The data plane

### 4.1 Data plane signaling

Data Plane Signaling (DPS) is the communication protocol that is used between control planes and data planes. Detailed
information about it and other topics such as data plane self-registration and public API authentication can be found
[here](./data-plane/data-plane-signaling/data-plane-signaling.md).

### 4.2 Writing a custom data plane extension (sink/source)

### 4.3 Writing a custom data plane (using only DPS)

## 5. Development best practices

### 5.1 Writing Unit-, Component-, Integration-, Api-, EndToEnd-Tests

test pyramid...
Like any other project, EDC has established a set of recommendations and rules that contributors must adhere to in order
to guarantee a smooth collaboration with the project. Note that familiarity with our [formal contribution guidelines](../../../CONTRIBUTING.md) is assumed.
There additional recommendations we have compiled that are relevant when deploying and administering EDC instances.

### 5.1 Coding best practices

Code should be written to conform with the EDC [style guide](../../contributing/styleguide.md) and our [coding principles](../../contributing/coding-principles.md). 

A frequent subject of critique in pull requests is logging. Spurious and very verbose log lines like "Entering/Leaving method X" or "Performing action Z" should be avoided because they pollute the log output and don't contribute any value. 

Please find detailed information about logging [here](../../logging.md).

### 5.2 Testing best practices

Every class in the EDC code base should have a test class that verifies the correct functionality of the code.

Detailed information about testing can be found [here](../../testing.md).

### 5.3 Other best practices

Please find general best practices and recommendations [here](./best-practices.md).

## 6. Further concepts

### 6.2 Autodoc

### 6.3 Adapting the Gradle build
