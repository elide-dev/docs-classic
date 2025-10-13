# Secrets

Elide has built-in secret management with multi-user remote management and escrow. Secrets are always encrypted when 
stored or transferred, and will only ever be plaintext in memory at runtime. The remote may currently either be within 
project sources or in a separate private GitHub repository, but support for other storage providers may be coming 
depending on interest.

## Concepts

There are a couple of core concepts to understand in how Elide handles secrets.

- Secret:
  - A piece of secret data
  - May be text or binary
  - May be retrieved in all guest languages in Elide runtime
  - Text secrets may be registered as environment variables that are only visible within the runtime
- Local files:
  - Files that are stored locally on a single machine
  - Should never be shared
- Remote files:
  - Files that are shared between all users
  - May exist locally if stored with project sources
- Secret profile:
  - A container of an arbitrary number of secrets
  - Encrypted with a randomly generated key

## Local secret management

This part of the guide covers the local part of managing secrets in Elide.

### Initialization

To start, run `elide secrets`. This is an interactive command line utility for managing secrets. On first launch, you 
get the following prompts:

1. How do you want to encrypt locally stored secrets?
   - Passphrase: Encrypt local secrets with a passphrase
   - GPG: Encrypt local secrets with GnuPG. Please note that this method expects to find the `gpg` binary in `PATH`. Only private keys in your default gpg configuration will be available
2. Do you want to initialize a new project or pull an existing project?
   - Initialize a new project: Select this option for now
   - Pull an existing project: Multi-user management will be covered later in this guide
3. What is the name of your project?
   - Enter the project name. This defaults to the project name in `elide.pkl`, or the name of the current directory if 
no manifest is present

After this, you are presented with the main menu.

### Main menu

The main menu has the following options:

- Create a new profile
- List profiles
- Select a profile to edit
- Remove a profile
- Change encryption details
- Pull changes from remote
- Push changes to remote
- Manage remote as a superuser
- Exit secret management

Most of these options are self-explanatory. Remote operations are covered later in this guide.

### Editing a profile

When selecting the `Select a profile to edit` option, you are asked which profile you want to edit. This list contains 
all existing profiles, as well as `Local secrets`. This last option allows you to edit the internal secrets, which can
at worst break the secrets setup and render some secrets irretrievable. Regardless which option you choose, you are 
presented with the following options:

- Create a secret
- List secrets
- Update a secret
- Remove a secret
- Reveal a secret
- Write changes and go to main menu
- Go to the main menu without writing changes

When creating a secret, you can choose whether the secret should be text or binary data. When creating a text secret, 
you can also choose to make it an environment variable. These environment variables are exposed to the Elide runtime. 
When creating a binary secret, you are asked for an absolute path to a file containing the secret data.

### Changing encryption details

The `Change encryption details` option is for changing your personal encryption details. When selecting this option,
you are asked the same questions as in step 1 of [initialization](#initialization), asking for encryption mode. This is
useful if you want to change your passphrase, or if your GnuPG key has been revoked or expired.

## Using secrets

All guest languages have access to a new module called `secrets`, which has one function, `get()`, that takes in a 
single argument that must be a string. But to use these, secrets must be first initialized for runtime. To make this 
happen, please follow these steps:

1. If you have just one secret profile, you can skip this step. But with multiple profiles, one of them must be 
   selected. This can be done by adding the following lines to `elide.pkl`:
   ```
   secrets {
     profile = "profileName"
   } 
   ```
   The profile can also be selected with the environment variable `ELIDE_SECRETS_PROFILE`.
2. Secrets must be decrypted for use. If you use passphrase encryption, you may provide the passphrase with the 
   `ELIDE_SECRETS_PASSPHRASE` environment variable. If you use GnuPG encryption, the `gpg` binary may ask for
   authentication interactively while blocking Elide initialization, depending on how you have set it up on your system.

After following these steps, you may call secrets from guest languages. Currently, implementation exists for JavaScript 
and Python. For example:

- Javascript:
  ```javascript
  const secrets = require('secrets');
  let token = secrets.get('appToken');
  ```
- Python:
  ```python
  from elide import secrets
  token = secrets.get('appToken')
  ```

## Remote secret management

Remote secret management is meant for teams to collaborate with secrets, while only revealing the necessary secrets to
a developer. For this system, there are two levels of access.

- Superuser:
  
  A superuser is someone who has access to all secrets and delegates access to normal users. Superuser access is 
  protected with the same encryption modes as local secrets, passphrase and GnuPG.

- Regular user:
  
  A regular user is someone who has access to a subset of secrets, specifically one or more secret profiles. The 
  superuser allocates one or more access files that are each encrypted with either of the encryption modes.

### Remote secret providers

The first time you are doing any remote operation, you are asked where to access remote secrets. Currently, two remote 
secret providers are available, `project` and `github`.

- Project:

  Project secrets are stored within a project's sources. When selecting this option, you are asked for a path relative 
  to the root directory of your project.

- GitHub:

  Project secrets are stored in a separate, private GitHub repository. When selecting this option, you are asked for a
  GitHub repository identity and a personal access token.


> The remote secret provider, project remote path and GitHub repository can also be defined in `elide.pkl`:
> ```
> secrets {
>     remote = "project" /// or "github"
>     project {
>         path = ".path-to-secrets"
>     }
>     /// or
>     github {
>         repository = "organization/repository"
>     }
> }
> ```
> {style="note"}

## Remote secret management as a regular user

To use secrets as a regular user, you must select the `Pull an existing project` option when first running 
`elide secrets`. If you have already initialized secrets, you can reset by moving or deleting the `.elide-secrets`
directory from your project root.

After selecting a remote secret provider as described above, you are asked if you want to pull as a superuser. Selecting
`No` then asks you for an access file to import. Select an access file and provide the credentials if necessary, and you
are done. You can now use all the secret profiles listed in the access file. In the main menu, you can now use the 
option `Pull changes from remote` to update your local secrets, and `Push changes to remote` to update the remote 
secrets. With GitHub remotes, your access token must have write permissions to push changes.

## Remote secret management as a superuser

To manage remote secrets as a superuser, select the `Manage remote as a superuser` option in the main menu. You are 
asked for credentials, and if your local secrets and remote secrets are not identical, you are asked which changes to 
overwrite locally. You are then brought to the remote management menu.

### Remote management menu

The remote management menu has the following options:

- Create an access file
- List access files
- Select an access file
- Remove an access file
- Regenerate profile keys
- Delete a profile
- Restore a deleted profile
- Push changes to remote and go to the main menu
- Go to the main menu without pushing changes

Like with the main menu, most of these options are self-explanatory. 

> Deleting a profile deletes it from local and remote
> secrets, as well as all access files. You can only restore a deleted profile if you deleted it within the same remote
> management session. As soon as you return to the main menu, you can no longer restore deleted profiles.
> {style="warning"}

### Editing an access file

When selecting the `Select an access file` option, you are asked which access file to edit. You are then presented with 
the following options: 

- Add a profile to the access file
- List profiles in the access file
- Remove a profile from the access file
- Change encryption details
- Return to the remote management menu

### Regenerating profile keys

If a profile's secret key becomes compromised, it is vital to change that key so that any new secrets cannot be 
decrypted by third parties with access to the remote files. The `Regenerate profile keys` option asks you to select one
or more profiles to rekey, and then regenerates their keys. This option does not change any access file credentials.

## Non-interactive remote secrets

Sometimes you have to access secrets without any interactive access. For these purposes, Elide secrets provides the flag
`elide secrets -n` or `elide secrets --non-interactive`. Running this command has only one purpose: Pull a single 
profile from a remote and initialize it to be used with the runtime. To do this, it requires some extra options that are
normally optional.

In `elide.pkl`, you must specify the profile name, remote type and remote options:

```
secrets {
    profile = "profileName"
    remote = "project" /// or "github"
    project {
        path = ".path-to-secrets"
    }
    /// or
    github {
        repository = "organization/repository"
    }
}
```

In addition, you need to define some environment variables:

- `ELIDE_SECRETS_PASSPHRASE` to specify a passphrase for local encryption. This variable must also be provided to the
  runtime.
- `ELIDE_SECRETS_PROFILE` to override the profile specified in `elide.pkl`. If used, this variable must also be provided
  to the runtime.
- `ELIDE_SECRETS_ACCESS` to specify the name of an access file. This access file must contain the profile specified in
  `elide.pkl` or the above variable.
- `ELIDE_SECRETS_ACCESS_PASSPHRASE` to specify the passphrase of the access file. If the access file uses GnuPG, this 
  variable is not necessary, but GnuPG may need some additional configuration to function non-interactively.
- `ELIDE_SECRETS_GITHUB_TOKEN` to specify the GitHub access token. This variable is not necessary if `project` remote is
  used.

> The environment variables `ELIDE_SECRETS_PASSPHRASE`, `ELIDE_SECRETS_PROFILE` and `ELIDE_SECRETS_GITHUB_TOKEN` can 
> also be used with interactive secrets.
> {style="note"}