<!-- BEGIN MUNGE: UNVERSIONED_WARNING -->

<!-- BEGIN STRIP_FOR_RELEASE -->

<img src="http://kubernetes.io/img/warning.png" alt="WARNING"
     width="25" height="25">
<img src="http://kubernetes.io/img/warning.png" alt="WARNING"
     width="25" height="25">
<img src="http://kubernetes.io/img/warning.png" alt="WARNING"
     width="25" height="25">
<img src="http://kubernetes.io/img/warning.png" alt="WARNING"
     width="25" height="25">
<img src="http://kubernetes.io/img/warning.png" alt="WARNING"
     width="25" height="25">

<h2>PLEASE NOTE: This document applies to the HEAD of the source tree</h2>

If you are using a released version of Kubernetes, you should
refer to the docs that go with that version.

<!-- TAG RELEASE_LINK, added by the munger automatically -->
<strong>
The latest release of this document can be found
[here](http://releases.k8s.io/release-1.1/docs/admin/authorization.md).

Documentation for other releases can be found at
[releases.k8s.io](http://releases.k8s.io).
</strong>
--

<!-- END STRIP_FOR_RELEASE -->

<!-- END MUNGE: UNVERSIONED_WARNING -->

# Authorization Plugins


In Kubernetes, authorization happens as a separate step from authentication.
See the [authentication documentation](authentication.md) for an
overview of authentication.

Authorization applies to all HTTP accesses on the main (secure) apiserver port.

The authorization check for any request compares attributes of the context of
the request, (such as user, resource, and namespace) with access
policies.  An API call must be allowed by some policy in order to proceed.

The following implementations are available, and are selected by flag:
  - `--authorization-mode=AlwaysDeny`
  - `--authorization-mode=AlwaysAllow`
  - `--authorization-mode=ABAC`

`AlwaysDeny` blocks all requests (used in tests).
`AlwaysAllow` allows all requests; use if you don't need authorization.
`ABAC` allows for user-configured authorization policy.  ABAC stands for Attribute-Based Access Control.

## ABAC Mode

### Request Attributes

A request has the following attributes that can be considered for authorization:
  - user (the user-string which a user was authenticated as).
  - group (the list of group names the authenticated user is a member of).
  - whether the request is for an API resource.
  - the request path.
    - allows authorizing access to miscellaneous endpoints like `/api` or `/healthz` (see [kubectl](#kubectl)).
  - the request verb.
    - API verbs like `get`, `list`, `create`, `update`, and `watch` are used for API requests
    - HTTP verbs like `get`, `post`, and `put` are used for non-API requests
  - what resource is being accessed (for API requests only)
  - the namespace of the object being accessed (for namespaced API requests only)
  - the API group being accessed (for API requests only)

We anticipate adding more attributes to allow finer grained access control and
to assist in policy management.

### Policy File Format

For mode `ABAC`, also specify `--authorization-policy-file=SOME_FILENAME`.

The file format is [one JSON object per line](http://jsonlines.org/).  There should be no enclosing list or map, just
one map per line.

Each line is a "policy object".  A policy object is a map with the following properties:
  - Versioning properties:
    - `apiVersion`, type string; valid values are "abac.authorization.kubernetes.io/v1beta1". Allows versioning and conversion of the policy format.
    - `kind`, type string: valid values are "Policy". Allows versioning and conversion of the policy format.

  - `spec` property set to a map with the following properties:
    - Subject-matching properties:
      - `user`, type string; the user-string from `--token-auth-file`. If you specify `user`, it must match the username of the authenticated user. `*` matches all requests.
      - `group`, type string; if you specify `group`, it must match one of the groups of the authenticated user. `*` matches all requests.

    - `readonly`, type boolean, when true, means that the policy only applies to get, list, and watch operations.

    - Resource-matching properties:
      - `apiGroup`, type string; an API group, such as `extensions`. `*` matches all API groups.
      - `namespace`, type string; a namespace string. `*` matches all resource requests.
      - `resource`, type string; a resource, such as `pods`. `*` matches all resource requests.

    - Non-resource-matching properties:
    - `nonResourcePath`, type string; matches the non-resource request paths (like `/version` and `/apis`). `*` matches all non-resource requests. `/foo/*` matches `/foo/` and all of its subpaths.

An unset property is the same as a property set to the zero value for its type (e.g. empty string, 0, false).
However, unset should be preferred for readability.

In the future, policies may be expressed in a JSON format, and managed via a REST interface.

### Authorization Algorithm

A request has attributes which correspond to the properties of a policy object.

When a request is received, the attributes are determined.  Unknown attributes
are set to the zero value of its type (e.g. empty string, 0, false).

A property set to "*" will match any value of the corresponding attribute.

The tuple of attributes is checked for a match against every policy in the policy file.
If at least one line matches the request attributes, then the request is authorized (but may fail later validation).

To permit any user to do something, write a policy with the user property set to "*".
To permit a user to do anything, write a policy with the apiGroup, namespace, resource, and nonResourcePath properties set to "*".

### Kubectl

Kubectl uses the `/api` and `/apis` endpoints of api-server to negotiate client/server versions. To validate objects sent to the API by create/update operations, kubectl queries certain swagger resources. For API version `v1` those would be `/swaggerapi/api/v1` & `/swaggerapi/experimental/v1`.

When using ABAC authorization, those special resources have to be explicitly exposed via the `nonResourcePath` property in a policy (see [examples](#examples) below):

* `/api`, `/api/*`, `/apis`, and `/apis/*` for API version negotiation.
* `/version` for retrieving the server version via `kubectl version`.
* `/swaggerapi/*` for create/update operations.

To inspect the HTTP calls involved in a specific kubectl operation you can turn up the verbosity:

    kubectl --v=8 version

### Examples

 1. Alice can do anything to all resources:                  `{"apiVersion": "abac.authorization.kubernetes.io/v1beta1", "kind": "Policy", "spec": {"user": "alice", "namespace": "*", "resource": "*", "apiGroup": "*"}}`
 2. Kubelet can read any pods:                               `{"apiVersion": "abac.authorization.kubernetes.io/v1beta1", "kind": "Policy", "spec": {"user": "kubelet", "namespace": "*", "resource": "pods", "readonly": true}}`
 3. Kubelet can read and write events:                       `{"apiVersion": "abac.authorization.kubernetes.io/v1beta1", "kind": "Policy", "spec": {"user": "kubelet", "namespace": "*", "resource": "events"}}`
 4. Bob can just read pods in namespace "projectCaribou":    `{"apiVersion": "abac.authorization.kubernetes.io/v1beta1", "kind": "Policy", "spec": {"user": "bob", "namespace": "projectCaribou", "resource": "pods", "readonly": true}}`
 5. Anyone can make read-only requests to all non-API paths: `{"apiVersion": "abac.authorization.kubernetes.io/v1beta1", "kind": "Policy", "spec": {"user": "*", "readonly": true, "nonResourcePath": "*"}}`

[Complete file example](http://releases.k8s.io/HEAD/pkg/auth/authorizer/abac/example_policy_file.jsonl)

### A quick note on service accounts

A service account automatically generates a user. The user's name is generated according to the naming convention:

```
system:serviceaccount:<namespace>:<serviceaccountname>
```

Creating a new namespace also causes a new service account to be created, of this form:*

```
system:serviceaccount:<namespace>:default
```

For example, if you wanted to grant the default service account in the kube-system full privilege to the API, you would add this line to your policy file:

```json
{"apiVersion":"abac.authorization.kubernetes.io/v1beta1","kind":"Policy","user":"system:serviceaccount:kube-system:default","namespace":"*","resource":"*","apiGroup":"*"}
```

The apiserver will need to be restarted to pickup the new policy lines.

## Plugin Development

Other implementations can be developed fairly easily.
The APIserver calls the Authorizer interface:

```go
type Authorizer interface {
  Authorize(a Attributes) error
}
```

to determine whether or not to allow each API action.

An authorization plugin is a module that implements this interface.
Authorization plugin code goes in `pkg/auth/authorizer/$MODULENAME`.

An authorization module can be completely implemented in go, or can call out
to a remote authorization service.  Authorization modules can implement
their own caching to reduce the cost of repeated authorization calls with the
same or similar arguments.  Developers should then consider the interaction between
caching and revocation of permissions.


<!-- BEGIN MUNGE: GENERATED_ANALYTICS -->
[![Analytics](https://kubernetes-site.appspot.com/UA-36037335-10/GitHub/docs/admin/authorization.md?pixel)]()
<!-- END MUNGE: GENERATED_ANALYTICS -->
