# Proxmox Authentication

## Adding an OIDC provider (Auth0)

1. Create an application in Auth0
2. Configure the application in Auth0 (set the login and callback URIs to the root URL of the Proxmox instance)
3. Create a new oidc realm in Proxmox. The `username-claim` flag should be set to the claim that will be used as the username in Proxmox. This can be any claim returned by the OIDC provider, usually `sub`, `username`, `email` or `nickname`.

```shell
pveum realm add auth0-oidc --type openid --issuer-url  https://*yourinstance*.eu.auth0.com --client-id *clientId* --client-key *clientSecret* --username-claim nickname
```

4. Create a user in the new realm. The name of the user should be the value of the selected claim name in Auth0 (in this case, `nickname`). The user can be added to groups by passing the optional `-groups` flag.

```shell
pveum user add testuser@auth0-oidc [-groups group1,group2]
```

5. The new realm should appear on the login screen, and the user should be able to log in using the Auth0 credentials.
