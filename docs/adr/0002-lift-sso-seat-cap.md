# ADR-0002: Lift the free-tier SSO seat cap

- Status: Accepted
- Date: 2026-10-12
- Scope: LiteLLM Admin UI SSO sign-in

## Context

The Tranxmart proxy signs employees in to the Admin UI through the company identity
provider with generic OIDC SSO. Upstream refuses every SSO sign-in once the proxy holds
more than 5 billable users unless an enterprise license is set, and the proxy holds more
than 200 users:

```text
You must be a LiteLLM Enterprise user to use SSO for more than 5 users.
```

`_raise_if_sso_exceeds_free_user_limit` in `litellm/proxy/management_endpoints/ui_sso.py`
is the only place the cap is enforced. Both the OIDC (`/sso/key/generate`) and SAML
(`/saml/callback`) entry points go through it. The file is outside the `enterprise/`
directory, so it is under the MIT license.

## Decision

Make `_raise_if_sso_exceeds_free_user_limit` return without checking. Nothing else
changes, so API key auth, model calls and spend tracking are untouched.

## Consequences

SSO sign-in works for any number of users. Upgrading to a new upstream release means
re-applying this change on the new release branch, together with ADR-0001.
