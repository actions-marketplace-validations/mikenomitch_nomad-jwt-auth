# Nomad JWT Auth Action

A GitHub Action to mint a short-lived Nomad token using GHA identity tokens and arbitrary JWTs.

If you provide an input of "identityToken", that will be used as the JWT to authorize into Nomad. If you do not provide an "identityToken" argument, the Github Action Identity token will be used.

## Usage

See [action.yml](action.yml) for details.

In the GitHub Actions workflow, the workflow needs permissions to read contents
and write the ID token.

```yaml
jobs:
    retrieve-secret:
        permissions:
            contents: read
            id-token: write
```

Example:
```
env:
  PRODUCT_VERSION: "1.5.6"
  NOMAD_ADDR: "http://my-nomad-addr.foo:4646"

jobs:
  Check-Nomad-Status:
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read
    steps:
      - name: Checkout
        uses: actions/checkout@v3
      - name: Setup `nomad`
        uses: lucasmelin/setup-nomad@v1
        id: setup
        with:
          version: ${{ env.PRODUCT_VERSION }}
      - name: Run `nomad version`
        run: "nomad version"
      - name: Auth Into Nomad
        id: nomad-jwt-auth
        uses: mikenomitch/nomad-jwt-auth@v1
        with:
          url: ${{ env.NOMAD_ADDR }}
        continue-on-error: true
      - name: Get Status
        run: NOMAD_ADDR="${{ env.NOMAD_ADDR }}" NOMAD_TOKEN="${{ steps.nomad-jwt-auth.outputs.nomadToken }}" nomad status

```

## Setting up Simple Auth for Github Actions

#### Create an Auth Method

Let's set up a simple auth method in Nomad to test authenticating via Github Actions identity tokens.

First we'll need an Auth Method. This tells Nomad that we'll have a new way of authenticating new users. We'll call it `github` and say that it is a "JWT" based method for machine-to-machine auth, instead of OIDC for human-based SSO.

The contents in `./nomad-config/auth-method.json` say that we'll validate the signature of this JWT with the Github public key at JWKSURL. And we'll map the JWT's value of "repository_owner" to an internal value of "owner". This is important because for this auth method, we'll simply trust that if the owner is "mikenomitch", we'll grant a short lived management token (note: this is unrealistic and insecure).

Run the following to create an auth method for this action:
```
nomad acl auth-method create -name="github" -type="JWT" -max-token-ttl="23h" -token-locality=global -config "@nomad-config/auth-method.json"
```

The contents of `./nomad-config/auth-method.json`:
```
{
  "JWKSURL": "https://token.actions.githubusercontent.com/.well-known/jwks",
  "ExpirationLeeway": "1h",
  "ClockSkewLeeway": "1h",
  "ClaimMappings": {
    "repository_owner": "owner"
  }
}
```

Confirm that it works with: `nomad operator api /v1/acl/auth-method/github`

#### Create a Binding Rule

Next, we'll create a binding rule. This tells Nomad which values from a specified Auth Method should map to roles, polices, or management tokens in Nomad.

In our case, we'll keep it simple and say that if the repo owner is "mikenomitch", they're granted a management token.

Run the following to update the action for this action:
```
nomad acl binding-rule create \
    -description "repo to mgmt" \
    -auth-method "github" \
    -bind-type "management" \
    -selector "value.owner == mikenomitch"
```

#### Run the Github Action

Now when you run this github action. If you point to this cluster and have the correct owner specified, you will be authorized to access Nomad.

## Setting up a more realistic Auth Method for Github Actions

Let’s look at how we would set up a Authentication in Nomad to achieve the following rule:
`I want any repo in my account using an action called “Nomad JWT Auth” to get a Nomad ACL token that grants them permissions for all the Nomad policies assigned to a role for their repo. Tokens should only be valid for one hour, and the action should only be valid for the “main” branch.`

#### Create an auth method

Run the following to create an auth method for this new action:
```
nomad acl auth-method create -name="github-adv" -type="JWT" -max-token-ttl="1h" -token-locality=global -config "@nomad-config/auth-method-adv.json"
```

The contents of `./nomad-config/auth-method-adv.json`:
```
{
  "JWKSURL": "https://token.actions.githubusercontent.com/.well-known/jwks",
  "ExpirationLeeway": "1h",
  "ClockSkewLeeway": "1h",
  "ClaimMappings": {
    "repository_owner": "owner",
    "repository": "repo",
    "workflow": "workflow",
    "ref": "ref"
  }
}
```

Note that I am setting "max-token-ttl" to "1h" and getting the contents of "repository_owner", "repository", "workflow", and "ref" from the JWT.

Now I want to create a role for each repo that I want to use with this auth method. Let's pretend that the two repos I want to grant roles to are "mikenomitch/example-foo" and "mikenomitch/example-bar".

#### Create roles for each repo

I would create a roles for "example-foo" and "example-bar" with names matching their repo name, as I'll be mapping the JWT's repo name value to the role's name in the binding rule.

`nomad acl role create -name="mikenomitch/example-foo" -policy=example-acl-policy`

`nomad acl role create -name="mikenomitch/example-bar" -policy=some-other-acl-policy`

Notes:
* Policy creation not shown in this Readme for brevity
* You can use another value other than repo name from the JWT, hardcode a role name, or skip role and map directly to a policy.

#### Create a Binding Rule

Now, we'll create a binding rule which maps the JWT values to the Role.

Run the following to update the action for this action:
```
nomad acl binding-rule create \
    -description "repo name mapped to role name, on main branch, for “Nomad JWT Auth workflow" \
    -auth-method "github-adv" \
    -bind-name "${value.repo}" \
    -selector "value.owner == \"mikenomitch\" && value.workflow = \"Nomad JWT Auth\" && value.ref == \"refs/heads/main\""
```


### Notes

See the file `./jws-example.json` for an example of the fields availible in the JWT.

See the dir `./nomad-config` for some helpful files for testing local config with Nomad.

## Development

To make changes, edit `src/main.ts` and then run `npm run build`.

You can run main locally with `npm run build && node dist/index.js` which can be helpful for development & debugging.

## Reference

Here are all the inputs available through `with`:

Sure, here is the requested table:

| Input             | Description                                                                                                                                         | Default                      |
| ----------------- | --------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------- |
| `url`             | The URL of the Nomad cluster                                                                                                                        | `http://localhost:4646`     |
| `methodName`      | The name of the Nomad Auth Method to use                                                                                                            | `github`                     |
| `identityToken`   | The JWT identity token to use. Will use Github Action if not provided                                                                               |                              |
| `jwtGithubAudience` | The audience to use for the Github Action identity token. Only used if JWT not provided and using GHA JWT                                          |                              |
| `exportToken`     | Whether or not export Nomad token as environment variables                                                                                          | `true`                       |
| `outputToken`     | Whether or not to set the `nomadToken` output to contain the Nomad token after authentication                                                        | `true`                       |
| `tlsSkipVerify`   | When set to true, disables verification of the Nomad server certificate - setting this to true in production is not recommended                      | `false`                       |
| `caCertificate`   | Base64 encoded CA certificate to verify the Nomad server certificate                                                                                 |                              |
| `clientCertificate` | Base64 encoded client certificate for mTLS communication with the Nomad server                                                                      |                              |
| `clientKey`       | Base64 encoded client key for mTLS communication with the Nomad server                                                                              |                              |
| `extraHeaders`    | A string of newline separated extra headers to include on every request                                                                             |                              |
