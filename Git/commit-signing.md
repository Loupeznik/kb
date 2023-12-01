# Commit signing

## Generating and exporting the PGP key

1. Generate the PGP key (per accepted PGP key types by your Git provider)

```powershell
gpg --full-generate-key

# choose default or select supported type (RSA/DSA is default)
# choose expiration (or without)
# fill account details (name, email)
```

2. Get ID of the key

```powershell
gpg --list-secret-keys --keyid-format=long
```

3. Export the public key

```powershell
gpg --armor --export *key_id*
```

4. [OPTIONAL] Export private key (if it should be used on another machine/for another user)

```powershell
gpg --export-secret-key -a *name_on_key* > *name_on_key*.gpg.key
```

## Enabling commit singing (Windows)

- If the key was generated on another system (WSL, other Linux install), a private key import might be needed. [GPG4Win](https://www.gpg4win.org/) can be a useful tool for managing keys
- GPG tools need to be installed (can be installed as part of GPG4Win or standalone)

```powershell
# Signing settings
git config commit.gpgsign true # current repo
git config --global commit.gpgsign true # globally

# GPG settings
git config --global user.signingkey <key_id>
git config --global gpg.program "C:\Program Files (x86)\GnuPG\bin\gpg.exe"

# sign the commit (-S flag)
git commit -S -m "initial"
```
