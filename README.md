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

## Step 8 - Authorization

It turns out that not everyone should be able to use the API. After a discussion with the team, it was decided that a time-bound bearer token would suffice. The token must be in `yyyy-MM-ddTHHmm` format and within 1 hour of the current time.

In the case a valid bearer token is not provided, we expect a `401`. Let's update the consumer to pass the bearer token, and capture this new `401` scenario.

In `consumer/src/api.js`:

```javascript
    generateAuthToken() {
        return "Bearer " + new Date().toISOString()
    }

    async getAllProducts() {
        return axios.get(this.withPath("/products"), {
            headers: {
                "Authorization": this.generateAuthToken()
            }
        })
            .then(r => r.data);
    }

    async getProduct(id) {
        return axios.get(this.withPath("/product/" + id), {
            headers: {
                "Authorization": this.generateAuthToken()
            }
        })
            .then(r => r.data);
    }
```

In `consumer/src/api.pact.spec.js` we add authentication headers to the request setup for the existing tests:

```js
await provider.addInteraction({
  states: [{ description: "no products exist" }],
  uponReceiving: "get all products",
  withRequest: {
    method: "GET",
    path: "/products",
    headers: {
      Authorization: like("Bearer 2019-01-14T11:34:18.045Z"),
    },
  },
  willRespondWith: {
    status: 200,
    headers: {
      "Content-Type": "application/json; charset=utf-8",
    },
    body: [],
  },
});
```

and we also add two new tests for the "no auth token" use case:

```js
// ...
test("no auth token", async () => {
  // set up Pact interactions
  await provider.addInteraction({
    states: [{ description: "product with ID 10 exists" }],
    uponReceiving: "get product by ID 10 with no auth token",
    withRequest: {
      method: "GET",
      path: "/product/10",
    },
    willRespondWith: {
      status: 401,
    },
  });

  await provider.executeTest(async (mockService) => {
    const api = new API(mockService.url);

    // make request to Pact mock server
    await expect(api.getProduct("10")).rejects.toThrow(
      "Request failed with status code 401"
    );
  });
});
```

Generate a new Pact file:

```console
❯ npm run test:pact --prefix consumer

PASS src/api.pact.spec.js
  API Pact test
    getting all products
      ✓ products exists (23ms)
      ✓ no products exists (13ms)
      ✓ no auth token (14ms)
    getting one product
      ✓ ID 10 exists (12ms)
      ✓ product does not exist (12ms)
      ✓ no auth token (14ms)

Test Suites: 1 passed, 1 total
Tests:       6 passed, 6 total
Snapshots:   0 total
Time:        2.469s, estimated 3s
Ran all test suites matching /pact.spec.js/i.
```

We should now have two new interactions in our pact file.

Let's test the provider:

```console
❯ npm run test:pact --prefix provider

Verifying a pact between FrontendWebsite and ProductService

  get all products
    returns a response which
      has status code 200 (OK)
      includes headers
        "Content-Type" with value "application/json; charset=utf-8" (OK)
      has a matching body (OK)

  get product by ID 10 with no auth token
    returns a response which
      has status code 401 (FAILED)
      has a matching body (OK)

  get product with ID 10
    returns a response which
      has status code 200 (OK)
      includes headers
        "Content-Type" with value "application/json; charset=utf-8" (OK)
      has a matching body (OK)

  get product with ID 11
    returns a response which
      has status code 404 (OK)
      has a matching body (OK)

  get all products
    returns a response which
      has status code 401 (FAILED)
      has a matching body (OK)


Failures:

1) Verifying a pact between FrontendWebsite and ProductService Given product with ID 10 exists - get product by ID 10 with no auth token
    1.1) has status code 401
           expected 401 but was 200
2) Verifying a pact between FrontendWebsite and ProductService Given products exist - get all products
    2.1) has status code 401
           expected 401 but was 200
```

Now with the most recently added interactions where we are expecting a response of 401 when no authorization header is sent, we are getting 200...

Move on to [step 9](https://github.com/bookmd/pact-workshop-js/tree/step9#step-9---implement-authorisation-on-the-provider)\*
