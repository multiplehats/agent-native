# Proposal: configurable public framework route prefix

Status: proposed. This document does not add a configuration option or change routing behavior.

## Motivation

Agent-Native separates framework routes from app-owned `/api/*` routes with
`/_agent-native`, introduced in [#132](https://github.com/BuilderIO/agent-native/pull/132).
That separation should remain the default.

Deployments can also need to place framework endpoints under a namespace selected
by their gateway or reverse proxy. Today the public namespace is embedded in
browser requests, authentication URLs, server mounts, and deployment output. An
inbound proxy rewrite alone cannot make the framework consistently emit a different
namespace. Applications must carry a dependency patch or maintain outbound URL
rewrites alongside their gateway configuration.

A supported, optional public prefix would preserve the separation between app and
framework routes while letting deployments choose their own routing convention.
The prefix remains a routing choice, with the same authentication and authorization
requirements as the default namespace.

## Proposed configuration

```ts
import { defineAgentNativeConfig } from "@agent-native/core/config";

export default defineAgentNativeConfig({
  runtime: {
    frameworkRoutePrefix: "/_platform",
  },
});
```

The proposed field belongs to the public `AgentNativeConfig` surface. Its deployment
alias would follow the existing descriptor convention:
`AGENT_NATIVE_CONFIG_RUNTIME_FRAMEWORK_ROUTE_PREFIX`. It would use the existing
workspace, app, and environment resolution rules, without a separate setter.

The value would be resolved during development startup or deployment build and
embedded consistently into client and server output. Changing it would require a
restart or rebuild, rather than allowing the browser and server to independently
read different runtime values.

The default would remain `/_agent-native`. App-owned `/api/*` endpoints, page routes,
and static assets would keep their current paths. The existing app base path would
compose with the prefix: an app mounted at `/mail` with `/_platform` would expose
`/mail/_platform/actions/...`.

Initial validation should accept one absolute path segment made from ASCII letters,
digits, `_`, and `-`, with at least one letter or digit. Reject an empty value, `/`,
trailing slashes, queries, fragments, escapes, dot segments, and reserved namespaces
such as `/api` and `/.well-known`. A strict initial shape keeps URL matching and
security checks unambiguous; nested namespaces can be considered separately.
Existing app routes and workspace mounts must be checked for collisions at startup
or build time, with a diagnostic identifying both owners.

## Routing contract

Use an explicit boundary between the public namespace and the internal framework
namespace. The internal namespace can remain `/_agent-native`, keeping route
registration and discovery stable. A shared, browser-safe path module would own:

- The default/internal prefix and the validated configured public prefix.
- Segment-aware matching and conversion, including exact-root requests.
- Construction of public URLs with the app base path applied exactly once.

Incoming public framework requests would map to the internal namespace before
route selection, authentication classification, and CSRF checks. The transform must
preserve the query, method, headers, streaming body, and original public URL where
needed by origin checks and OAuth validation. Mapping must be idempotent and must
not match paths such as `/_platform-extra`.

The framework request handler centralizes plugin mount matching and is a candidate
for mapping public requests to canonical mounts. Global auth and CSRF classification
must agree with that mapping. The Vite dev gateway and generated worker also have
app-base-path handling; generated action routes bypass the plugin mount shim and
need equivalent treatment. Other deployment adapters need coverage too; support
cannot be inferred from the Vite browser bundle alone.

Outgoing URLs must use the public path builder at their point of construction:

- Client requests, event streams, uploads, navigation, and toolkit calls.
- Sign-in and reset documents, session endpoints, magic links, and auth redirects.
- Google, identity SSO, integration OAuth, MCP metadata, and callback validation.
- Agent-run continuation, scheduled jobs, health probes, and durable background
  dispatch that call back into the same deployment.
- Generated function routing, redirects, adapter manifests, and CLI-generated URLs.

Do not rewrite arbitrary HTML, JavaScript, JSON payloads, or response bodies. A
boundary rewrite plus the existing client helper is insufficient while other
outgoing surfaces still construct the default path directly.

The option belongs to the current deployment. Never apply its prefix to another
installation's URL or a third-party service. For cross-app or cross-installation
calls, use the destination's declared endpoint; keep legacy discovery defaults for
destinations that do not advertise an alternative. Discovery metadata and bundled
CLI/desktop clients must be audited before declaring custom-prefix support complete.

## Compatibility and migration

Without configuration, the public URLs, default helper results, authentication
behavior, and deployment output must stay unchanged.

A custom prefix changes externally registered URLs. Documentation must cover OAuth
redirect registrations, integration/webhook subscriptions, MCP connection URLs,
external monitors, existing login/reset links, and clients with cached assets.
Changing the value must not automatically rewrite stored third-party destinations.

Whether to retain the old public prefix during migration is a maintainer decision.
If aliases are supported, they should be explicit and bounded to framework routes;
both paths must receive identical authentication and CSRF enforcement. Aliases must
not silently shadow application routes, and incoming POST bodies must never be
redirected across namespaces. A deploy without aliases needs a documented cutover
and rollback procedure, including links already sent to users.

## Implementation and acceptance criteria

Implement the configuration descriptor and shared path functions first, then wire
request boundaries and all outgoing surfaces in the same feature. Do not advertise
the setting as supported while authentication or a deployment adapter still requires
a dependency patch. Keep default behavior covered throughout the migration.

| Area            | Required checks                                                                                                                       |
| --------------- | ------------------------------------------------------------------------------------------------------------------------------------- |
| Configuration   | Default; file/environment precedence; invalid types and paths; unknown keys; collisions.                                              |
| Path handling   | Root and nested requests; queries; similar prefixes; base-path composition; idempotence; external URLs unchanged.                     |
| Clients         | Actions, upload, SSE, chat, and toolkit calls use the selected public path.                                                           |
| Authentication  | Session and sign-in flows, reset/magic links, generated documents, callback construction and validation use the selected path.        |
| Security        | Protected requests stay protected; unsafe cross-origin requests fail; malformed and encoded paths cannot bypass classification.       |
| Server dispatch | Health probes, scheduled jobs, continuation, and background task handoff reach the configured public routes.                          |
| Deployment      | Dev plus each supported adapter emits and serves the selected routes, including app-base-path mounts and reserved discovery paths.    |
| Compatibility   | Default fixture behavior is unchanged; remote installations retain their own endpoints; any explicit aliases enforce the same policy. |

At least one integration fixture should build and serve with a custom prefix and
exercise an authenticated action and an event stream through the actual request
boundary. URL-builder unit tests alone do not establish routing support. Provider
OAuth tests should assert the generated redirect URI and callback validation without
requiring live provider credentials.

## Current code entry points

- [Public config and environment descriptors](../packages/core/src/config.ts)
- [Client path construction](../packages/core/src/client/api-path.ts)
- [Framework request handler and mount matching](../packages/core/src/server/framework-request-handler.ts)
- [Core route registration](../packages/core/src/server/core-routes-plugin.ts)
- [CSRF route classification](../packages/core/src/server/csrf.ts)
- [Vite dev gateway and config injection](../packages/core/src/vite/client.ts)
- [Deployment entries and adapter output](../packages/core/src/deploy/build.ts)

## Decisions requested from maintainers

1. Is a separate public framework namespace a supported deployment use case, and
   is `runtime.frameworkRoutePrefix` the right configuration surface?
2. Should internal route registration remain canonical with translation at the
   request boundary, or should all registrations use the configured prefix?
3. Should a custom prefix have an explicit migration-alias option, or should the
   first version require a coordinated cutover?
