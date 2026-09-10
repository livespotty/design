<!--
PROPOSAL WORKFLOW:
1. Copy this template to: proposals/000-<descriptive-name>.md
2. Fill in your proposal content
3. Open a PR on GitHub
4. Rename the file to use your PR number: proposals/<PR#>-<descriptive-name>.md
   Example: git mv proposals/000-my-feature.md proposals/105-my-feature.md
5. Update the heading below to include your PR number: # <PR#> - <Title>
6. Push the rename and updated title to your PR

See proposals/README.md for complete instructions.
-->

# 136 - Vault KMS: Kubernetes Authentication and Credentials Restructure

<!-- TOC -->
* [136 - Vault KMS: Kubernetes Authentication and Credentials Restructure](#136---vault-kms-kubernetes-authentication-and-credentials-restructure)
  * [Current situation](#current-situation)
  * [Motivation](#motivation)
  * [Proposal](#proposal)
    * [Grouped `credentials` configuration node](#grouped-credentials-configuration-node)
    * [Token Authentication (`credentials.token`)](#token-authentication-credentialstoken)
    * [Kubernetes Authentication (`credentials.kubernetes`)](#kubernetes-authentication-credentialskubernetes)
    * [Backward compatibility and deprecation of top-level `vaultToken`](#backward-compatibility-and-deprecation-of-top-level-vaulttoken)
    * [Java Configuration Schema](#java-configuration-schema)
    * [Request flow for Kubernetes authentication](#request-flow-for-kubernetes-authentication)
    * [Token lifecycle and refresh strategy](#token-lifecycle-and-refresh-strategy)
    * [End-to-end testing with Testcontainers and Minikube](#end-to-end-testing-with-testcontainers-and-minikube)
  * [Affected/not affected projects](#affectednot-affected-projects)
  * [Compatibility](#compatibility)
  * [Rejected alternatives](#rejected-alternatives)
<!-- TOC -->

This proposal introduces native Kubernetes authentication for the HashiCorp Vault KMS provider and restructures Vault KMS credentials under a grouped `credentials` configuration node, deprecating the legacy flat `vaultToken` configuration property.

## Current situation

The HashiCorp Vault KMS provider for the Record Encryption filter currently authenticates to Vault using a static or file-based token configured via the top-level `vaultToken` property on the `Config` record:

```yaml
kms: VaultKmsService
kmsConfig:
  vaultTransitEngineUrl: https://myhashicorpvault:8200/v1/transit
  vaultToken:
    passwordFile: /opt/vault/token
  tls:
    trust: ...
```

In the current implementation:
- `vaultToken` is a required `PasswordProvider` property on the `Config` record.
- The token is read during initialization and passed to Vault requests via the `X-Vault-Token` HTTP header.
- For long-lived deployments in Kubernetes, users must rely on external sidecars or cron jobs (such as Vault Agent) to refresh token files on disk, or manually administer periodic tokens.

## Motivation

1. **Native Kubernetes Workload Identity**: In Kubernetes, pods have access to a projected `ServiceAccount` JWT token mounted by kubelet. HashiCorp Vault provides a native [Kubernetes auth method](https://developer.hashicorp.com/vault/docs/auth/kubernetes), where a client submits its projected ServiceAccount JWT and a configured role name to `auth/<authPath>/login`, and Vault validates the token against the Kubernetes `TokenReview` API before issuing a temporary client token with a lease.
2. **Eliminate Static Token Management**: Requiring long-lived static tokens in Kubernetes clusters introduces security anti-patterns and operational burden. Using Kubernetes ServiceAccount tokens provides automatic, short-lived, verifiable authentication tied to the pod's identity.
3. **Consistency with Established KMS Design Patterns**: Proposals 017 and 018 restructured AWS KMS from flat mutually exclusive fields (`longTermCredentials`, `ec2MetadataCredentials`) to a grouped `credentials` node (`credentials.longTerm`, `credentials.webIdentity`, `credentials.podIdentity`). Applying the same pattern to HashiCorp Vault ensures consistent configuration across KMS providers and allows future Vault auth methods (e.g., AppRole, TLS certificates) to be added cleanly without top-level configuration bloat.

## Proposal

### Grouped `credentials` configuration node

Group all Vault authentication mechanisms under a `credentials` node in `kmsConfig`. Exactly one credential mechanism must be configured:

```yaml
kms: VaultKmsService
kmsConfig:
  vaultTransitEngineUrl: https://myhashicorpvault:8200/v1/transit
  credentials:
    token:
      token:
        passwordFile: /opt/vault/token
```

```yaml
kms: VaultKmsService
kmsConfig:
  vaultTransitEngineUrl: https://myhashicorpvault:8200/v1/transit
  credentials:
    kubernetes:
      role: kroxylicious-vault-role
      serviceAccountTokenPath: /var/run/secrets/kubernetes.io/serviceaccount/token
      authPath: kubernetes
```

### Token Authentication (`credentials.token`)

Configured via a dedicated `TokenCredentialsConfig` record:

```yaml
credentials:
  token:
    token:
      passwordFile: /opt/vault/token
      # or inline:
      # password: s.my-vault-token
```

Provides a `PasswordProvider` supplying the static or file-based Vault token.

### Kubernetes Authentication (`credentials.kubernetes`)

Configured via a dedicated `KubernetesCredentialsConfig` record:

```yaml
credentials:
  kubernetes:
    role: kroxylicious-vault-role
    serviceAccountTokenPath: /var/run/secrets/kubernetes.io/serviceaccount/token # optional
    authPath: kubernetes # optional
```

| Field | Description | Required? | Default |
|---|---|---|---|
| `role` | Vault role bound to the Kubernetes ServiceAccount. | Yes | - |
| `serviceAccountTokenPath` | Path to the projected Kubernetes ServiceAccount token file. | No | `/var/run/secrets/kubernetes.io/serviceaccount/token` |
| `authPath` | Mount path of the Kubernetes authentication method in Vault. | No | `kubernetes` |

### Backward compatibility and deprecation of top-level `vaultToken`

To ensure full backward compatibility:
1. The top-level `vaultToken` field is retained on `Config` and marked with `@Deprecated(since = "0.25.0", forRemoval = true)`.
2. The `Config` compact constructor transparently maps `vaultToken` into `credentials.token`.
3. If both `vaultToken` and `credentials` are specified, validation fails fast with an `IllegalArgumentException`.
4. Marking the deprecated `vaultToken` with `@JsonProperty(access = Access.WRITE_ONLY)` ensures that serializing or round-tripping configuration outputs only the modern `credentials` node.

```yaml
# Legacy style - continues to work, but deprecated
kms: VaultKmsService
kmsConfig:
  vaultTransitEngineUrl: https://myhashicorpvault:8200/v1/transit
  vaultToken:
    passwordFile: /opt/vault/token
```

### Java Configuration Schema

```java
public record Config(
    @JsonProperty(value = "vaultTransitEngineUrl", required = true) URI vaultTransitEngineUrl,
    @Deprecated(since = "0.25.0", forRemoval = true) @JsonProperty(value = "vaultToken", required = false, access = Access.WRITE_ONLY) @Nullable PasswordProvider vaultToken,
    @JsonProperty(value = "credentials", required = false) @Nullable VaultCredentialsConfig credentials,
    @Nullable Tls tls) {

    public Config {
        Objects.requireNonNull(vaultTransitEngineUrl);
        if (vaultToken != null && credentials != null) {
            throw new IllegalArgumentException("Cannot specify both 'vaultToken' and 'credentials' - use 'credentials.token' instead");
        }
        if (vaultToken == null && credentials == null) {
            throw new IllegalArgumentException("Either 'credentials' or deprecated 'vaultToken' must be provided");
        }
        if (credentials == null) {
            credentials = new VaultCredentialsConfig(new TokenCredentialsConfig(vaultToken), null);
        }
    }
}

public record VaultCredentialsConfig(
    @JsonProperty("token") @Nullable TokenCredentialsConfig token,
    @JsonProperty("kubernetes") @Nullable KubernetesCredentialsConfig kubernetes) {

    public VaultCredentialsConfig {
        if (token == null && kubernetes == null) {
            throw new IllegalArgumentException("Either 'token' or 'kubernetes' credentials must be provided");
        }
        if (token != null && kubernetes != null) {
            throw new IllegalArgumentException("Only one of 'token' or 'kubernetes' credentials may be provided");
        }
    }
}

public record TokenCredentialsConfig(
    @JsonProperty(value = "token", required = true) PasswordProvider token) {
    public TokenCredentialsConfig {
        Objects.requireNonNull(token, "token must not be null");
    }
}

public record KubernetesCredentialsConfig(
    @JsonProperty(value = "role", required = true) String role,
    @JsonProperty(value = "serviceAccountTokenPath", required = false) @Nullable String serviceAccountTokenPath,
    @JsonProperty(value = "authPath", required = false) @Nullable String authPath) {

    public KubernetesCredentialsConfig {
        Objects.requireNonNull(role, "role must not be null");
        if (serviceAccountTokenPath == null) {
            serviceAccountTokenPath = "/var/run/secrets/kubernetes.io/serviceaccount/token";
        }
        if (authPath == null) {
            authPath = "kubernetes";
        }
    }
}
```

### Request flow for Kubernetes authentication

1. **Read projected token**: When acquiring a token, the provider reads the ServiceAccount JWT from `serviceAccountTokenPath` (re-reading on refresh so token rotations by kubelet are observed).
2. **Login to Vault**: An HTTP POST is dispatched to `/v1/auth/<authPath>/login` on the same Vault host as `vaultTransitEngineUrl`. The provider derives the scheme and authority from `vaultTransitEngineUrl`; for example, `https://myhashicorpvault:8200/v1/transit` results in `https://myhashicorpvault:8200/v1/auth/kubernetes/login`:
   ```json
   {
     "jwt": "<service-account-jwt>",
     "role": "<role>"
   }
   ```
3. **Parse auth response**: Vault validates the JWT with Kubernetes and returns a JSON response containing `auth.client_token` and `auth.lease_duration`.
4. **Cache and pre-emptive refresh**: The returned `client_token` is used for `X-Vault-Token` headers. The token is refreshed before expiry based on `lease_duration`.

### Token lifecycle and refresh strategy

- Vault tokens issued by the Kubernetes auth method have a finite TTL (`lease_duration`).
- A percentage-based refresh threshold (80% of `lease_duration`, leaving a 20% safety window before hard expiry) is used to trigger background re-authentication.
- Concurrent requests share the cached or in-flight `CompletableFuture<String>` to prevent duplicate login calls.
- Non-successful HTTP responses and responses missing `auth.client_token` or a positive `auth.lease_duration` fail the login attempt with an error containing Vault's response status and message when available.
- If a refresh fails while the cached token is still valid, requests continue using that token and the refresh is retried with backoff. Once the cached token expires, requests fail until re-authentication succeeds.

### End-to-end testing with Testcontainers and Minikube

1. **Integration tests (`VaultKmsKubernetesAuthIT`)**:
   - Uses `Testcontainers` to start a Vault container.
   - Spins up a WireMock server stubbing the Kubernetes TokenReview API (`/apis/authentication.k8s.io/v1/tokenreviews`).
   - Generates RSA key pairs and signed JWTs using `jose4j` to simulate the Kubernetes ServiceAccount token.
    - Configures Vault with Kubernetes auth pointing to the WireMock stub via `Testcontainers.exposeHostPorts(...)`, including the Kubernetes API host, CA certificate, JWT issuer, and reviewer JWT.
    - Configures a Vault Kubernetes auth role binding the test JWT's ServiceAccount namespace and name, and stubs a successful `TokenReview` response for that JWT.
    - Verifies the returned Vault client token, lease duration, refresh after expiry, and failure behavior for rejected or malformed login responses.
   - Annotated with `@EnabledIf(value = "isDockerAvailable", disabledReason = "docker unavailable")`.
2. **Minikube deployment verification**:
   - Verified end-to-end on Minikube with a live Vault pod, ClusterRoleBinding for `system:auth-delegator`, and a dedicated `ServiceAccount` for Kroxylicious.

## Affected/not affected projects

**Affected:**
- `kroxylicious-kms-providers/kroxylicious-kms-provider-hashicorp-vault`:
  - New config records: `VaultCredentialsConfig`, `TokenCredentialsConfig`, `KubernetesCredentialsConfig`
  - Updated `Config` record with deprecation handling
  - New token providers: `VaultTokenProvider`, `KubernetesTokenProvider`, `StaticTokenProvider`
  - Updated `VaultKmsService` and `VaultKms`
- `kroxylicious-kms-providers/kroxylicious-kms-provider-hashicorp-vault-test-support`: test fixtures and facades
- `kroxylicious-docs`: updated Vault setup instructions and configuration examples

**Not affected:**
- Other KMS providers (`aws-kms`, `azure-key-vault-kms`, `fortanix-dsm`, `inmemory`)
- Core Kroxylicious runtime and filter APIs
- Kroxylicious Kubernetes Operator

## Compatibility

- **Backward compatible**: Existing deployments using `vaultToken` at the root of `kmsConfig` continue to operate without change.
- **Deprecation notice**: `vaultToken` is marked as `@Deprecated(since = "0.25.0", forRemoval = true)`. A clear error is thrown if both `vaultToken` and `credentials` are specified.
- **Forward extensible**: Additional Vault authentication engines (e.g., AppRole, TLS certificates) can be introduced as new fields on `VaultCredentialsConfig` without altering `Config`.

## Rejected alternatives

1. **Flat top-level properties on `Config`**:
   Placing `role`, `serviceAccountTokenPath`, and `authPath` directly on `Config` alongside `vaultToken`.
   *Rejected because*: It creates confusing mutual exclusivity between top-level fields and diverges from the structured `credentials` pattern established in AWS KMS (proposals 017/018).

2. **Relying solely on external sidecars (Vault Agent Injector)**:
   Requiring users in Kubernetes to run a Vault Agent sidecar to populate a token file, keeping only `vaultToken.passwordFile`.
   *Rejected because*: Sidecars add pod startup latency, resource overhead, and operational complexity. Native in-process authentication provides a frictionless Kubernetes experience.
