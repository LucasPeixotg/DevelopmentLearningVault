---
tags: [api-design, sdk, versioning, best-practices]
created: 2026-06-03
---

# SDK versioning

> Summary: SDKs use Semantic Versioning (`MAJOR.MINOR.PATCH`) while APIs commonly use date-based versions. The two systems move independently — a client can pin an old SDK that silently calls an old API version, or manually override the API version in a new SDK. Best practice: pin the API version explicitly in the SDK constructor, use `@deprecated` annotations on SDK methods for IDE-level warnings, and keep the SDK `CHANGELOG.md` linked to the corresponding API version changelog.

## SemVer for SDKs

| Bump | When | API correlation |
|---|---|---|
| **MAJOR** | Breaking change in the SDK's public interface — method renames, removed methods, parameter changes | Usually corresponds to a major API version change |
| **MINOR** | New functionality, backward compatible — new methods for new endpoints, new response fields | New API endpoints or fields |
| **PATCH** | Bug fixes, no API surface change | Internal fixes |

```
stripe-node 12.0.0  →  targets Stripe API 2024-10-28
stripe-node 12.1.0  →  adds new endpoint methods, same API version
stripe-node 13.0.0  →  breaking SDK changes, targets new API version
```

The SDK `README` and release notes should explicitly state which API version each major SDK version targets.

## The alignment problem

SDK version and API version move independently. A client can be on any of these combinations:

| SDK version | API version | Problem |
|---|---|---|
| Current | Pinned (old) | Old API behaviour, new SDK surface |
| Old (pinned) | Old default | Both outdated — often the worst case |
| Current | Default (latest) | Breaking changes hit automatically on SDK upgrade |
| Current | Pinned (current) | Correct — explicit and deliberate |

The most dangerous case is a client pinned on an old SDK whose default API version is a deprecated one. When the deprecated API version is sunset, the client breaks — but the signal (deprecation headers) was only visible in the SDK's HTTP layer, which they never looked at.

## Pinning the API version in the SDK

**Always pin** the API version explicitly in the SDK constructor for production code. The SDK's default version is a footgun — it may change on SDK upgrade:

```typescript
// Wrong — API version upgrades silently with SDK upgrades
const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!);

// Right — explicit pin, upgrade deliberately
const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!, {
  apiVersion: '2024-10-28',
});
```

When upgrading the SDK major version, read the changelog for both the SDK and the API version it now defaults to.

## SDK deprecation annotations

The most effective deprecation signal is one that appears in the developer's IDE before they ever read a changelog. Use `@deprecated` JSDoc in TypeScript SDKs:

```typescript
/**
 * @deprecated Use `createCharge()` instead. This method will be
 * removed in sdk-name v13.0.0 (API sunset: 2026-03-31).
 * Migration: https://docs.example.com/migration/v1-v2
 */
createLegacyCharge(params: LegacyChargeParams): Promise<Charge> {
  console.warn(
    '[sdk-name] createLegacyCharge() is deprecated. ' +
    'Use createCharge() instead. See: https://docs.example.com/migration/v1-v2'
  );
  return this._request('POST', '/v1/charges', params);
}
```

- `@deprecated` shows in IDE hover tooltips and produces TypeScript compiler warnings with `noImplicitAny`.
- `console.warn()` appears in logs during development.
- Both include the migration URL so developers can act immediately.

## Coordinating SDK and API sunsets

When you sunset an API version, SDK versions that default to that API version will stop working. The coordination checklist:

```
1. Identify which SDK versions default to (or allow) the deprecated API version.
2. Release a new SDK MAJOR version that removes support for the old API version.
3. Deprecate old SDK major versions using @deprecated + console.warn.
4. Publish a SDK migration guide alongside the API migration guide.
5. Notify consumers of both the API sunset AND the required SDK upgrade.
6. In the SDK CHANGELOG.md, explicitly link to the API changelog for this change.
```

> [!warning] SDK upgrade ≠ API upgrade
> A developer who upgrades the SDK but doesn't update the pinned `apiVersion` is still on the old API version. And a developer who updates `apiVersion` without upgrading the SDK may hit SDK methods that don't support new response fields. Both must be updated together.

## Surfacing deprecation headers in the SDK

The SDK can detect and surface `Deprecation` and `Sunset` response headers automatically:

```typescript
// In the SDK's HTTP layer
private async _request(method: string, path: string, params: unknown) {
  const response = await fetch(/* ... */);

  if (response.headers.get('deprecation')) {
    const sunset = response.headers.get('sunset');
    console.warn(
      `[sdk-name] ${method} ${path} is deprecated.` +
      (sunset ? ` Sunset: ${sunset}.` : '') +
      ' See response Link header for migration docs.'
    );
  }

  return response.json();
}
```

This surfaces API-level deprecation in the developer's logs without requiring them to inspect raw HTTP headers manually.

## Common mistakes

> [!warning] Anti-patterns
> - **Not pinning `apiVersion` in production.** Silent upgrades on SDK update can break production.
> - **No `@deprecated` annotations in the SDK.** IDE warnings are the most actionable signal — use them.
> - **SDK and API changelogs that don't reference each other.** Always link them.
> - **Not warning on deprecated method use at runtime.** `@deprecated` only shows in IDEs; `console.warn` surfaces it in CI logs and runtime.
> - **Major SDK bump without a migration guide.** Same rules as API migration guides apply.

## References

- Semantic Versioning: https://semver.org/
- TypeScript `@deprecated` JSDoc: https://www.typescriptlang.org/docs/handbook/jsdoc-supported-types.html#deprecated
- Stripe SDK versioning: https://github.com/stripe/stripe-node#configuration
- `changesets` for monorepo SDK changelog management: https://github.com/changesets/changesets

## Related

- [[API Changelog]]
- [[API Migration guides]]
- [[API lifecycle communication]]
- [[API Versioning Strategies]]
- [[API Deprecation & Sunset]]
