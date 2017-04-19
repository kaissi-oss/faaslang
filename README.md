# FaaSlang

![FaaSlang Logo](/images/faaslang-logo-small.png)

## Function as a Service Language

![travis-ci build](https://travis-ci.org/faaslang/faaslang.svg?branch=master)
![npm version](https://badge.fury.io/js/faaslang.svg)

The following is a working draft of the latest FaaSlang specification, version
**0.0.1**, dated **April 18th, 2017**.

FaaSlang is a simple **open specification** intended to standardize implementation
details around FaaS ("serverless") functions, gateways and client interfaces
across languages. It has been designed with the goal of decreasing
organizational complexity around FaaS microservices by encouraging a simple
convention for how we document and interface with them, **including type safety
mechanisms**. In the same way GraphQL is intended to standardize the way
developers interface with nested relational data, FaaSlang does the same for
FaaS resources.

The current working draft of the FaaSlang specification is
more akin to a query language and protocol rather than a Turing-complete
programming language.

# Table of Contents

1. [Introduction](#introduction)
1. [Why FaaSlang?](#why-faaslang)
1. [Specification](#specification)
   1. [FaaSlang Resource Definition](#faaslang-resource-definition)
   1. [Context Definition](#context-definition)
   1. [Parameters](#parameters)
      1. [Constraints](#constraints)
      1. [Types](#types)
      1. [Type Conversion](#type-conversion)
      1. [Nullability](#nullability)
   1. [FaaSlang Resource Requests](#faaslang-resource-requests)
      1. [Context](#context)
      1. [Errors](#errors)
         1. [GatewayError](#gatewayerror)
         1. [ParameterError](#parametererror)
            1. [Details: Required](#details-required)
            1. [Details: Invalid](#details-invalid)
         1. [FatalError](#fatalerror)
         1. [RuntimeError](#runtimeerror)
         1. [ValueError](#valueerror)
1. [FaaSlang Server and Gateway: Implementation](#faaslang-server-and-gateway-implementation)
1. [Acknowledgements](#acknowledgements)

# Introduction

The way we access resources on the web is changing. What started as SOAP evolved
into the more palatable and standardized RESTful interfaces, and now we're
seeing CRUD take a another step forward as GraphQL begins to mature.

This evolution runs in parallel with the changing way we're thinking about
managing compute resources. We started with servers, physical boxes sitting in
warehouses or maybe even our basement, that we sent information to. These gave
way to the abstractions of VMs, containers, and now individual functional units
with services like AWS Lambda, Google Cloud Functions and Microsoft Azure
Functions.

CRUD isn't the only thing we do on the web, and the emergence of these Function
as a Service (often referred to colloquially as **"serverless"**) offerings
paints that picture very clearly - we now have a tool at our fingertips that
allows us to execute any individual function resource we'd like, conceptually
written in any language, at an unconstrained and massively parallel scale
without managing any server resources directly.

The problem is that the abstraction layer of compute resources has moved faster
than developers can keep up. Each infrastructure provider imposes its own
standard and way of doing things around FaaS to the point we're relying on
developers implementing their own workflows and toolchains instead of
standardizing the offering. This is what the current workflow resembles, for
most FaaS providers:

![Current FaaS Workflow](/images/current-faas-workflow.jpg)

This is where FaaSlang comes in. The goal of FaaSlang is simple; provide a
unified, standardized definition and request interface for FaaS resources. This
standard should be enforceable at the request layer, ideally through a gateway,
and polyglot, meaning it must be able to communicate with functions written in a
majority of industry-standard programming languages from standard web protocols.

![FaaSlang Workflow](/images/faaslang-workflow.jpg)

# Why FaaSlang?

FaaSlang defines semantics for how functions "as services" should be implemented
by vendors (or developers running their own compute clusters). By defining and
enforcing conventions around FaaS, we're creating a standard language and
idioms for how developers think about interfacing with functions as web
services.

FaaSlang is the result of tens of thousands of FaaS deployments, by thousands of
developers, spread across a number of cloud service providers and the need to
standardize our ability to organize and communicate with these functions.

Take a current example of an AWS Lambda function **(A)**;

```javascript
exports.handler = (event, context, callback) => {
  // TODO implement
  callback(null, 'Hello from Lambda');
};
```

Or a Microsoft Azure function **(B)**;

```javascript
module.exports = function (context, req) {
  context.res = {body: 'Hell from Microsoft Azure'};
  context.done();
}
```

FaaSlang instead defines the Node.js function footprint;

```javascript
/**
* @param {Number} myVar A number
* @param {String} requiredVar must be a string!
* @returns {String}
*/
module.exports = (myVar = 1, requiredVar, context, callback) => {
  callback(null, 'Hello from FaaSlang-compliant service vendor.');
};
```

Or the async Node.js function footprint;

```javascript
/**
* @param {Number} myVar A number
* @param {String} requiredVar must be a string!
* @returns {String}
*/
module.exports = async function (myVar = 1, requiredVar, context) => {
  return 'Hello from FaaSlang-compliant service vendor.';
};
```

Where **comments are used as part of the semantic definition** for type-safety
(if they can't be inferred from defaults), expected parameters can be
specifically defined, and you still have an optional `context` object for
more robust execution (argument overloading, etc.)

# Specification

## FaaSlang Resource Definition

A FaaSlang definition is a `definition.json` file that respects the following
format.

Given a function like this (filename `my_function.js`):

```javascript
/**
* This is my function, it likes the greek alphabet
* @param {String} alpha Some letters, I guess
* @param {Number} beta And a number
* @param {Boolean} gamma True or false?
* @return {Object} some value
*/
module.exports = async function my_function (alpha, beta = 2, gamma, context) {
  /* your code */
};
```

You would provide a function definition that looks like this:

```json
{
  "name": "my_function",
  "format": {
    "language": "nodejs",
    "async": true
  },
  "description": "This is my function, it likes the greek alphabet",
  "context": null,
  "params": [
    {
      "name": "alpha",
      "type": "string",
      "description": "Some letters, I guess"
    },
    {
      "name": "beta",
      "type": "number",
      "defaultValue": 2,
      "description": "And a number"
    },
    {
      "name": "gamma",
      "type": "boolean",
      "description": "True or false?"
    }
  ],
  "returns": {
    "type": "object",
    "description": "some value"
  }
}
```

This definition is *extensible*, meaning you can add additional fields to it,
but it **must** obey this schema.

A definition must implement the following fields;

| Field | Definition |
| ----- | ---------- |
| name | A user-readable function name (used to execute the function), must match `/[A-Z][A-Z0-9_]*/i` |
| format | An object requiring a `language` field, along with any implementation details |
| description | A brief description of what the function does, can be empty (`""`) |
| context | An object `{}` or `null`, representing whether or not this function accesses the execution context |
| params | An array of `NamedParameter`s, representing function arguments
| returns | A `Parameter` without a `defaultValue` representing function return value |

## Context Definition

If the function does not access execution context details, this should always
be null. If it is an object, it indicates that the function *does* access
context details (i.e. `remoteAddress`, http headers, etc. - see [Context](#context)).

This object **does not have to be empty**, it can contain vendor-specific
details; for example `"context": {"user": ["id", "email"]}` may indicate
that the execution context specifically accesses authenticated user id and email
addresses.

## Parameters

Parameters have the following format;

| Field | Required | Definition |
| ----- | -------- | ---------- |
| name | NamedParameter Only | The name of the Parameter, must match `/[A-Z][A-Z0-9_]*/i` |
| type | yes | A string representing a valid FaaSlang type |
| description | yes | A short description of the parameter, can be empty string (`""`) |
| defaultValue | no | Must match the specified type, **if not provided this parameter is required** |

### Constraints

The **first parameter can never be of type "Object"**. This is to ensure
request consistency with generic calls (i.e. support for argument overloading)
across all language implementations.

### Types

As FaaSlang is intended to be polyglot, functions defined with it must have
a strongly typed signature. Not all types are guaranteed to be consumable in
the same way in every language, and we will continue to define specifications
for how each language should interface with FaaSlang types. At present,
the types are a limited superset of JSON values.

| Type | Definition | Example Input Values (JSON) |
| ---- | ---------- | -------------- |
| boolean | True or False | `true` or `false` |
| string | Basic text or character strings | `"hello"`, `"GOODBYE!"` |
| number | Any double-precision [Floating Point](https://en.wikipedia.org/wiki/IEEE_floating_point) value | `2e+100`, `1.02`, `-5` |
| float | Alias for `number` | `2e+100`, `1.02`, `-5` |
| integer | Subset of `number`, integers between `-2^53 + 1` and `+2^53 - 1` (inclusive) | `0`, `-5`, `2000` |
| object | Any JSON-serializable Object | `{}`, `{"a":true}`, `{"hello":["world"]}` |
| array | Any JSON-serializable Array | `[]`, `[1, 2, 3]`, `[{"a":true}, null, 5]` |
| buffer | Raw binary octet (byte) data representing a file | `{"_bytes": [8, 255]}` or `{"_base64": "d2h5IGRpZCB5b3UgcGFyc2UgdGhpcz8/"}` |
| any | Any value mentioned above | `5`, `"hello"`, `[]` |

### Type Conversion

The `buffer` type will automatically be converted from any `object` with a
**single key-value pair matching the footprints** `{"_bytes": []}` or `{"_base64": ""}`.

Otherwise, parameters provided to a function are expected to match their
defined types. Requests made over HTTP via query parameters or POST data
with type `application/x-www-form-urlencoded` will be automatically
converted from strings to their respective expected types, when possible
(see [FaaSlang Resource Requests](#faaslang-resource-requests) below):

| Type | Conversion Rule |
| ---- | --------------- |
| boolean | `"t"` and `"true"` become `true`, `"f"` and `"false"` become `false`, otherwise **do not convert** |
| string | No conversion |
| number | Determine float value, if NaN **do not convert**, otherwise convert |
| float | Determine float value, if NaN **do not convert**, otherwise convert |
| integer | Determine float value, if NaN **do not convert**, may fail integer type check if not in range |
| object | Parse as JSON, if invalid **do not convert**, object may fail type check (array, buffer) |
| array | Parse as JSON, if invalid **do not convert**, object may fail type check (object, buffer) |
| buffer | Parse as JSON, if invalid **do not convert**, object may fail type check (object, array) |
| any | No conversion |

### Nullability

All types are nullable, but **nullability can only be specified** by setting
`"defaultValue": null` in the `NamedParameter` definition. That is to say,
if a default value is provided, the type is no longer nullable.

## FaaSlang Resource Requests

FaaSlang-compliant requests *must* complete the following steps;

1. Ensure the **Resource Definition** is valid and compliant, either on storage
    or accession.
1. Performs a handshake (i.e. HTTP) with initial request details
1. Accept an `Array`, `Object` or a string of URLencoded variables
1. If over HTTP and query parameters present, query parameters used as
   URL encoded variables
1. If over HTTP POST and query parameters present, reject requests that try to
   specify a POST body as well with a `GatewayError`
1. If over HTTP POST, requests **must** include a `Content-Type` header or
   a `GatewayError` is immediately returned
1. If over HTTP POST, `Content-Type` **must** be `application/json` for `Array`
   or `Object` data, or `application/x-www-form-urlencoded` for string data or
   a `GatewayError` is immediately returned
1. If `application/x-www-form-urlencoded` values are provided (either via POST
   body or query parameters), convert types based on [Type Conversion](#type-conversion)
   and knowledge of the function definition and create an `Object`
1. If `Array`: Parameters will be checked for type consistency in the order of
   the definition `params`
1. If `Object`: Parameters will be checked for type consistency based on names
   of the definition `params`
1. If any inconsistencies are found, cease execution and immediately return a
   `ParameterError`
1. If a parameter has no defaultValue specified and is not provided, immediately
   return a `ParameterError`
1. Try to execute the function, if the function fails to parse or is not valid,
   immediately return a `FatalError`
1. If a function hits a specified timeout (execution time limit), immediately
   return a `FatalError`
1. If a function returns an error (via callback) or one is thrown and not caught,
   immediately return a `RuntimeError`
1. If function returns inconsistent response (does not match `returns` type),
   immediately return a `ValueError`
1. If no errors are encountered, return the value to the client
1. If over HTTP and `content-type` is not being overloaded (i.e. developer
   specified through a vendor-specific mechanism), return `buffer` type data as
   `application/octet-stream` and any other values as `application/json`.

### Context

Every function intended to be consumed via FaaSlang has the option to specify
an *optional* magic `context` parameter that receives vendor-specific
information about the function execution context - for example, if consumed over
HTTP, header details. FaaSlang definitions must specify whether or not they
consume a `context` object. Context objects are extensible but **MUST** contain
the following fields;

| Field | Definition |
| ----- | ---------- |
| params | An `object` mapping called parameter names to their values |
| http | `null` if not accessed via http, otherwise an `object` |
| http.headers | If accessed via HTTP, an `object` containing header values |

### Errors

Errors returned by FaaSlang-compliant services must follow the following JSON
format:

```json
{
  "error": {
    "type": "GatewayError",
    "message": "You know nothing, Jon Snow",
    "details": {}
  }
}
```

`details` is an optional object that can provide additional Parameter details.
Valid Error types are:

- `GatewayError`
- `ParameterError`
- `FatalError`
- `RuntimeError`
- `ValueError`

#### GatewayError

`GatewayError`s are returned as a result of bad or malformed client data,
  including lack of authorization or a missing function (not found). If over
  HTTP, they **must** returns status codes in the range of `4xx`.

#### ParameterError

`ParameterError`s are a result of Parameters not passing type-safety checks,
  and **must** return status code `400` if over HTTP.

Parameter Errors **must** have the following format;

```json
{
  "error": {
    "type": "ParameterError",
    "message": "ParameterError",
    "details": {...}
  }
}
```

`"details"` should be an object mapping parameter names to their respective
validation (type-checking) errors. Currently, this specification defines
two classifications of a ParameterError for a parameter; *required* and
*invalid*. The format of `"details": {}` should follow this format;

##### Details: Required

```json
{
  "param_name": {
    "message": "((descriptive message stating parameter is required))",
    "required": true
  }
}
```

##### Details: Invalid

```json
{
  "param_name": {
    "message": "((descriptive message stating parameter is invalid))",
    "invalid": true,
    "expected": {
      "type": "number"
    },
    "actual": {
      "type": "string",
      "value": "hello world"
    }
  }
}
```

#### FatalError

`FatalError`s are a result of function mismanagement - either your function
  could not be loaded, executed, or it timed out. These **must** return status
  code `500` if over HTTP.

#### RuntimeError

`RuntimeError`s are a result of uncaught exceptions in your code as it runs,
  including errors you explicitly choose to throw (or send to clients via a
  callback, for example). These **must** return status code `403` if over
  HTTP.

#### ValueError

`ValueError`s are a result of your function returning an unexpected value
  based on FaaSlang type-safety mechanisms. These **must** return status code
  `502` if over HTTP.

`ValueError` looks like an *invalid* ParameterError, where the `details`
Object only ever contains a single key called `"returns"`. These are encountered
due to implementation issues on the part of the function developer.

```json
{
  "error": {
    "type": "ValueError",
    "message": "ValueError",
    "details": {
      "returns": {
        "message": "((descriptive message stating return value is invalid))",
        "invalid": true,
        "expected": {
          "type": "boolean"
        },
        "actual": {
          "type": "number",
          "value": 2017
        }
      }
    }
  }
}
```

# FaaSlang Server and Gateway: Implementation

A fully-compliant FaaSlang gateway (that just uses local function resources)
is available with this package, simply clone it and run `npm test` or look
at the `/tests` folder for more information.

The current FaaSlang specification is used in production by the FaaS
provider [StdLib](https://stdlib.com), and is available for local use with the
[StdLib CLI Package](https://github.com/stdlib/lib) which relies on this
repository as a dependency.

# Acknowledgements

The software contained within this repository has been developed and is
copyrighted by the [StdLib](https://stdlib.com) Team at Polybit Inc. and is
MIT licensed. The specification itself is not intended to be owned by a
specific corporate entity, and has been developed in conjunction with other
developers and organizations.
