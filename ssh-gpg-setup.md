# Setting up SSH/GPG for multiple accounts

## Use Case

Different repositories on git can require different accounts in order to access them. This could be a default/personal salesforce account, a git soma account, and an SFEMU account for example. The key idea allowing this to work is using aliases for [github.com](http://github.com/) in your ssh config and setting up of the remote repository in your git config files.

## **Step 1: Creating SSH Keys for all accounts**

```
cd ~/.ssh
ssh-keygen -t rsa -C "account-1-email-address" -f "account-1-name"
ssh-keygen -t rsa -C "account-2-email-address" -f "account-2-name"
```

-f is for specifying the file name so the passed in string doesn’t have to be something specific, just use it to identify which ssh key is for which account

“rsa” specifies the type of encryption they key uses. The best practice at Salesforce may change over time as there are multiple encryption methods that can be used but rsa is what was suggested in futureforce onboarding trainings (ex. you may need to replace rsa with ed25519 in the command)

After you run the command, you have the option of setting an optional passphrase for using the key. 


## **Step 2: Add SSH keys to the SSH agent**

```
ssh-add --apple-use-keychain ~/.ssh/[account 1 ssh file name no .pub extension]
ssh-add --apple-use-keychain ~/.ssh/[account 2 ssh file name no .pub extension]
```

## **Step 3: Generate GPG keys**

Some repositories require signed commits, to sign your commits you need to have a gpg key setup. If you skip this step you will likely get an error along the lines of ‘gpg failed to sign the data’ when trying to interact with this repository. You can also complete step 4 to finish SSH setup and return to this one to setup commit signing separately.
For reference: [Generating a new GPG key - GitHub Docs](https://docs.github.com/en/authentication/managing-commit-signature-verification/generating-a-new-gpg-key)

1. Download and install [GPG command line tools](https://www.gnupg.org/download/) if you haven’t already
2. Generate a GPG key pair by running `gpg —full-generate-key`  from the terminal
    1. Use the default RSA key option and enter a name and email address when prompted. 
    2. This will be used to create a User-ID for the key ex. “`Account 1 Name <account-1-email@salesforce.com>"` so make sure they name provides some info that can allow you to differentiate between accounts so you know which gpg key is for which account.
3. Repeat step 2 for your other accounts



## **Step 4: Add public SSH keys to Github**

If you navigate to your `~/.ssh` folder and type `ls` , you should see files with the format:

```
account-1-name.pub
account-1-name
account-2-name.pub
account-2-name
```

1. Copy all the contents of `account-1-name.pub` to your clipboard 
    1. pbcopy < ~/.ssh/account-1-name.pub
2. Navigate to https://github.com/settings/keys
3. Click the **New SSH Key** button on the top right
4. Add a title of your choice, key type is Authentication Key, and paste the contents of account-1-name.pub in the ‘key’ text field
5. Finish by clicking **Add SSH key**
6. Repeat the same process for your second account



## **Step 5: Add GPG Keys to Github** 

1. To access your generated GPG keys, run 

`gpg --list-secret-keys --keyid-format=long` from the terminal

You should get something like this:

```
$ gpg --list-secret-keys --keyid-format=long
/Users/hubot/.gnupg/pubring.kbx
------------------------------------
sec   ed25519/3AA5C34371567BD2 2016-03-10 [expires: 2024-03-10]
uid                          Account 1 <account-1@example.com>
ssb   cv25519/4BB6D45482678BE3 2016-03-10
sec   ed25519/5AA5C34371567BD2 2016-03-10 [expires: 2024-03-10]
uid                          Account 2 <account-2@example.com>
ssb   cv25519/6BB6D45482678BE3 2016-03-10
```

The keys are displayed in the format `encryption_type/key`

1. Copy the key following `encryption type/` (ex. 3AA5C34371567BD2 for account-1) and run the command:
    1. `gpg —armor —export 3AA5C34371567BD2 // replace with your key`
2. Copy your GPG key, beginning with `-----BEGIN PGP PUBLIC KEY BLOCK-----` and ending with `-----END PGP PUBLIC KEY BLOCK-----`
3. Navigate to https://github.com/settings/keys (same place used for adding SSH keys)
4. Click **New GPG key**
5. Set the title to something that indicates to you what machine this key is being used from and paste your copied GPG key into the “Key” field.
6. Click **Add GPG key** when finished
7. Repeat for any other accounts.

## **Step 6: Set up config files** 

There are 3 types of config files that need to be setup correctly for commit signing to work correctly:

1. ~/.ssh/config
2. ~/.gitconfig (global git config)
3. repository_name/.git/config (local git config)

Note: The local git config overrides information in the global config so you can setup a default account’s details in the global config and override them with a local config in a repository to change the account used.
Here’s what each one should look like
**~/.ssh/config**

```
### Account 1
Host account-1 github.com
  HostName github.com
  PreferredAuthentications publickey
  AddKeysToAgent yes
  IdentitiesOnly yes
  User account-1-username
  IdentityFile ~/.ssh/account-1

### Account 2
Host account-2 github.com
  HostName github.com
  PreferredAuthentications publickey
  User account-2-username
  IdentityFile ~/.ssh/account-2
```

If this file does not exist, create it `touch ~/.ssh/config`

This sets up an alias for the host name [github.com](http://github.com/) that will change the way you clone repositories (the default git clone via ssh command will need to be edited) and the way your repository .git/config file but also allows you to specify which ssh key is being used for each repository cleanly. The alias does not need to be the same as the account name but in this case I used the same name for both for simplicity.

**~/.gitconfig**

```
[user]
  name = Account 1 Name
  email = account-1@salesforce.com
  signingkey = 3AA5C34371567BD2 // set's this as default signing key
[alias]
        ci = commit
        co = checkout
        st = status
        br = branch
        lg1 = log --graph --abbrev-commit --decorate --format=format:'%C(bold blue)%h%C(reset) - %C(bold green)(%ar)%C(reset) %C(white)%s%C(reset) %C(dim white)- %an%C(reset)%C(bold yellow)%d%C(reset)' --all
        lg2 = log --graph --abbrev-commit --decorate --format=format:'%C(bold blue)%h%C(reset) - %C(bold cyan)%aD%C(reset) %C(bold green)(%ar)%C(reset)%C(bold yellow)%d%C(reset)%n''          %C(white)%s%C(reset) %C(dim white)- %an%C(reset)' --all
        lg = !"git lg1"
[commit]
  gpgsign = true // set to false to turn off by default
[push]
  gpgsign = if-asked
[gpg]
  program = /opt/homebrew/bin/gpg
```

This sets up account-1 as the default option for committing and signing from. 

This same format of information can also be stored/modified in the repository git config to override the information in the global config. For example, if you want to keep gpg signing on by default but turn it off in a certain repository. simple add this to your local repositories `.git/config` file:

```
[commit]
  gpgsign = false
```

**~/repository_name/.git/config**

```
[core]
    repositoryformatversion = 0
    filemode = true
    bare = false
    logallrefupdates = true
    ignorecase = true
    precomposeunicode = true
[remote "origin"]
    url = git@account-#:author-name/repo-name.git
    fetch = +refs/heads/*:refs/remotes/origin/*
[branch "main"]
    remote = origin
    merge = refs/heads/main
[user]
    signingkey = YOUR_SIGNING_KEY // overrides the ~/.gitconfig chosen key
```

You can also setup the remote origin for a pre-existing repository, using this command to set the which account you want to use for ssh/gpg:

```
git remote add origin git@**account****-#**:author_name/repo_name.git
```

**account-#** will be the alias setup for [github.com](http://github.com/) in your ~/ssh/config file

If you are cloning the repository use:

```
git clone git@**account-#**:author_name/repo_name.git
```

[Note: You must clone the repository using SSH not HTTPS for this to work correctly]

These commands add the following to the **~/repository_name/.git/config** file:

```
[remote "origin"]
    url = git@account-1:author_name/repo_name.git
    fetch = +refs/heads/*:refs/remotes/origin/*
```

Adding `[git@](mailto:git@github.com)alias` instead of the default just `[git@github.com](mailto:git@github.com)` is the key to making the different accounts be used seamlessly across different repositories.

# **Common Fixes**

Sometimes, you may still get an error when trying to sign your commit. Here are two common fixes:

1. Run `export GPG_TTY=$(tty)`  from the terminal
2. Download [GPG Suite](https://gpgtools.org/). Run the program and your keys should show up in the list. Right click your sec/pub for your account and click “Sign.”

Authorize the key if you’re getting a single sign on error

ssh -Tv is also good for debugging

## Improvements to this Doc

* I think decoupling account-1 being used everywhere would be good. The alias doesn’t need to be the same as the account name which can be confusing. 
* Showing examples of all of the configuration files high level right at the beginning would make the doc easy to use for someone who is an experience developer. The configs themselves are pretty self explanatory
* Doc is a bit long and has a lots of words

