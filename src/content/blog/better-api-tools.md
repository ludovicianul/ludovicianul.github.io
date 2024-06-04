---
pubDatetime: 2020-10-05T20:08:00Z
title: We need better tools for Web and API software testing
featured: true
draft: false
tags:
  - api
  - rest
  - testing
description:
  Thoughts about how we can improve the tools we use for testing APIs.
---


We need better and smarter tools for Web & API Software Testing. Especially when it comes to the functional testing side.
This area is mostly dominated by **frameworks** rather than **products**, particularly the open source space.
What I mean by this is that you usually get the means to assemble a set of steps, following a given syntax and execute the result.
And you get some value added services on top like: save your request/response, pretty formatting, UI interfaces, etc.
But you rarely get any intelligence on top of it.

You would expect that after so many years of testing login screens, you will point your tool to the login page and let it do its magic.
And when I say magic I'm not only referring to recognizing the fields, put data in and then submit.
It's also generating the expected test cases which are typical for such a functionality: invalid emails, short passwords, forms of injection, etc. and run those automatically.
But no, you must tell the framework everytime to open the page, find elements, enter data inside the input fields, submit it, wait and so on.
And create all the test cases again to support it. And start from the beginning next time.
90% of the activities and test cases will most likely be the same.

Same goes for API testing. Independent of the logic, business domain or type of API, you have a solid common ground of testing scenarios which will need to be run no matter what:
send invalid values, right-left boundary testing, large values, special characters, injection payloads and so on.
So you start creating all these test cases and use your favorite tooling/framework to model it. When the API changes, you need to change all these test cases also. Some changes might be trivial,
but we know that's rarely the case. What you get in the end is all the activities which are the most time-consuming will need to happen manually,
while the tooling will do the most simple (oversimplifying a bit) aspects of it: submit the request and do some reporting. In the end everything is just another friendlier version of `curl` with some
pretty UI, storage & management capabilities and some nice reporting.

(This is quite similar to ToDo & time management apps. You just get fancier way of doing the same stuff: keep a list of items and organize it and setting some deadlines/reminders. But usually the tool is not the problem, so
using X instead of Y won't make me a time management guru.)

So what's the approach a software engineer has when dealing with repetitive work? It automates it! And this is how [CATS](https://github.com/Endava/cats) started.
From the frustration of testing the same stuff over and over again (especially in a microservices platform).
The ultimate goal was to have something a bit more intelligent which can do its magic with minimum configuration.
When people see `intelligent` they kind of associate it with some sort of AI and/or machine learning.
But I would argue that just having some convention of configuration standards in place can give you enough intelligence.

I had 3 goals in mind when building `CATS`:

- minimum configuration
- no coding effort, but still meaningful and comprehensive
- **entire** testing process must be automated (**create**, run and report test cases) - the `intelligence` part

So that in the end, instead of having QAs doing all that repetitive work for each new service or endpoint, let them focus on creative and exploratory testing scenarios anchored in the context they are operating.

You can find out more about CATS here: [https://ludovicianul.github.io/2020/09/09/cats/](https://ludovicianul.github.io/2020/09/09/cats/). But as a summary,
by only providing: [auth data](https://endava.github.io/cats/docs/getting-started/headers-file) and [maybe some context](https://endava.github.io/cats/docs/getting-started/reference-data-file)
you can point it to your API endpoint and let it do its magic. As simple as that.

As with all the good things there are some caveats:
- it only works with OpenAPI endpoints
- it only works for JSON formats

There is also a limited set of capabilities offered through [template payloads](https://endava.github.io/cats/docs/fuzzers/special-fuzzers/template-fuzzer).

If you feel the same about repetitive work in testing APIs, take CATS for a spin and feel free to contribute.