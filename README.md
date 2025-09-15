# Securegit

## Intro
securegit is an open-source tool that provides end-to-end security properties for Git users against other attackers, including Git servers. The underlying scheme of this tool is shown in "End-to-End encrypted Git services" https://eprint.iacr.org/2025/1208.pdf. The artifact is licensed under Apache 2.0.

## Install instrucions
Before installing, make sure you have **Python 3.8 or higher**, pip, and Git installed on your system.  
It is recommended to use a virtual environment to isolate the package and its dependencies.
Download `securegit-0.3.0-py3-none-any.whl`.

### 1. Create and activate a virtual environment

#### On macOS / Linux:
```bash
python3 -m venv venv
source venv/bin/activate
```
#### On Windows:
```bash
python -m venv venv
venv\Scripts\activate
```
### 2. Upgrade pip (optional but recommended)
```bash
python -m pip install --upgrade pip
```
### 3. Install securegit
```bash
python3 -m pip install securegit-0.3.0-py3-none-any.whl
```

### 4. Set the local Git username
Please make sure that your local Git username matches your GitHub account username. You can check and update it as follows:
```bash
# Check your current Git username:
$ git config --global user.name
# Set your Git username to match your GitHub account username:
$ git config --global user.name "YourGitHubUsername"
```

## Commands

### 1. securegit init - Initialize repositories

#### SYNOPSIS
 ```bash
securegit init <name> [--key-path <path>]
```

#### DESCRIPTION
 
This command creates new SecureGit repositories consisting of two linked repositories:

- A plaintext repository named `\<name\>`
- An encrypted repository named `\<name\>_cipher`

It also generates a random 32-byte symmetric key file used for encryption and decryption.
 
#### ARGUMENTS

**\<name\>**
The base name of the repositories. The plaintext repository will be created at `./\<name\>`, and the encrypted repository will be created at `./\<name\>_cipher`.

#### OPTIONS
**--key-path \<path\>**
Path to save the generated key file. Defaults to `./symkey.bin`.

#### EXAMPLES
 ```bash
# Initialize a plaintext repository named `project` and an encrypted repository named `project_cipher` with the default key file:
$ securegit init project
# Initialize repositories with a custom key file path:
$ securegit init project --key-path /keys/docs.key
```


### 2. securegit adduser - Share an encrypted SecureGit repository with a user

#### SYNOPSIS
 ```bash
securegit adduser <enc_repo_path> --owner-key <owner_key> [--share-name <share_name>]
```
#### DESCRIPTION
The command adds a new collaborator (sharee) to an encrypted SecureGit repository.
It performs the following actions:
1) Copy the sharee’s encryption and signing public keys into the encrypted repository’s `shareinfo` directory.
2) Encrypt the symmetric key for the sharee with AES-CTR and write it to `shareinfo/<share_name>_keycipher.bin`.
3) Sign the `shareinfo` folder using the owner's private key.
4) Send a GitHub repository invitation if a GitHub token and origin remote are available.

#### ARGUMENTS

**<enc_repo_path>**
Path to the encrypted repository. Must exist.

**Before running this command, make sure that you have created a repository on the GitHub server, and the local encrypted repository has added the remote url. **

#### OPTIONS

**--owner-key <owner_key>**
Path to the owner’s signing private key (DER format). **Required.**

**--share-name <share_name>**
Username of the new collaborator. **Optional.**
No need when add the owner's public keys.

**--enc-pub <enc_pub>**
Path to the sharee’s encryption public key (DER). **Required.**

**--sig-pub <sig_pub>**
Path to the sharee’s signing public key (DER). **Required.**

**--sym-key <sym_key>**
Path to the symmetric key file (binary) to encrypt for the sharee. **Required.**

**--token <token>**
GitHub personal access token for sending a collaborator invite. **Optional.**
No need when add the owner's public keys.

#### EXAMPLES

 ```bash
# Share with the user alice and automatically send an invitation:
$ securegit adduser myrepo_cipher --owner-key owner_sign.der --share-name alice --enc-pub alice_enc.der
           --sig-pub alice_sign.der --sym-key sym.key --token ghp_xxx
```
The username **must** be the same as the one registered on GitHub.

#### NOTE
**A repository owner also needs to run this command to add their own public keys `owner_enc_pub.der` and `owner_sign_pub.der` to the encrypted repository.** For example:
 ```bash
securegit adduser <enc_repo_path> --owner-key <owner_key>
    --enc-pub owner_enc_pub.der --sig-pub owner_sign_pub.der --sym-key <sym_key>
```

The minimal requirement of tokens used in this command:
- Personal access tokens (classic): choose `repo`
- Fine-grained personal access token: choose `Administration: Read and Write` for the seleted reopsitory.


### 3. securegit add - Secure version of `git add` that updates both plaintext and encrypted repositories 

#### SYNOPSIS
 ```bash
securegit add [--char | --line] <plaintext_repo_path> <encrypted_repo_path> <key_path>
```

#### DESCRIPTION
The command works like `git add`, but with encryption support.
It stages changes in the plaintext repository and simultaneously updates the encrypted repository, encrypting modified files according to the configured mode.

On first use, you must specify an encryption mode (`--char` or `--line`). This mode will be stored in a configuration file inside the encrypted repository (`securegit_config.json`). Subsequent invocations will reuse this mode automatically.

#### ARGUMENTS
**<plaintext_repo_path>**
Path to the plaintext repository. Changes are staged here with `git add -A`.
**<encrypted_repo_path>**
Path to the corresponding encrypted repository.
**<key_path>**
Path to the AES key file used for encryption.

#### OPTIONS
**--char**
Update the ciphertext by encrypting the character-wise diff
**--line**
Update the ciphertext by encrypting the line-wise diff

#### EXAMPLES

 ```bash
# Initialize in the character-level mode:
$ securegit add --char ./plain ./encrypted ./keys/aes.key
# Initialize in the line-level mode:
$ securegit add --line ./plain ./encrypted ./keys/aes.key
# Subsequent usage (mode auto-loaded from config):
$ securegit add ./plain ./encrypted ./keys/aes.key
```

### 4. securegit commit - Secure version of `git commit` that signs encrypted repository commits

#### SYNOPSIS
 ```bash
securegit commit <plaintext_repo_path> <encrypted_repo_path> <msg> <private_key_path>
```

#### DESCRIPTION
The command commits changes to both the plaintext and the encrypted repositories.

#### ARGUMENTS
**<plaintext_repo_path>**
Path to the plaintext repository to be committed.
**<encrypted_repo_path>**
Path to the encrypted repository to be committed.
**\<msg>**
Commit message to use for the plaintext repository. It must be placed in double quotes.
**<private_key_path>**
Path to the private key (DER format) used for generating the commit signature.

#### EXAMPLES

 ```bash
# Commit changes with signing key:
$ securegit commit ./plain ./encrypted "Update README" ./keys/signing_key.der
```


### 5. securegit branch - Same as `git branch`, but in both plaintext and encrypted repositories at the same time.

#### SYNOPSIS
 ```bash
securegit branch <plaintext_repo_path> <encrypted_repo_path> [git branch args...]
```

#### DESCRIPTION
This command mirrors the functionality of `git branch`, but applies it to both the plaintext repository and the encrypted repository.
Any arguments passed after the repository paths (such as `-M`, `-d`, `-a`, or branch names) are forwarded directly to `git branch`.

#### ARGUMENTS
**<plaintext_repo_path>**
Path to the plaintext repository.
**<encrypted_repo_path>**
Path to the encrypted repository.
**[git branch args...]**
Additional arguments passed to `git branch`. For example:
- `-M main` (rename branch to main)
- `-d old-branch` (delete branch)
- `-a` (list all branches, including remote)

#### EXAMPLES
 ```bash
# List branches in both repositories `plain` and `encrypted`:
$ securegit branch plain encrypted
# Rename current branch to `main`:
$ securegit branch plain encrypted -M main
```

### 6. securegit remote - Same as `git remote`, but only in encrypted repositories

#### SYNOPSIS
 ```bash
securegit remote <encrypted_repo_path> [git remote args...]
```

#### DESCRIPTION
The command wraps the standard git remote command but applies only to the encrypted repository.
It is typically used to configure or inspect remote connections for the encrypted repository. Any additional arguments after `<encrypted_repo_path>` are passed directly to `git remote`.

#### ARGUMENTS
**<encrypted_repo_path>**
Path to the encrypted repository.
**[git remote args...]**
Arguments passed to `git remote`. For example:
- `-v` — Show remotes with URLs.
- `add <name> <url>` — Add a new remote.
- `remove <name>` — Remove a remote.
- `rename <old> <new>` — Rename a remote.
- `set-url <name> <url>` — Change a remote URL.

#### EXAMPLES
 ```bash
# List remotes for the encrypted repository `encrypted`:
$ securegit remote encrypted -v
# Add a new remote:
$ securegit remote encrypted add origin https://github.com/user/myrepo.git
```


### 7. securegit push - Same as `git push`, but only in encrypted repositories

#### SYNOPSIS
 ```bash
securegit push <encrypted_repo_path> [git push args...]
```

#### DESCRIPTION
This command runs `git push` inside the encrypted repository and streams the output directly to the terminal.

#### ARGUMENTS
**<encrypted_repo_path>**
Path to the encrypted repository.
**[git push args...]**
Additional arguments passed to `git push`. For example:
- `<remote>` — Name of the remote (e.g., origin).
- `<branch>` — Name of the branch to push.
- `--force` — Force push.

#### EXAMPLES
 ```bash
# Push the current branch to `origin`:
$ securegit push encrypted origin main
```

### 8. securegit clone - Clone an encrypted repository and restore its plaintext history

#### SYNOPSIS
 ```bash
securegit clone <remote_url> <encrypted_repo_path> <plaintext_repo_path> <owner_name> <sharee_name> <sharee_privkey>
               [--sym_key_out <file>] [-b <branch>]
```

#### DESCRIPTION
This command fetches an encrypted repository from a remote and reconstructs the corresponding plaintext repository locally.
It replays all commits in chronological order, decrypting and applying changes to the plaintext working tree while preserving author and committer information.
 
#### ARGUMENTS
**<remote_url>**
Remote URL of the encrypted repository.
**<encrypted_repo_path>**
Local path where the encrypted repository will be cloned.
**<plaintext_repo_path>**
Local path where the decrypted plaintext repository will be constructed.
**<owner_name>**
Repository owner’s username.
**<sharee_name>**
Current collaborator’s username.
**<sharee_privkey>**
Path to the collaborator’s private key (used to decrypt the symmetric key).

#### OPTIONS

**--sym_key_out \<file>**
Save the decrypted symmetric key to <file> (default: `./symkey.bin`)
**-b, --branch \<branch>**
Branch to clone and replay (default: `main`)


#### EXAMPLES

 ```bash
# Clone encrypted repo and restore plaintext history
securegit clone https://github.com/user/encrypted_repo.git repo_cipher repo_plain alice bob keys/bob_priv.pem

# Save decrypted symmetric key to a custom path
securegit clone https://github.com/user/encrypted_repo.git repo_cipher repo_plain alice bob keys/bob_priv.pem --sym_key_out out/key.bin
```

### 9. securegit ignore - Manage `.gitignore` in both plaintext and encrypted repositories

#### SYNOPSIS
 ```bash
securegit ignore <plaintext_repo_path> <encrypted_repo_path> <patterns...>
```

#### DESCRIPTION
This command ensures that the same `.gitignore` file is created or updated in both the plaintext and encrypted repositories.

 
#### ARGUMENTS
**<plaintext_repo_path>**
Path to the local plaintext repository

**<encrypted_repo_path>**
Path to the corresponding encrypted repository

**<patterns...>**
One or more ignore patterns (files, directories) to be added to `.gitignore`

#### EXAMPLES

 ```bash
# Ignore build/ directory and .DS_Store files
securegit ignore repo_plain repo_cipher build/ .DS_Store
```

### 10. securegit keygen - Generate a key pair

#### SYNOPSIS
 ```bash
securegit keygen [--privkey-out <path>] [--pubkey-out <path>]
```

#### DESCRIPTION
The command generates a new key pair that is exported in DER format. Both output paths can be customized.


#### OPTIONS
**--privkey-out \<path\>**
Path to save the private key (default: ./private_key.der)

**--pubkey-out \<path\>**
Path to save the public key (default: ./public_key.der)

#### EXAMPLES

 ```bash
# Generate keys with default filenames:
$ securegit keygen

# Generate keys into a custom directory:
$ securegit keygen --privkey-out ~/.keys/alice_priv.der --pubkey-out ~/.keys/alice_pub.der
```

### 11. securegit newrepo - Create a new GitHub repository under your account

#### SYNOPSIS
 ```bash
securegit newrepo <repository-name> [--private] --token <token>
```

#### DESCRIPTION
This command can create a new repository on the GitHub server under the authenticated user’s account. By default, the repository will be public, unless the `--private` option is given.
 
#### ARGUMENTS
**\<repository-name\>**
The name of the repository to create. Must be unique under your account.

**--token \<token>**
GitHub Personal Access Token used for authentication.

#### OPTIONS
**--private**
Create the a private repository. By default, repositories are public.

#### EXAMPLES

 ```bash
# Create a public repository named myrepo:
$ securegit newrepo myrepo
# Create a private repository with a token:
$ securegit newrepo myrepo --private --token ghp_xxx123
```

The minimal requirement of tokens used in this command:
- Personal access tokens (classic): choose `repo` for private repositories and only choose `repo: public` for public repositories
- Fine-grained personal access token: choose `Administration: Read and Write` for the All reopsitory.

### 12. securegit pull - Secure version of `git pull`

#### SYNOPSIS
 ```bash
securegit pull <plaintext-repo-path> <encrypted-repo-path> <owner-name> <user-name>
              [--sym-key <file> | --private-key <file>] [--sym-key-out <file>]
              [<remote> [<branch>]]
```

#### DESCRIPTION
This command can updates the encrypted repository by fetching from the given remote and branch and then integrates changes into the local plaintext repository.
 

#### ARGUMENTS
**\<plaintext-repo-path>**
Local path of the plaintext repository.
**\<encrypted-repo-path>**
Local path of the encrypted repository.

#### OPTIONS
**--sym-key \<file>**
Read the symmetric AES key from the specified file.

**--private-key \<file>**
Use the specified private key to decrypt the AES key share stored in the encrypted repository.

**--sym-key-out \<file>**
Write the decrypted AES key to the specified path. If not given, defaults to `./symkey.bin`.

**\<remote>**
The name of the remote to fetch from. Defaults to `origin`.

**\<branch>**
The branch to fetch and replay commits from. Defaults to `main`.

#### EXAMPLES

 ```bash
# Pull using an existing symmetric key:
$ securegit pull plain encrypted owner alice --sym-key ./symkey.bin
# Pull and derive the AES key with private key:
$ securegit pull plain encrypted owner alice --private-key alice_priv.der
```

## Note:
The user must run `securegit commit` after running `securegit add`.

The user can run `securegit adduser` multiple times before running `securegit commit`.

For one commit, both `securegit add` and `securegit adduser` commands can be run.

## Usage process:
**Navigate to your desired directory where you want to store your repositories.**

In order to be able to execute all of the above commands, we recommend generating a classic token with `repo` permission or a fine-grained token with `Administration: Read and Write` for all repository.


Before initializing new repositories, you need to creat two public/private keypairs, one is used for encryption, and the other one is used for signing. If you do not have keypairs, please run the following command to generate,
```bash
securegit keygen enc_priv.der enc_pub.der
securegit keygen sign_priv.der sign_pub.der
```
You can change the output path and name, reffering the detailed introduction to this command before.

#### 1. Initialize a plaintext repository and a ciphertext repository.
```bash
securegit init myrepo 
```
It will initialize a plaintext repository `myrepo` and a ciphertext repository `myrepo_cipher`. It will also output a file named `symkey.bin` at the current dictionary to store the encryption key for `myrepo`.

#### 2. Create `.gitignore` file
On macOS, it is **necessary** to create a `.gitignore` file to prevent `.DS_Store` from being tracked by Git.  
```bash
securegit ignore myrepo myrepo_cipher .DS_Store
```

#### 3. Create a new repository on the Github server.
```bash
securegit newrepo myrepo_cipher --private --token ghp_XXX
```

Add the remote url with \<token> for the encrypted repository:
```bash
securegit branch myrepo myrepo_cipher -M main
securegit remote myrepo_cipher add origin https://<token>@github.com/XXX/XXX.git
```
It also supports SSH url, please make sure that your local environment is properly configured with an SSH key linked to your GitHub account.

#### 4. Add collaborator (inlcuding owner)
Run the following command to add owner's public key
 ```bash
securegit adduser <enc_repo_path> --owner-key <owner_key>
    --enc-pub owner_enc_pub.der --sig-pub owner_sign_pub.der --sym-key <sym_key>
```
For example, use the following command to add owner's public keys `owner_enc_pub.der` and `owner_sign_pub.der` and sign them with their private key `owner_sign_priv.der` and encrypt the symmetric key `symkey.bin` with `owner_enc_pub.der`.
```bash
securegit adduser myrepo_cipher --owner-key owner_sign_priv.der --enc-pub owner_enc_pub.der --sig-pub owner_sign_pub.der --sym-key symkey.bin
```

Run the following command to add collaborator's public key
 ```bash
securegit adduser <enc_repo_path> --owner-key <owner_key> --share-name <share_name>
    --enc-pub <enc_pub> --sig-pub <sig_pub> --sym-key <sym_key> --token <token>
```
For example, invite a sharee named `alice`:
```bash
securegit adduser myrepo_cipher --owner-key owner_sign_priv.der --share-name alice --enc-pub alice_enc_pub.der --sig-pub alice_sign_pub.der --sym-key symkey.bin --token ghp_XXX
```

#### 5. Add the modifications
After making changes in your plaintext repository, use `securegit add` to stage the modifications for encryption:
```bash
securegit add --char/line myrepo myrepo_cipher symkey.bin
```
Only need to specify the mode for the first time.

#### 6. Commit the modifications
After running `securegit add`, then commit the modifications:
```bash
securegit commit myrepo myrepo_cipher "msg" owner_sign_priv.der
```

#### 7. Upload the ciphertext repository to the GitHub server
```bash
securegit push myrepo_cipher -u origin main
```

#### 8. Fetch the latest commits
Assume alice fetches the latest commits, then run
```bash
securegit pull myrepo myrepo_cipher <owner_name> alice --private-key alice_enc_priv.der 
```

If alice has the symetric encryption key, then run
```bash
securegit pull myrepo myrepo_cipher <owner_name> alice --sym-key-out symkey.bin
```

#### 9. Clone the whole repository
Assume alice wants to clone the repository:
```bash
securegit clone https://github.com/owner_name/myrepo_cipher.git myrepo_cipher myrepo <owner_name> alice alice_enc_priv.der 
```


To Do:
support `git merge`;
support default path;
support git credential manager to get token;
support different permission for share
support bitbuct (adduser, newrepo)
