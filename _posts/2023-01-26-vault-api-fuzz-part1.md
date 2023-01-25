---
layout: post
title: I've fuzzed the Hashicorp's Vault API. Here are my findings (1)
---

# Setup

I'll be using [CATS](https://github.com/Endava/cats) to do API fuzzing. You can get a 1 minute intro to CATS here: [Get Started in 1 minute tutorial](https://endava.github.io/cats/docs/intro/).
The fuzzing will have 2 parts:
- part 1 (this article) is doing a blackbox fuzzing i.e. provide just access token without additional context
- part 2 (future article) will do more context-driven fuzzing i.e. I'll do some data setup before and do the fuzzing with additional context

[Vault](https://github.com/hashicorp/vault) is freshly installed and I don't have any setup or context for it.

I have [CATS installed](https://endava.github.io/cats/docs/intro/):

![cats](https://github.com/ludovicianul/ludovicianul.github.io/raw/master/images/cats-v.png)

I have [Vault installed](https://developer.hashicorp.com/vault/tutorials/getting-started/getting-started-install):

![cats](https://github.com/ludovicianul/ludovicianul.github.io/raw/master/images/vault-v.png)

I have the [Vault OpenAPI file](https://developer.hashicorp.com/vault/api-docs/system/internal-specs-openapi) saved as `api.json`.

Let's start Vault in dev mode:

```shell
vault server -dev
```

![cats](https://github.com/ludovicianul/ludovicianul.github.io/raw/master/images/vault-dev.png)

Export the Root Token as an environment variable:

```shell
export token=hvs.9eagj2vkhh7VXm40oUux5Dxw
```

According to the [Get Started in 1 minute tutorial](https://endava.github.io/cats/docs/intro/), we are now ready to run CATS.

# Run

Run CATS in blackbox mode:

```shell
cats --contract=api.json --server=http://localhost:8200/v1 -H "X-Vault-Token=$token" -b 
```

After 11 minutes we get the following results:

- 26314 successful tests (success in blackbox mode means not `500`)
- 165 errors (i.e. `500`)

![cats](https://github.com/ludovicianul/ludovicianul.github.io/raw/master/images/vault-r2.png)

# Findings
## Read timeouts
Many requests done on `/sys/pprof/profile` result in read timeout. It's not clear what happens, as there is nothing present in the logs.
To reproduce this, I can take any of these tests and do a replay: `cats replay Test31142`. 

## Socket hangups resulting in connection failures
Multiple requests are causing connection failures:

- a `POST` on `/sys/leases/lookup` with 
```json
{
  "lease_id": null
}
```

- a `POST` on `/sys/leases/` with
```json
{
  "url_lease_id": "mY77mev5F2AZ7Mbd",
  "increment": 2,
  "lease_id": null
}
```

- a `POST` on `/sys/leases/revoke` with
```json
{
  "url_lease_id": "uo6etjzzBKOsuN",
  "sync": true,
  "lease_id": null
}
```

The logs show the following:

```shell
 [INFO]  http: panic serving 127.0.0.1:57259: interface conversion: interface {} is nil, not string
goroutine 54353 [running]:
net/http.(*conn).serve.func1()
        /opt/hostedtoolcache/go/1.19.3/x64/src/net/http/server.go:1850 +0xbf
panic({0x595efa0, 0xc0019c49f0})
...
```

Same connection failure, but with a different cause, by doing a `DELETE` on `/identity/mfa/login-enforcement/2haK9t2O`. This is the log:

```shell
[INFO]  http: panic serving 127.0.0.1:57352: runtime error: invalid memory address or nil pointer dereference
goroutine 54900 [running]:
net/http.(*conn).serve.func1()
        /opt/hostedtoolcache/go/1.19.3/x64/src/net/http/server.go:1850 +0xbf
panic({0x58dcca0, 0xa521930})
        /opt/hostedtoolcache/go/1.19.3/x64/src/runtime/panic.go:890 +0x262
...
```

## Some confusing errors and behaviour

### Example 1

The `/identity/group/name/{name}` endpoint has some strange and inconsistent behaviour with confusing errors:

- I get `204` on a `DELETE` even though the `name` does not exist. I would expect a `404` here.
- When I do a `POST` with a full payload, like:
```json
{
  "member_group_ids": [
    "yEBjQD3UJvZn5ySGXTT",
    "yEBjQD3UJvZn5ySGXTT"
  ],
  "metadata": {
    "key": "value",
    "anotherKey": "anotherValue"
  },
  "policies": [
    "OOOOOOOOOOO",
    "OOOOOOOOOOO"
  ],
  "id": "071",
  "type": "y6R7Vt9",
  "member_entity_ids": [
    "8kGuW9WE2ww4xI8X9bRn",
    "8kGuW9WE2ww4xI8X9bRn"
  ]
}
```

I get a `400`: 
```json
{
    "errors": [
      "group type cannot be changed"
    ]
  }
```

But again, the group doesn't exist. I would expect a `404`. And if the `type` cannot be changed, why I'm able to send it in the request? 

And it gets weirder. When the `type` is removed by the `RemoveFieldsFuzzer`, I get a `500` error:
```json
{
    "errors": [
      "1 error occurred:\n\t* invalid entity ID \"8kGuW9WE2ww4xI8X9bRn\"\n\n"
    ]
  }
```

### Example 2
If I do a `POST` on `/sys/capabilities` with:

```json
{
  "path": [
    "DnsnmZ",
    "DnsnmZ"
  ],
  "paths": [
    "nxKLCAgp",
    "nxKLCAgp"
  ],
  "token": ""
}
```

I get a `500` with:

```json
{
    "errors": [
      "1 error occurred:\n\t* no token found\n\n"
    ]
}
```

But if send a value within the `token` field:

```json
{
  "path": [
    "DnsnmZ",
    "DnsnmZ"
  ],
  "paths": [
    "nxKLCAgp",
    "nxKLCAgp"
  ],
  "token": "ZUXO3Mk"
}
```

I get a `400`:
```json
{
    "errors": [
      "1 error occurred:\n\t* invalid token\n\n"
    ]
}
```

## Unexpected HTTP response codes
Multiple endpoints will return `500` instead of `4xx` (which seems more suitable for those cases). 
`500` should usually be reserved for unexpected behaviour during server side processing, rather than predictable business errors.

For example, a `POST` on `/sys/config/cors` with
```json
{
  "allowed_headers": [
    "AaiFkW79i80SuaXTFT0",
    "AaiFkW79i80SuaXTFT0"
  ],
  "enable": true
}
```

Will return a `500`: 
```json
{
    "errors": [
      "1 error occurred:\n\t* at least one origin or the wildcard must be provided\n\n"
    ]
}
```
For which is clearly a validation issue. 

A `GET` on `/internal/counters/activity/export` will return a `500`:
```json
{
    "errors": [
      "1 error occurred:\n\t* no data to export in provided time range\n\n"
    ]
  }
```

Bypassing authentication for a `GET` on `/sys/internal/ui/namespaces` will result in a `500` rather than a `403`:
```json
{
    "errors": [
      "client token empty"
    ]
}
```

A `GET` on `/sys/policies/password/FZhyI/..%20;/` will result in a `500`:
```json
{
    "errors": [
      "1 error occurred:\n\t* failed to retrieve password policy\n\n"
    ]
}
```

There are also a bunch of `500` cause by namespace not being found. Some examples: `cats replay Test37474`, `cats replay Test442`.

Most important issues are raised on GitHub: 
- https://github.com/hashicorp/vault/issues/18849
- https://github.com/hashicorp/vault/issues/18850
- https://github.com/hashicorp/vault/issues/18851
- https://github.com/hashicorp/vault/issues/18852










