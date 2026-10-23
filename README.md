You can configure both in Bitbucket Cloud, but they serve different purposes. For your staging Azure Function, use an access token to read and create pull requests through the REST API. Create an access key only if the Function App also needs to clone or pull repository code over SSH.

## A. Create an access token for reading and creating pull requests

Recommended for your Azure Function's REST API integration.

1

Navigate to the correct settings

Open the target repository in Bitbucket Cloud, then go to Repository settings → Security → Access tokens.

Your current screenshot is on Project settings → Security → Access tokens. Use that location instead if the token needs access to multiple repositories within the project; project access tokens require Premium.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://support.atlassian.com\&sz=32)

Atlassian Support

+1



2

Create the token

Click Create access token and enter:

- Name: `staging-azure-function-pr`
- Expiry: Select a date in accordance with your organization's security policy.

3

Select permissions

Pull requests → Write

Allows reading and listing PRs, as well as creating PRs. It also permits commenting, approving, declining, and merging pull requests. Grant this only if those additional operations are acceptable.

Repositories → Read (optional)

Add this if the Function App also needs repository information or other repository-read API endpoints.

Bitbucket's Pull requests: Read permission is sufficient when the function only needs to list and retrieve PRs. Write is required to create them.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://support.atlassian.com\&sz=32)

Atlassian Support

+1



4

Create and store the token

Click Create, copy the token once, and store it in Azure Key Vault. Configure the staging Function App to reference the secret.

Do not put the token in source code, Git, or application logs.

5

Call the REST API

For reading PRs:

```
GET https://api.bitbucket.org/2.0/repositories/{workspace}/{repo_slug}/pullrequests?state=OPEN
Authorization: Bearer <access-token>
```

To create a PR, send a `POST` request to the same endpoint with a JSON body containing the title, source branch, destination branch, and any other required fields.

Official guides: [Repository access tokens](https://support.atlassian.com/bitbucket-cloud/docs/repository-access-tokens/?utm_source=chatgpt.com) · [Access token permissions](https://support.atlassian.com/bitbucket-cloud/docs/repository-access-token-permissions/?utm_source=chatgpt.com).

## B. Create an access key for SSH repository access

Use this only if the Function App also needs to clone or pull repository code.

1

Generate an SSH key pair

Generate a dedicated key pair for the staging integration on the system that will use SSH.

Example, if your environment supports Ed25519:

```
ssh-keygen -t ed25519 -C "staging-azure-function"
```

Keep the private key secure; you will register only the public key with Bitbucket.

2

Open Access keys in Bitbucket

For one repository, navigate to Repository settings → Security → Access keys. For multiple repositories in a project, use the equivalent Project settings → Security → Access keys page shown in your screenshot.

3

Add the public key

Click Add key, enter a descriptive label, and paste the contents of your public key file, such as `id_ed25519.pub`.

The project access key grants read-only SSH access to repositories in that project. It is not a pull-request REST API credential.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://support.atlassian.com\&sz=32)

Atlassian Support

+1



4

Configure SSH in the Function App

Securely supply the private key to the process that needs to clone the repository, then use an SSH Git URL such as:

```
git clone git@bitbucket.org:<workspace>/<repo>.git
```

For Azure Functions, this requires deliberate handling of the private key and the SSH client. It is unnecessary if your function only calls the REST API.

Official guide: [Set up Project Access keys](https://support.atlassian.com/bitbucket-cloud/docs/set-up-project-access-keys-on-linux/?utm_source=chatgpt.com).

## Final recommendation

| Requirement                            | Credential                          |
| -------------------------------------- | ----------------------------------- |
| Read existing pull requests            | Access token — Pull requests: Read  |
| Read and create pull requests          | Access token — Pull requests: Write |
| Clone or pull repository code over SSH | Access key                          |
| Read/create PRs and clone code         | Both, with separate credentials     |

For your staging Function App, start with one repository-scoped access token with Pull requests: Write if creating PRs is genuinely required. If the function only reads PRs, use Read instead. This token is tied to the repository, not your personal user account.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://support.atlassian.com\&sz=32)

Atlassian Support

+1
