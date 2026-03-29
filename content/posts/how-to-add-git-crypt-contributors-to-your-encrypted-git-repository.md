+++
date = '2025-01-11T15:07:40-04:00'
title = 'How to Add git-crypt Contributors to Your Encrypted Git Repository'
+++

Managing sensitive information in a Git repository can be challenging, but tools like `git-crypt` make it easier by encrypting specific files. When adding a new contributor to such a repository, the admin needs to ensure they have the necessary access to decrypt and work with these sensitive values. This tutorial aims to provide a detailed, step-by-step guide to help admins manage contributors effectively, as the official [git-crypt](https://github.com/AGWA/git-crypt) repository provides only basic setup instructions.

## I. Prerequisites

Before you begin, ensure the following:

*   `git-crypt` has been initialized in the Git repository and it's currently unlocked on the administrator's local machine
    
*   [homebrew](https://brew.sh/) installed on the contributor's local machine
    

## II. Step-by-Step Guide

### A. Generate a GPG key

On the contributor’s local machine:

*   1 - Install [GnuPG](https://gnupg.org/) :
    

```sh
brew install gnupg
```

*   2 - Generate a GPG key pair that uses a RSA algorithm for both encrypting and signing. Make sure to set an expiration date and a passphrase to enhance security :
    

````sh
gpg --full-generate-key
Please select what kind of key you want:
   (1) RSA and RSA
   (2) DSA and Elgamal
   (3) DSA (sign only)
   (4) RSA (sign only)
   (9) ECC (sign and encrypt) *default*
  (10) ECC (sign only)
  (14) Existing key from card
Your selection? 1
RSA keys may be between 1024 and 4096 bits long.
What keysize do you want? (3072) 4096
Requested keysize is 4096 bits
Please specify how long the key should be valid.
         0 = key does not expire
      <n>  = key expires in n days
      <n>w = key expires in n weeks
      <n>m = key expires in n months
      <n>y = key expires in n years
Key is valid for? (0) 365
Key does not expire at all
Is this correct? (y/N) y
GnuPG needs to construct a user ID to identify your key.
Real name: <contributor full name>
Email address: <contributor email>
Comment:
You selected this USER-ID:
    "<contributor full name> <contributor email>"
Change (N)ame, (C)omment, (E)mail or (O)kay/(Q)uit? O
    ```

* 3 - Export the public key :
    
```sh
gpg --armor --export <contributor email> > publickey.asc
````

*   4 - Output the GPG key ID :
    

```sh
gpg --list-keys | awk '/^pub/{getline; print $1}'
```

*   5 - Share the GPG public key and the key ID to the administrator
    

### B. Add the contributor to git-crypt

On the administrator ’s local machine:

*   1 - Import the contributor's public key in the GPG keyring :
    

```sh
gpg --import publickey.asc 
gpg: key XXX: public key "<contributor full name> <contributor email>" imported
gpg: Total number processed: 1
gpg:               imported: 1
```

*   2 - Add the contributor
    

```sh
cd <Git repository path>
git-crypt add-gpg-user --trusted <contributor GPG key ID>
```

### C. Verify access

On the contributor’s local machine:

*   1 - Install `git-crypt`:
    

```sh
brew install git-crypt
```

*   2 - Clone the Repository :
    

```sh
git clone https://github.com/your-repo.git
cd <Git repository>
```

*   3 - Decrypt the repo :
    

```sh
git-crypt unlock
```

*   4 - The contributor has now access to the encrypted files
    

## III. Conclusion

By following these steps, an admin can securely add a contributor to a Git repository that uses `git-crypt` for sensitive values. This ensures that sensitive information remains protected while allowing the new contributor to work effectively within the repository.

## IV. Future Exploration

In the near future, I will explore [SOPS](https://github.com/getsops/sops) as a potential replacement for `git-crypt` to see if it better meets the same use case.
