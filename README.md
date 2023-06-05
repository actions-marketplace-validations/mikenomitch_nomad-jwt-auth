A GitHub Action to mint a short-lived Nomad token using GHA identity tokens

https://github.com/hashicorp/vault-action#jwt-with-github-oidc-tokensjwtPrivateKey

https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/configuring-openid-connect-in-hashicorp-vault


Run the following to create an auth method for this action:
`nomad acl auth-method create -name="github" -type="JWT" -max-token-ttl="23h" -token-locality=global -config "@config.json"`

Run the following to update the action for this action:
`nomad acl auth-method update -type="JWT" -max-token-ttl="23h" -token-locality=global -config "@config.json" github`

Confirm that it works with:
`nomad operator api /v1/acl/auth-method/github`