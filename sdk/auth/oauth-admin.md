# OAuth Admin

**Priority:** LOW
**Methods:** 6
**Status:** Complete

OAuth client management for configuring OAuth 2.1 providers. Server-side only — requires `service_role` key. Only relevant when OAuth 2.1 server is enabled in Supabase Auth.

---

## Methods (Brief Format)

#### `listClients(params?)`
**Priority:** Low | **Complexity:** Simple

**Signature:** `listClients(params?: PageParams): Promise<OAuthClientListResponse>`
**Purpose:** Lists all registered OAuth clients with optional pagination.
**Usage:** `const { data, error } = await supabase.auth.admin.oauth.listClients({ page: 1, perPage: 20 })`
**Implementation:** GET `/admin/oauth/clients`. Parses pagination from `x-total-count` and `link` headers. Returns `{ clients, aud, nextPage, lastPage, total }`.
**Use Case:** Admin dashboard listing all OAuth applications registered with your auth server.

---

#### `createClient(params)`
**Priority:** Low | **Complexity:** Simple

**Signature:** `createClient(params: CreateOAuthClientParams): Promise<OAuthClientResponse>`
**Purpose:** Registers a new OAuth client with the auth server.
**Usage:**
```ts
const { data, error } = await supabase.auth.admin.oauth.createClient({
  client_name: 'My App',
  redirect_uris: ['https://myapp.com/callback'],
})
// data.client_secret only returned on creation — store it securely
```
**Implementation:** POST `/admin/oauth/clients`. `client_secret` is only included in the response at creation time. Optional fields: `client_uri`, `grant_types`, `response_types`, `scope`.
**Use Case:** Registering a new third-party application to authenticate via your OAuth 2.1 server.

---

#### `getClient(clientId)`
**Priority:** Low | **Complexity:** Simple

**Signature:** `getClient(clientId: string): Promise<OAuthClientResponse>`
**Purpose:** Retrieves details of a specific OAuth client by ID.
**Usage:** `const { data, error } = await supabase.auth.admin.oauth.getClient('client-uuid')`
**Implementation:** GET `/admin/oauth/clients/{clientId}`. Returns `OAuthClient` without `client_secret`.
**Use Case:** Viewing configuration details for a specific registered OAuth application.

---

#### `updateClient(clientId, params)`
**Priority:** Low | **Complexity:** Simple

**Signature:** `updateClient(clientId: string, params: UpdateOAuthClientParams): Promise<OAuthClientResponse>`
**Purpose:** Updates an existing OAuth client's configuration.
**Usage:**
```ts
const { data, error } = await supabase.auth.admin.oauth.updateClient('client-uuid', {
  client_name: 'Updated Name',
  redirect_uris: ['https://newdomain.com/callback'],
})
```
**Implementation:** PUT `/admin/oauth/clients/{clientId}`. All fields optional: `client_name`, `client_uri`, `logo_uri`, `redirect_uris`, `grant_types`.
**Use Case:** Updating redirect URIs or display name for an existing OAuth application.

---

#### `deleteClient(clientId)`
**Priority:** Low | **Complexity:** Simple

**Signature:** `deleteClient(clientId: string): Promise<{ data: null; error: AuthError | null }>`
**Purpose:** Deletes an OAuth client registration.
**Usage:** `const { data, error } = await supabase.auth.admin.oauth.deleteClient('client-uuid')`
**Implementation:** DELETE `/admin/oauth/clients/{clientId}`. Returns `{ data: null }` on success. Revokes all associated tokens/grants.
**Use Case:** Removing a decommissioned third-party application from your auth server.

---

#### `regenerateClientSecret(clientId)`
**Priority:** Low | **Complexity:** Simple

**Signature:** `regenerateClientSecret(clientId: string): Promise<OAuthClientResponse>`
**Purpose:** Regenerates the client secret for a confidential OAuth client.
**Usage:**
```ts
const { data, error } = await supabase.auth.admin.oauth.regenerateClientSecret('client-uuid')
// data.client_secret contains the new secret — store it securely
```
**Implementation:** POST `/admin/oauth/clients/{clientId}/regenerate_secret`. Invalidates previous secret immediately. New `client_secret` returned in response.
**Use Case:** Rotating credentials after a suspected secret compromise or as part of routine security policy.

---

## Key Types

```ts
type OAuthClient = {
  client_id: string
  client_name: string
  client_secret?: string              // only on create/regenerate
  client_type: 'public' | 'confidential'
  token_endpoint_auth_method: string
  registration_type: 'dynamic' | 'manual'
  client_uri?: string
  logo_uri?: string
  redirect_uris: string[]
  grant_types: ('authorization_code' | 'refresh_token')[]
  response_types: ('code')[]
  scope?: string
  created_at: string
  updated_at: string
}

type CreateOAuthClientParams = {
  client_name: string
  client_uri?: string
  redirect_uris: string[]
  grant_types?: ('authorization_code' | 'refresh_token')[]
  response_types?: ('code')[]
  scope?: string
}

type UpdateOAuthClientParams = {
  client_name?: string
  client_uri?: string
  logo_uri?: string
  redirect_uris?: string[]
  grant_types?: ('authorization_code' | 'refresh_token')[]
}

type PageParams = { page?: number; perPage?: number }
```

---

## Summary

All 6 methods are simple CRUD operations over REST endpoints at `/admin/oauth/clients`. Server-side only — never expose `service_role` key in browser. `client_secret` only returned on `createClient` and `regenerateClientSecret` calls. Requires OAuth 2.1 server to be enabled in Supabase Auth configuration.
