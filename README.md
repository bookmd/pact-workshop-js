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

## Step 6 - Consumer updates contract for missing products

We're now going to add 2 more scenarios for the contract

- What happens when we make a call for a product that doesn't exist? We assume we'll get a `404`.

- What happens when we make a call for getting all products but none exist at the moment? We assume a `200` with an empty array.

Let's write a test for these scenarios, and then generate an updated pact file.

In `consumer/src/api.pact.spec.js`:

```javascript
// within the 'getting all products' group
test("no products exists", async () => {
  // set up Pact interactions
  await provider.addInteraction({
    state: "no products exist",
    uponReceiving: "get all products",
    withRequest: {
      method: "GET",
      path: "/products",
    },
    willRespondWith: {
      status: 200,
      headers: {
        "Content-Type": "application/json; charset=utf-8",
      },
      body: [],
    },
  });

  const api = new API(provider.mockService.baseUrl);

  // make request to Pact mock server
  const product = await api.getAllProducts();

  expect(product).toStrictEqual([]);
});

// within the 'getting one product' group
test("product does not exist", async () => {
  // set up Pact interactions
  await provider.addInteraction({
    state: "product with ID 11 does not exist",
    uponReceiving: "get product with ID 11",
    withRequest: {
      method: "GET",
      path: "/product/11",
    },
    willRespondWith: {
      status: 404,
    },
  });

  await provider.executeTest(async (mockService) => {
    const api = new API(mockService.url);

    // make request to Pact mock server
    await expect(api.getProduct("11")).rejects.toThrow(
      "Request failed with status code 404"
    );
  });
});
```

Notice that our new tests look almost identical to our previous tests, and only differ on the expectations of the _response_ - the HTTP request expectations are exactly the same.

```console
❯ npm run test:pact --prefix consumer

PASS src/api.pact.spec.js
  API Pact test
    getting all products
      ✓ products exists (24ms)
      ✓ no products exists (13ms)
    getting one product
      ✓ ID 10 exists (14ms)
      ✓ product does not exist (14ms)

Test Suites: 1 passed, 1 total
Tests:       4 passed, 4 total
Snapshots:   0 total
Time:        2.437s, estimated 3s
Ran all test suites matching /pact.spec.js/i.

```

What does our provider have to say about this new test:

```console
❯ npm run test:pact --prefix provider

Verifying a pact between FrontendWebsite and ProductService

  get all products
    returns a response which
      has status code 200 (OK)
      includes headers
        "Content-Type" with value "application/json; charset=utf-8" (OK)
      has a matching body (FAILED)

  get product with ID 10
    returns a response which
      has status code 200 (OK)
      includes headers
        "Content-Type" with value "application/json; charset=utf-8" (OK)
      has a matching body (OK)

  get product with ID 11
    returns a response which
      has status code 404 (FAILED)
      has a matching body (OK)

  get all products
    returns a response which
      has status code 200 (OK)
      includes headers
        "Content-Type" with value "application/json; charset=utf-8" (OK)
      has a matching body (OK)


Failures:

1) Verifying a pact between FrontendWebsite and ProductService Given no products exist - get all products
    1.1) has a matching body
           $ -> Expected an empty List but received [{"id":"09","name":"Gem Visa","type":"CREDIT_CARD","version":"v1"},{"id":"10","name":"28 Degrees","type":"CREDIT_CARD","version":"v1"},{"id":"11","name":"MyFlexiPay","type":"PERSONAL_LOAN","version":"v2"}]
2) Verifying a pact between FrontendWebsite and ProductService Given product with ID 11 does not exist - get product with ID 11
    2.1) has status code 404
           expected 404 but was 200
```

We expected this failure, because the product we are requesing does in fact exist! What we want to test for, is what happens if there is a different _state_ on the Provider. This is what is referred to as "Provider states", and how Pact gets around test ordering and related issues.

We could resolve this by updating our consumer test to use a known non-existent product, but it's worth understanding how Provider states work more generally.

_Move on to [step 7](https://github.com/bookmd/pact-workshop-js/tree/step7#step-7---adding-the-missing-states)_
