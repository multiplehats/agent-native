# Proposal: configurable public framework route prefix

I'm floating this for feedback before writing any code. Nothing here is implemented, and no runtime behavior changes today.

## Motivation

Agent-Native keeps framework routes clear of app-owned `/api/*` routes by putting them under `/_agent-native`, added in [#132](https://github.com/BuilderIO/agent-native/pull/132). That separation should stay, and it should stay the default.

A deployment may need framework endpoints under a namespace chosen by its gateway or reverse proxy. Right now the public namespace is baked into browser requests, authentication URLs, server mounts, and deployment output, so an inbound proxy rewrite on its own can't get the framework to emit a different namespace consistently. Apps end up carrying a dependency patch, or maintaining outbound URL rewrites next to their gateway config.

An optional, supported prefix would keep the app/framework split intact while letting a deployment choose its own routing convention. It stays a routing choice: the same authentication and authorization rules apply as under the default namespace.

## Proposed configuration

```ts
import { defineAgentNativeConfig } from "@agent-native/core/config";

export default defineAgentNativeConfig({
  runtime: {
    frameworkRoutePrefix: "/_agent-native",
  },
});
```

That example sets the existing default explicitly; leaving the field out behaves identically. Anything else, such as `/_platform`, is opt-in.

The field belongs on the public `AgentNativeConfig` surface. Its deployment alias follows the existing descriptor convention: `AGENT_NATIVE_CONFIG_RUNTIME_FRAMEWORK_ROUTE_PREFIX`. It uses the workspace, app, and environment resolution rules already in place, with no separate setter.

The value resolves once, at dev startup or deployment build, and is embedded into client and server output together. Changing it requires a restart or rebuild — I'd rather that than let the browser and the server read different runtime values.

The default stays `/_agent-native`. App-owned `/api/*` endpoints, page routes, and static assets keep their current paths. The app base path composes with the prefix: an app mounted at `/mail` with `/_platform` serves `/mail/_platform/actions/...`.

Validation, at least to start, accepts one absolute path segment of ASCII letters, digits, `_`, and `-`, containing at least one letter or digit. It rejects an empty value, `/`, trailing slashes, queries, fragments, escapes, dot segments, and reserved namespaces such as `/api`, `/mcp`, and `/.well-known`. A narrow shape keeps URL matching and security checks unambiguous; nested namespaces can be revisited on their own. Startup or build time should also check existing app routes and workspace mounts for collisions and report both owners in the diagnostic.

## Routing contract

Draw an explicit boundary between the public namespace and the internal one. Internally the framework can go on using `/_agent-native`, which leaves route registration and discovery untouched. A shared, browser-safe path module owns:

- The default/internal prefix and the validated configured public prefix.
- Segment-aware matching and conversion, including exact-root requests.
- Public URL construction with the app base path applied exactly once.

Incoming public framework requests map to the internal namespace before route selection, authentication classification, and CSRF checks. The transform has to preserve the query, method, headers, streaming body, and the original public URL wherever origin checks and OAuth validation need it. It must be idempotent, and it must not match paths like `/_platform-extra`.

The framework request handler already centralizes plugin mount matching, so it looks like the natural place to map public requests onto canonical mounts. Global auth and CSRF classification then have to agree with that mapping. The Vite dev gateway and the generated worker do their own app-base-path handling, and generated action routes skip the plugin mount shim, so both need equivalent treatment. Other deployment adapters need coverage too. A passing Vite browser build won't prove those adapters work.

Outgoing URLs have to go through the public path builder at the point where they're constructed:

- Client requests, event streams, uploads, navigation, and toolkit calls.
- Sign-in and reset documents, session endpoints, magic links, and auth redirects.
- Google, identity SSO, integration OAuth, MCP metadata, and callback validation.
- Agent-run continuation, scheduled jobs, health probes, and durable background dispatch that calls back into the same deployment.
- Generated function routing, redirects, adapter manifests, and CLI-generated URLs.

No rewriting of arbitrary HTML, JavaScript, JSON payloads, or response bodies. And a boundary rewrite plus the existing client helper isn't enough on its own while other outgoing surfaces still build the default path directly.

The option belongs to the current deployment. Its prefix must never be applied to another installation's URL or to a third-party service. Cross-app and cross-installation calls use the destination's declared endpoint, and destinations that advertise nothing keep the legacy discovery default. Discovery metadata and the bundled CLI/desktop clients both need an audit before we can call custom-prefix support complete.

## Compatibility and migration

With no configuration, public URLs, default helper results, authentication behavior, and deployment output all stay exactly as they are.

A custom prefix changes URLs that are registered elsewhere. Documentation has to cover OAuth redirect registrations, integration and webhook subscriptions, MCP connection URLs, external monitors, login and reset links already in the wild, and clients holding cached assets. Changing the value must not automatically rewrite stored third-party destinations.

Whether to keep the old public prefix alive during a migration is a maintainer call. If aliases are supported, they should be explicit and limited to framework routes, and both paths must get identical authentication and CSRF enforcement. Aliases must not silently shadow application routes, and an incoming POST body must never be redirected across namespaces. Shipping without aliases means a documented cutover and rollback procedure, links already sent to users included. With aliases off, the canonical internal namespace must not stay externally routable, and internal self-dispatch has to use the public builder or a separate internal transport.

## Implementation and acceptance criteria

Build the configuration descriptor and the shared path functions first, then wire the request boundaries and every outgoing surface in the same feature. Don't advertise the setting as supported while authentication or any deployment adapter still needs a dependency patch. Default behavior stays covered throughout.

| Area            | Required checks                                                                                                                                            |
| --------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Configuration   | Default; file/environment precedence; invalid types and paths; unknown keys; collisions.                                                                   |
| Path handling   | Root and nested requests; queries; similar prefixes; base-path composition; idempotence; external URLs unchanged.                                          |
| Clients         | Actions, upload, SSE, chat, and toolkit calls use the selected public path.                                                                                |
| Authentication  | Session and sign-in flows, reset/magic links, generated documents, callback construction and validation use the selected path.                             |
| Security        | Protected requests stay protected; unsafe cross-origin requests fail; malformed and encoded paths cannot bypass classification.                            |
| Server dispatch | Health probes, scheduled jobs, continuation, and background task handoff reach the configured public routes.                                               |
| Deployment      | Dev and each supported adapter serve action and discovered file routes under the selected prefix, including base-path mounts and reserved discovery paths. |
| Compatibility   | Default fixture behavior is unchanged; remote installations retain their own endpoints; any explicit aliases enforce the same policy.                      |

Dev checks need to include extension-bearing endpoints such as `.json` and images, which have to reach framework handlers rather than static-file middleware. Configured framework routes must never fall through to the SSR/static shell, and the existing framework `no-store` header rules have to follow the configured namespace.

At least one integration fixture should build and serve under a custom prefix and exercise an authenticated action and an event stream through the real request boundary. URL-builder unit tests alone don't establish routing support. Provider OAuth tests should assert the generated redirect URI and the callback validation without needing live provider credentials.

## Current code entry points

- [Public config and environment descriptors](../packages/core/src/config.ts)
- [Client path construction](../packages/core/src/client/api-path.ts)
- [Framework request handler and mount matching](../packages/core/src/server/framework-request-handler.ts)
- [Core route registration](../packages/core/src/server/core-routes-plugin.ts)
- [Authentication and public request URLs](../packages/core/src/server/auth.ts)
- [Better Auth base path and email links](../packages/core/src/server/better-auth-instance.ts)
- [Google OAuth URLs](../packages/core/src/server/google-oauth.ts)
- [Standalone sign-in client](../packages/core/src/client/auth/AuthPage.tsx)
- [CSRF route classification](../packages/core/src/server/csrf.ts)
- [Vite dev gateway and config injection](../packages/core/src/vite/client.ts)
- [Deployment entries and adapter output](../packages/core/src/deploy/build.ts)
- [File route discovery](../packages/core/src/deploy/route-discovery.ts)
- [Workspace deployment routing](../packages/core/src/deploy/workspace-deploy.ts)
- [Netlify framework cache headers](../packages/core/src/deploy/netlify-static-headers.ts)

## Decisions requested from maintainers

1. Is a separate public framework namespace a deployment case you want to support, and is `runtime.frameworkRoutePrefix` the right place to configure it?
2. Should internal route registration stay canonical with translation at the request boundary, or should every registration use the configured prefix?
3. Should a custom prefix come with an explicit migration-alias option, or should the first version require a coordinated cutover?
