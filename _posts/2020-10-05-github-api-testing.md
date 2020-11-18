---
layout: post
title: Small tips for improving the GitHub API
---
# Disclaimer
<span style="color:red">1. **!!! WARNING !!! If you choose to run the steps in this article, please note that you will end up with a significant number of dummy GitHub repos under your username. Over 1k in my case.
There is a script at the end of the article that you can use to delete them. Be careful not to delete your real repos though!**</span>

![Repos](https://github.com/ludovicianul/ludovicianul.github.io/raw/master/images/github_repos.png)

2. This is a concrete example on how you can use [CATS](https://github.com/Endava/cats) and personal view on how I would improve the GitHub API. The findings are small things that will make the API more usable, predictable and consistent. 

# The Beginning
Building good APIs is hard. There are plenty of resources out there with plenty of good advice on how to achieve this. While some of the things are a must, like following the [OWASP REST Security Practices](https://cheatsheetseries.owasp.org/cheatsheets/REST_Security_Cheat_Sheet.html),
some others might be debatable, based on preference, "like using `snake_case` or `camelCase` for naming JSON objects". (I plan to write a more detailed article in the next weeks on what I consider good practices.)
As the number of APIs usually grows significantly, even when dealing with simple systems, it's very important to have quick ways to make sure the APIs are meeting good practices consistently.

I've shown in a [previous article](https://ludovicianul.github.io/2020/09/09/cats/) how easy it is to use a tool like [CATS](https://github.com/Endava/cats) to quickly verify OpenAPI endpoints 
while covering a significant number of tests cases. But that was a purely didactic showcase using the OpenAPI demo `petstore` app. Which was obviously not built as a production ready service.
Today I'll pick a real-life API, specifically the GitHub API, which recently published [their OpenAPI specs](https://github.com/github/rest-api-description/blob/main/descriptions/ghes-2.22/ghes-2.22.yaml).


I've downloaded the 2.22 version and saved the file locally as `github.yml`. Before we start, we need to [create an access token](https://github.com/settings/tokens/new) in order to be able to call the APIs. Make sure it has proper scopes for repo creation (and deletion when using the script at the end of the article).
Also, as the API is quite rich (the file has 59k lines), I've only selected the `/user/repos` path for this showcase. You'll see that there are plenty of findings only using this endpoint.

You can run `CATS` as a blackbox testing tool and incrementally add minimal bits of context until you end up with consistent issues or a green suite. 

As shown in the [previous article](https://ludovicianul.github.io/2020/09/09/cats/), running [CATS](https://github.com/Endava/cats) is quite simple:  
  
```shell
./cats.jar --contract=github.yml --server=https://api.github.com --paths="/user/repos" --headers=headers_github.yml
```  

With the `headers_github.yml` having the following content:  

```yaml 
all:
  Authorization: token XXXXXXXXXXXXX
```  

Let's see what we get on a first run:

![First Run](https://github.com/ludovicianul/ludovicianul.github.io/raw/master/images/first_run_github.png)  

We have `42 warnings` and `156 errors`. Let's go through the errors first. Looking at the result of `Test 118` we see that a request failed due to the name of the repository not being unique. 
Indeed, `CATS`, for each Fuzzer, preserves an initial payload that will be used to fuzz each of the request fields. This means that we need a way to force `CATS` to send unique names for the `name` field. Noted!

![Test 118](https://github.com/ludovicianul/ludovicianul.github.io/raw/master/images/test_118_github.png)  

`Test 426` says that `If you specify visibility or affiliation, you cannot specify type.`. Let's note this down also.  

![Test 426](https://github.com/ludovicianul/ludovicianul.github.io/raw/master/images/test_426_github.png)  

Considering the above 2 problems are reported consistently, let's give it another go with a [Reference Data File](https://github.com/Endava/cats#reference-data-file).
This is the `refData_github.yml` file that will be used:

```yaml
/user/repos:
  name: "T(org.apache.commons.lang3.RandomStringUtils).random(5,true,true)"
  type: "cats_remove_field"
```  

`CATS` supports [dynamic values in properties values](https://github.com/Endava/cats#dynamic-values-in-configuration-files) via the [Spring Expression Language](https://docs.spring.io/spring-framework/docs/3.0.x/reference/expressions.html).
Using the above `refData` file, `CATS` will now generate a new random `name` everytime it will execute a request to the GitHub API.
Also, using the `cats_remove_field` value, `CATS` will remove this field from all requests before sending them to the endpoint. More details on this feature [here](https://github.com/Endava/cats#removing-fields).

Running `CATS` again:  

```shell
./cats.jar --contract=github.yml --server=https://api.github.com --paths="/user/repos" --headers=headers_github.yml --refData=refData_github.yml
```  

We now get `17 warnings` and `90 errors`. Again, looking though the tests failures/warnings there are some tests which are failing due to the fact the `since` and `before` are not sent in ISO8061 timestamp format (more on this inconsistency in the [Findings](#Findings) section).

![Second Run](https://github.com/ludovicianul/ludovicianul.github.io/raw/master/images/second_run_github.png)

We'll now update the `refData` file to look as follows:

```yaml
/user/repos:
  before: "T(java.time.Instant).now().toString()"
  since: "T(java.time.Instant).now().minusSeconds(86400).toString()"
  name: "T(org.apache.commons.lang3.RandomStringUtils).random(5,true,false)"
  type: "cats_remove_field"
```  

and run `CATS` again:  

```shell
./cats.jar --contract=github.yml --server=https://api.github.com --paths="/user/repos" --headers=headers_github.yml --refData=refData_github.yml
```  

![Third Run](https://github.com/ludovicianul/ludovicianul.github.io/raw/master/images/third_run_github.png)

We now get `5 warnings` and `83 errors`. Looking through the errors and warnings, there is a significant amount which I consider legit issues while some are debatable points, depending on preference/standards being followed. 
Let's go through the findings. 

# Findings
##  Invalid values for boolean fields implicitly converted to false
One of the Fuzzers that `CATS` has is the `BooleanFieldsFuzzer`. 
This Fuzzer works on the assumption that if you send an invalid value into a `boolean` field, the service should return a validation error.
Obviously, the GitHub API does not do this, but is rather silently converting the value to `false`. It's true this is a consistent behaviour, applying for all boolean fields like `auto_init`, `allow_merge_commits`, etc, but I would personally choose to return a validation error in these cases.

This is in contradiction with the [OWASP recommendation](https://cheatsheetseries.owasp.org/cheatsheets/REST_Security_Cheat_Sheet.html#input-validation) around strong input validation and data type enforcing.


## Invalid values for enumerated fields implicitly converted to the default value (?)
The `InvalidValuesInEnumsFieldsFuzzer` will send invalid values in enum fields. It expects a validation error in return. 
The GitHub API does not seem to reject invalid values, but rather convert them to a default value and respond successfully to the request.

This is in contradiction with the [OWASP recommendation](https://cheatsheetseries.owasp.org/cheatsheets/REST_Security_Cheat_Sheet.html#input-validation) around strong input validation and data type enforcing.

## Integer fields accepting decimal or large negative or positive values
The `DecimalValuesInIntegerFieldsFuzzer` expects an error when it sends a `decimal` value inside an `integer` field. The GitHub API seems to accept these invalid values in the `team_id` field without returning any error, but rather resulting in a successful processing of the request.
Same applies for `ExtremePositiveValueInIntegerFieldsFuzzer` and `ExtremeNegativeValueIntegerFieldsFuzzer` which will send values such as `9223372036854775807` or `-9223372036854775808` in the `team_id` field.
Strings also seem to be accepted in the `team_id` field.

This is in contradiction with the [OWASP recommendation](https://cheatsheetseries.owasp.org/cheatsheets/REST_Security_Cheat_Sheet.html#input-validation) around strong input validation and data type enforcing.

## Accepts unsupported or dummy Content-Type headers
The GitHub API seems to successfully accept and process requests containing unsupported (according to the OpenAPI contract) `Content-Type` headers such as: `image/gif`, `multipart/related`, etc. or dummy ones such as `application/cats`. 

This is in contradiction with the [OWASP recommendation](https://cheatsheetseries.owasp.org/cheatsheets/REST_Security_Cheat_Sheet.html#validate-content-types) around validation around content types.

## Accepts duplicate headers
The GitHub API does not reject requests that contain duplicate headers. The HTTP standard itself allows duplicate headers for specific cases, but allowing duplicate headers might lead to hidden bugs.

## Spaces are not trimmed from values
If the request fields are prefixed or trailed with spaces, they are rejected as invalid values. For example sending a `Haskell` space-prefixed value in the `gitignore_template` will cause the service to return an error.
Although this is inconsistent with the fact that if you trail or prefix the `since` and `before` fields with spaces, the values get trimmed successfully and converted to dates.
As a common approach I think that services should consistently trim spaces by default (maybe with some business-driven special cases) for all request fields and perform the validation after.

## Accepting new fields in the request
The GitHub API seems to allow injection of new json fields inside the requests. The `NewFieldsFuzzer` adds a new `catsField` inside a request, but GitHub API accepts it as valid.
This is again in contradiction with the [OWASP recommendation](https://cheatsheetseries.owasp.org/cheatsheets/REST_Security_Cheat_Sheet.html#input-validation) which suggests rejecting unexpected content.

## Doesn't make proper use of enumerated values
There are cases when it makes sense to use enums rather than free text. Some examples are the `gitignore_template` field or the `license_template` field, which are rejecting invalid values. They are obviously having a pre-defined list of values, but do not enforce this in any way in the contract.
Having them listed as enums will also make it easier to understand what are all supported templates for example.

# Fields are not making proper use of the format
There are 2 fields called `since` and `before` which seem to actually be a `date-time`, although in the OpenAPI contract they are only marked as `string` without any additional `format` information. In the description of the fields it states that this is actually an ISO date, but it will also be good to leverage [the features of OpenAPI](https://swagger.io/docs/specification/data-models/data-types/) and mark them accordingly.

![Date-Time Error](https://github.com/ludovicianul/ludovicianul.github.io/raw/master/images/date_time_github.png)

## Mismatch of validation between front-end driven calls and direct API calls
The `homepage` field seems to accept any values when doing a direct API call, but when you try to set this from the UI, you will get an error saying that you need to enter a valid URL. 

![Validation Error from the UI](https://github.com/ludovicianul/ludovicianul.github.io/raw/master/images/ui_error_github.png)

Having the same level of validation for backend and frontend is another good practice for making sure you don't end up with inconsistent data.

# Additional findings
There are some other failures which might seem debatable or not applicable:
- `description` accepts very large strings (50k characters sent by CATS), although the GitHub API doesn't actually have constraint information in the contract; again this is not necessarily a problem, but it's important for the contract to enforce constraints
- The `RecommendedHeadersFuzzer` expects a `CorrelationId/TraceId` to be defined in the headers, but this being a public API, it's not actually applicable
- The `CheckSecurityHeadersFuzzer` expects a `Cache-Control: no-store` as per the OWASP recommendations, but the endpoint does not operate critical data, so allowing caching of the information is fine

# Cleaning Up
**Before proceeding, please be careful to not delete your real repos**.
This is the script I've used to delete the repos. Run it incrementally and please check the `repos` file before deleting.

```shell
# get the latest 100 repos (by creation date)
curl -s "https://api.github.com/users/ludovicianul/repos?sort=created&direction=desc&per_page=100" | jq -r '.[].name' >> repos

# Maybe check them a bit before deleting them to make sure you don't delete real repos
for URL in `cat repos`; do echo $URL;  curl -X DELETE -H 'Authorization: token PERSONAL_TOKEN' https://api.github.com/repos/ludovicianul/$URL; done

# delete the file
rm repos

#start over until you delete everything
```

You need to run it several times as it deletes in batches of 100.

# Final Conclusions
`CATS` can be very powerful and can save lots of time while testing APIs. Because it tests every single field within a request, it's easy to spot inconsistencies or cases when the APIs are not as explicit as it should.
Although I only tested a single endpoint from the GitHub API, I suspect most the findings will apply to all the other endpoints.
Why not [give it a try](https://github.com/Endava/cats) for your API?
