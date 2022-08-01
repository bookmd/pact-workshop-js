# Pact JS workshop

## Introduction

This workshop is aimed at demonstrating core features and benefits of contract testing with Pact.

Whilst contract testing can be applied retrospectively to systems, we will follow the [consumer driven contracts](https://martinfowler.com/articles/consumerDrivenContracts.html) approach in this workshop - where a new consumer and provider are created in parallel to evolve a service over time, especially where there is some uncertainty with what is to be built.

This workshop should take from 1 to 2 hours, depending on how deep you want to go into each topic.

**Workshop outline**:

- [step 1: **create consumer**](https://github.com/bookmd/pact-workshop-js/tree/step1#step-1---simple-consumer-calling-provider): Create our consumer before the Provider API even exists
- [step 2: **unit test**](https://github.com/bookmd/pact-workshop-js/tree/step2#step-2---client-tested-but-integration-fails): Write a unit test for our consumer
- [step 3: **pact test**](https://github.com/bookmd/pact-workshop-js/tree/step3#step-3---pact-to-the-rescue): Write a Pact test for our consumer
- [step 4: **pact verification**](https://github.com/bookmd/pact-workshop-js/tree/step4#step-4---verify-the-provider): Verify the consumer pact with the Provider API
- [step 5: **fix consumer**](https://github.com/bookmd/pact-workshop-js/tree/step5#step-5---back-to-the-client-we-go): Fix the consumer's bad assumptions about the Provider
- [step 6: **pact test**](https://github.com/bookmd/pact-workshop-js/tree/step6#step-6---consumer-updates-contract-for-missing-products): Write a pact test for `404` (missing User) in consumer
- [step 7: **provider states**](https://github.com/bookmd/pact-workshop-js/tree/step7#step-7---adding-the-missing-states): Update API to handle `404` case
- [step 8: **pact test**](https://github.com/bookmd/pact-workshop-js/tree/step8#step-8---authorization): Write a pact test for the `401` case
- [step 9: **pact test**](https://github.com/bookmd/pact-workshop-js/tree/step9#step-9---implement-authorisation-on-the-provider): Update API to handle `401` case
- [step 10: **request filters**](https://github.com/bookmd/pact-workshop-js/tree/step10#step-10---request-filters-on-the-provider): Fix the provider to support the `401` case
- [step 11: **pact broker**](https://github.com/bookmd/pact-workshop-js/tree/step11#step-11---using-a-pact-broker): Implement a broker workflow for integration with CI/CD

_NOTE: Each step is tied to, and must be run within, a git branch, allowing you to progress through each stage incrementally. For example, to move to step 2 run the following: `git checkout step2`_

## Learning objectives

If running this as a team workshop format, you may want to take a look through the [learning objectives](./LEARNING.md).

## Requirements

[Docker](https://www.docker.com)

[Docker Compose](https://docs.docker.com/compose/install/)

[Node + NPM](https://nodejs.org/en/)

## Scenario

There are two components in scope for our workshop.

1. Product Catalog website. It provides an interface to query the Product service for product information.
1. Product Service (Provider). Provides useful things about products, such as listing all products and getting the details of an individual product.

## Step 9 - Implement authorisation on the provider

We will add a middleware to check the Authorization header and deny the request with `401` if the token is older than 1 hour.

In `provider/middleware/auth.middleware.js`

```javascript
// 'Token' should be a valid ISO 8601 timestamp within the last hour
const isValidAuthTimestamp = (timestamp) => {
  let diff = (new Date() - new Date(timestamp)) / 1000;
  return diff >= 0 && diff <= 3600;
};

const authMiddleware = (req, res, next) => {
  if (!req.headers.authorization) {
    return res.status(401).json({ error: "Unauthorized" });
  }
  const timestamp = req.headers.authorization.replace("Bearer ", "");
  if (!isValidAuthTimestamp(timestamp)) {
    return res.status(401).json({ error: "Unauthorized" });
  }
  next();
};

module.exports = authMiddleware;
```

In `provider/server.js`

```javascript
const authMiddleware = require("./middleware/auth.middleware");

// add this into your init function
app.use(authMiddleware);
```

We also need to add the middleware to the server our Pact tests use.

In `provider/product/product.pact.test.js`:

```javascript
const authMiddleware = require("../middleware/auth.middleware");
app.use(authMiddleware);
```

This means that a client must present an HTTP `Authorization` header that looks as follows:

```
Authorization: Bearer 2006-01-02T15:04
```

Let's test this out:

```console
❯ npm run test:pact --prefix provider

Verifying a pact between FrontendWebsite and ProductService

  get all products
    returns a response which
      has status code 200 (FAILED)
      includes headers
        "Content-Type" with value "application/json; charset=utf-8" (OK)
      has a matching body (FAILED)

  get product by ID 10 with no auth token
    returns a response which
      has status code 401 (OK)
      has a matching body (OK)

  get product with ID 10
    returns a response which
      has status code 200 (FAILED)
      includes headers
        "Content-Type" with value "application/json; charset=utf-8" (OK)
      has a matching body (FAILED)

  get product with ID 11
    returns a response which
      has status code 404 (FAILED)
      has a matching body (OK)

  get all products
    returns a response which
      has status code 401 (OK)
      has a matching body (OK)


Failures:

1) Verifying a pact between FrontendWebsite and ProductService Given no products exist - get all products
    1.1) has a matching body
           $ -> Type mismatch: Expected List [] but received Map {"error":"Unauthorized"}
    1.2) has status code 200
           expected 200 but was 401
2) Verifying a pact between FrontendWebsite and ProductService Given product with ID 10 exists - get product with ID 10
    2.1) has a matching body
           $ -> Actual map is missing the following keys: id, name, type
    2.2) has status code 200
           expected 200 but was 401
3) Verifying a pact between FrontendWebsite and ProductService Given product with ID 11 does not exist - get product with ID 11
    3.1) has status code 404
           expected 404 but was 401

There were 3 pact failures
```

Oh, dear. _More_ tests are failing. Can you understand why?

_Move on to [step 10](https://github.com/bookmd/pact-workshop-js/tree/step10#step-10---request-filters-on-the-provider)_
