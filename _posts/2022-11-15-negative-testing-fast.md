---
layout: post
title: Negative API Testing on Steroids
---

It's hard to find good titles for articles. You want to sound interesting enough, but avoid clickbait or sensational, keep it short but also comprehensive.
The *on Steroids* on this one is more like **How to write negative tests for APIs: fast, ideally with no development effort and let you focus on the exploratory part, you know, the one that actually challenges your brain**.

As stated in a [previous article](/2022/03/24/better-api-tools/), my view is that API testing is predictable for more than 50% of the test cases. Independent of the business logic, 
you want to do the same negative scenarios: boundary values, invalid values, very large values, different types of injections and so on. 
Instead of starting all over again with the 124th microservice, why not **automate this boring and predictable part** and focus on the things that are specific 
to the business context and challenge your brain with creative work. 

In the next minutes I'll show you how easy is to use [CATS](https://github.com/Endava/cats) to do negative testing. 
I'll use the [Get Started in 1 minute tutorial](https://endava.github.io/cats/docs/intro/) and use [Vault](https://github.com/hashicorp/vault) as the API under test.
Vault is freshly installed and I don't have any setup or context for it. Following the tutorial:

- I have CATS installed:
- 
![cats](https://github.com/ludovicianul/ludovicianul.github.io/raw/master/images/cats-v.png)

- I have Vault installed:
- 
![cats](https://github.com/ludovicianul/ludovicianul.github.io/raw/master/images/vault-v.png)

- I have the [Vault OpenAPI file](https://developer.hashicorp.com/vault/api-docs/system/internal-specs-openapi)

I want to run CATS now in [blackbox mode](https://endava.github.io/cats/docs/getting-started/running-cats#blackbox-mode):

1. Let's start Vault in dev mode

```shell
vault server -dev
```

![cats](https://github.com/ludovicianul/ludovicianul.github.io/raw/master/images/vault-dev.png)

2. Export the Root Token as an environment variable

```shell
export token=hvs.9eagj2vkhh7VXm40oUux5Dxw
```

3. Run CATS in blackbox mode

```shell
cats --contract=api.json --server=http://localhost:8200/v1 -H "X-Vault-Token=$token" -b 
```

![cats](https://github.com/ludovicianul/ludovicianul.github.io/raw/master/images/vault-r.png)

At the end we get: `26 429` tests in almost `11 minutes`, out of which:
- `26 364` are success (blackbox mode)
- `165` are potential problems (i.e. response from the server was `500`)

Let's now open `cats-report/index.html` to better understand the errors. 

> As the report has 26k tests, it will take a few seconds to load. You can run CATS with `--skipReportingForIgnored` argument to only report errors.

Some findings from the report:
- `/identity/mfa/login-enforcement/{name}` - lots of errors, CATS receives `connection timeout` exceptions
- `/sys/pprof/profile` - lots of error, CATS receives `read timeout` exceptions
- many other errors related to invalid configuration or input data that return `500` instead of `4XX`


Next step is to log these issues under the Vault project. I'll update the article with references once done.

Running CATS in `blackbox mode` is the simplest and fastest way to get a sense on the level of negative testing coverage and how your APIs is 
handling unexpected input. Next article will present how `context mode` helps you uncover deeper issues.