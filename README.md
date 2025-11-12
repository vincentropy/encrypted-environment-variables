# How to manage environment files securely during local development

## The problem

When working on local development environments, it's common to use environment files (e.g., `.env` files) to store configuration variables such as API keys and other sensitive information.
Assuming you already have your machine secured and your drive encrypted, this data is reasonable secure.

However, **Generative AI** coding assistants can access your local files and therefore access to your sensitive environment variables. Even if your coding assistant promises not to train models on your data there is often no guarantee that your data will be stored securely or deleted after use.

### Encrypting your environment files can protect them from several threats:

1. **Generative AI** coding assistants accessing local files
2. **Accidental exposure** through version control systems (e.g., pushing to a public GitHub repository)
3. **Unauthorized access** if your machine is compromised and your drive is not encrypted (or you used a weak password)

## Goals

This guide will help you achieve the following goals:

1. Encrypt your environment files using strong encryption methods.
2. Decrypt the environment files in-memory when needed, without writing decrypted data to disk.
3. Integrate the decryption process into your development workflow seamlessly.

## Tools we will use

- [SOPS](https://github.com/getsops/sops)
- [age](https://github.com/FiloSottile/age)
- Bash scripting

## How To use sops with age

### Basic concepts

`SOPS` is a tool for managing secrets that supports various encryption backends. `age` is a simple, modern, and secure encryption tool that `SOPS` can use as a backend.

We will use `age` to generate a key pair and configure `SOPS` to use the public key for encrypting our environment files. The private key will be used for decryption.

We will create simple bash scripts to handle the decryption and loading of environment variables into the shell if the shell is started in a folder containing an encrypted environment file.

## Prerequisites

- install `sops`
- install `age`

(you can use your system's package manager on linux or [brew](https://brew.sh/) on MacOS)

## Setting up sops with age

### Generate an age key pair

```bash
age-keygen -o  ~/.config/sops/age/keys.txt
chmod 600 ~/.config/sops/age/keys.txt
```

On MacOS the config file location is elsewhere by default.
Below, we'll use the environment variable `XDG_CONFIG_HOME` to point to the right location.

### Configure sops to use age

Create a file at `~/.config/sops/config.yaml` with the following content:

```yaml
creation_rules:
  - age: <public-key>
```

The public key can be found in the same file you just created, `~/.config/sops/age/keys.txt`.
Note: no quotes around the public key.

### Working with existing environment files

With the simple setup above, you can now encrypt and decrypt environment files from the command line. We'll streamline this process with simple bash scripts below. But it's useful to know the commands.

#### Encrypt an environment file

```bash
XDG_CONFIG_HOME=$HOME/.config sops --config ~/.config/sops/config.yaml encrypt .env > .env.enc
```

#### Decrypt an environment file

```bash
XDG_CONFIG_HOME=$HOME/.config sops decrypt --filename-override .env  .env.enc > .env.dec
```

## Automating decryption and loading environment variables

### A bash script to decrypt an environment file and read it into the environment

We'll first create a script called `sopssetenv.sh` that decrypts a `.env.enc` file, if present in the current directory, and outputs export statements for each variable. This script can be used with `eval` or in a sub-shell to set the environment variables for the current shell session.

(This file is also available in this repository as `scripts/sopssetenv`)

```bash
#!/bin/bash
# Decrypt .env.enc using sops and output export statements for each variable

set -e

if [ ! -f .env.enc ]; then
  echo ".env.enc not found in current directory."
  exit 1
fi

# Decrypt and output export statements for each variable
XDG_CONFIG_HOME=$HOME/.config sops decrypt --filename-override .env .env.enc | while IFS= read -r line; do
  # Skip empty lines and comments
  if [[ -z "$line" || "$line" =~ ^# ]]; then
    continue
  fi
  echo "export $line"
done
```

If this script is saved as `sopssetenv.sh` and made executable with `chmod +x sopssetenv.sh`, you can run it in a sub-shell to set the environment variables for the current shell session:

```bash
eval "$(./sopssetenv.sh)" && echo $MY_SECRET_VARIABLE
```

### Adding to your path

to make a command called `sopssetenv` available globally, you can create this file in a directory that is included in your system's PATH.

```bash
cp sopssetenv.sh <some-directory-in-your-PATH>/sopssetenv
```

You can then run the command from any directory:

```bash
eval "$(sopssetenv)" && echo $MY_SECRET_VARIABLE
```

### Editing the encrypted file

Similartly to setting environment variables, you we can edit the encrypted file using sops.

create a file called `sopseditenv` with the following content, make it executable and add it to your path.

```bash
#!/bin/bash
# Script to edit the encrypted .env.enc file using sops
XDG_CONFIG_HOME=$HOME/.config sops --config ~/.config/sops/config.yaml edit --input-type dotenv --output-type dotenv .env.enc
```

### Automatic loading of environment variables on shell startup

When using an IDE like VSCode, it's common to run a development server from a VSCode shortcut. To ensure that the environment variables from the encrypted `.env.enc` file are loaded automatically when VSCode starts a new shell session, you can add the following snippet to your shell's configuration file (e.g., `~/.zprofile`, `~/.profile`, or `~/.bash_profile`, depending on your shell):

```bash
[ -f .env.enc ] && eval "$(sopssetenv)" || echo 'No encoded env file found, skipping...'
```

This snippet checks for the presence of a `.env.enc` file in the current directory. If it exists, it runs the `setenvsops` script to load the environment variables into the shell session. If the file is not found, it simply outputs a message indicating that no encoded environment file was found.

## Migrating existing environment files

You're now ready to migrate existing environment files to the encrypted format.
In a folder containing a `.env` file, run the following command to create an encrypted version:

```bash
XDG_CONFIG_HOME=$HOME/.config sops --config ~/.config/sops/config.yaml encrypt .env > .env.enc
```

you can test that the encrypted file works by running

```bash``
eval "$(sopssetenv)" && echo $MY_SECRET_VARIABLE
```

where `MY_SECRET_VARIABLE` is one of the variables defined in your original `.env` file.
It should now be ok to delete the original `.env` file. Make sure all your important variables are backed up somewhere safe before deleting the unencrypted file!
If you lose your encryption keys you will not be able to recover the data in the encrypted file.
