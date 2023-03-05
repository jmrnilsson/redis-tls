# A dive into client support of Redis with self-signed certificates

## Description
A probe into self-signed certificate support for a few Redis clients: `ioredis`, `redis (node_redis)` and `redis (Python)`.
This is useful when Redis is used behind a firewall e.g. Kubernetes but without a service mesh, like
[Istio](https://istio.io/latest/about/service-mesh/) or [Linkerd](https://linkerd.io/), for mTLS.

## Background 
I don't know exactly what's going on here. But TLS support for redis doesn't appear to be a a priority. There is [this
post](http://antirez.com/news/96) from the author; then a [follow-up](http://antirez.com/news/118). There are some
pretty decent looking [performance measurements](https://github.com/redis/redis/issues/7595). There's the note about
lack of IO threading in [the official documentation](https://redis.io/docs/management/security/encryption/). And then 
some more ideas on connection pooling here (TBD). There is this [stale repo](https://github.com/josiahcarlson/redis-tls)
from where TLS probably reintegrated from once. And that team had a funny story that they aren't ready to be share yet.

Unfortunately, there are plenty invalid documentation floating around when it comes to client use. And there're troves
of unanswered questions asked in StackOverflow.com when it comes to redis and TLS. 

In conclusion, finding a client library can run self-signed certificates reliably isn't exactly easy. I reckon Redis
can be put as a sidecar with UNIX sockets. Another option, memcached, only has TLS as an
[experimental feature](https://github.com/memcached/memcached/wiki/TLS).

## Requirements
- Windows 10
- Python 3.10+
- Node v.18 LTS
- Powershell Core v.6+ (typically shipped with Windows 10+)
- WSL2

## Getting started
Open WSL terminal and run:

```zsh
sh ./setup.sh
```

## Running tests
This will run both Jasmine tests for redis (previously node_redis), ioredis and redis (Python). In PowerShell run:

```pwsh
.\tests.ps1
```

## Outcomes
### Used abbreviations
- **DOCS**: Certificates self-signed per official Redis documentation.
- **AZCA**: Certificates self-signed per Microsoft Azure custom root CA example.

### Test runs
Based on integration tests. Still hoping things will improve with some tweaking of certificate generation.

| Client | Version | Signed per | Status |
|------|------|------|------|
| `redis (node_redis)` | 4.6.5 | DOCS | ⛔ Invalid CN |
| `redis (node_redis)` | 4.6.5 | AZCA  | ⛔ DEPTH_ZERO_SELF_SIGNED_CERT |
| `ioredis` | 5.3.1 | DOCS | ⛔ Invalid CN |
| `ioredis` | 5.3.1 | AZCA   | ⛔ Self-sign failure |
| `redis (Python)` | 4.5.1 | DOCS | ✅ OK |
| `redis (Python)` | 4.5.1 | AZCA | ⛔ Self-sign failure |

Some common errors
- Hostname/IP does not match certificate's altnames: Host: localhost. is not cert's CN: Generic-cert
- 'Host: localhost. is not cert's CN: Generic-cert', host: 'localhost'.
- Certificate verify failed: self signed certificate (_ssl.c:1129).

## TL;DR
Out of the tested libraries it looks like `redis (Python)` is the only option that verifies a self-signed certificates.
The other libraries can be configured to accept them, but currently only by disabling certificate verification all
together.

Going deeper, there is a option in Nodejs to disable reject unauthorized TLS. It can in some cases be set to `false` via
a constructor argument. Alternatively it can be set through an environment variable which disables all certificate
verification for Nodejs rather than the Redis client.

Some integrations will not provide log information. To drill-down you may have to run `node .\debug.js` to capture log
output.