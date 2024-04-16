---
pcx_content_type: how-to
title: Local development
weight: 3
layout: wide
---

# Local development

To test out your [dispatch Worker](/cloudflare-for-platforms/workers-for-platforms/reference/how-workers-for-platforms-works/#dynamic-dispatch-worker), [user Worker](/cloudflare-for-platforms/workers-for-platforms/reference/how-workers-for-platforms-works/#user-workers) and [outbound Worker](/cloudflare-for-platforms/workers-for-platforms/configuration/outbound-workers/) before deploying to production, you can use [Wrangler](/workers/wrangler) for development and testing.

{{<Aside>}}
Support for Workers for Platforms with `wrangler dev` in local mode is experimental and may change in the future. Use the prerelease branch: `wrangler@dispatch-namespaces-dev` to try out Workers for Platforms locally.
{{</Aside>}}

### 1. Create a user worker

```sh
$ npm create cloudflare@latest customer-worker-1
```

When following the interactive prompts, answer the questions as below:

- Select `no` to using TypeScript.
- **Select `no` to deploying your application.**

Update the `src/index.js` file for customer-worker-1

```javascript
---
filename: "src/index.js"
---
export default {
  async fetch(request) {
    // make a subrequest to the internet
    const response = await fetch('https://example.com');
    return new Response(`user worker got "${await response.text()}" from fetch`);
  }
}
```

Update the `wrangler.toml` file for customer-worker-1 and add the dispatch namespace:

```toml
---
filename: "wrangler.toml"
---
# ... other content above ...

dispatch_namespace = "my-namespace"
```

{{<Aside>}}
Note that the dispatch namespace used here does not have to be created using the [Wrangler command](/workers/wrangler/commands#dispatch-namespace) when using `wrangler dev` locally, but is required when deploying to production.
{{</Aside>}}

### 2. Create a dispatch worker

```sh
$ npm create cloudflare@latest dispatch-worker
```

When following the interactive prompts, answer the questions as below:

- Select `no` to using TypeScript.
- **Select `no` to deploying your application.**

Update the `src/index.js` file for dispatch-worker

```javascript
---
filename: "src/index.js"
---
export default {
  async fetch(request, env) {
    // get the user worker, specifying parameters that the outbound worker will see when it intercepts a user worker's subrequest
    const customerScript = env.DISPATCH_NAMESPACE.get(
      "customer-worker-1",
      {},
      {
        outbound: {
          paramCustomerName: 'customer-1'
        }
      }
    );
    // invoke user worker
    return await customerScript.fetch(request);
  }
}
```

Update the `wrangler.toml` file for dispatch-worker and add the dispatch namespace binding:

```toml
---
filename: "wrangler.toml"
---
# ... other content above ...

[[dispatch_namespaces]]
binding = "DISPATCH_NAMESPACE"
namespace = "my-namespace"
outbound = { service = "outbound-worker", parameters = ["paramCustomerName"] }
```

### 3. Create an outbound worker

```sh
$ npm create cloudflare@latest outbound-worker
```

When following the interactive prompts, answer the questions as below:

- Select `no` to using TypeScript.
- **Select `no` to deploying your application.**

Update the `src/index.js` file for outbound-worker

```javascript
---
filename: "src/index.js"
---
export default {
  async fetch(request, env) {
    const { paramCustomerName } = env;
    // use the paramters passed by the dispatcher to know what this user this request is for
    // and return custom content back to the user worker
    return new Response(`intercepted a request for ${paramCustomerName} by the outbound`);
  }
}
```

### 4. Start dev sessions for your workers

In separate terminals, start dev sessions for each of your workers:

For your dispatcher worker
```sh
$ cd dispatch-worker
$ npx wrangler@dispatch-namespaces-dev dev --port 8600
```

For your outbound worker
```sh
$ cd outbound-worker
$ npx wrangler@dispatch-namespaces-dev dev --port 8601
```

And for your user worker
```sh
$ cd customer-worker-1
$ npx wrangler@dispatch-namespaces-dev dev --port 8602
```

## 5. Test out requests

Make a request to your dispatcher worker!

```sh
$ curl http://localhost:8600
# -> user worker got "intercepted a request for customer-1 by the outbound" from fetch
```