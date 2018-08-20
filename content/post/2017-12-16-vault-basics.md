---
title: 'Vault basics'
thumbnail: /images/server.vault.up.png
date: '2017-12-16'
categories:
- devops
tags:
- vault
- security
comments: []
---

[Vault](https://vaultproject.io) from HashiCorp is an amazing tool to manage the secrets on your organisation. It not only can help you to manage what they call static secrets that you can write and read, but also allows you to manage dynamic secrets to, for example, create temporary users in a MySQL database with certain permissions. It helps you to have a more secure organization.

Today we're going to see how can we configure and use the basics of Vault.

## Environment

We're going to define a non-production ready environment for our tests. If you want to learn what you should do to productionise this environment, please follow their [hardening](https://www.vaultproject.io/guides/production.html) guide.

Let's start then. Assuming that you already have docker installed, let's create a docker compose file to create the vault server. Create a folder in your laptop and create a file called *docker-compose.yml* with the following content:

```
version: '3'
 
services:
  vault:
    image: vault
    container_name: vault.server
    ports:
      - 8200:8200
    volumes:
      - ./etc/vault.server/config:/mnt/vault/config
      - ./etc/vault.server/data:/mnt/vault/data
      - ./etc/vault.server/logs:/mnt/vault/logs
    restart: always
    cap_add:
      - IPC_LOCK
    environment:
      VAULT_ADDR: http://127.0.0.1:8200
    entrypoint: vault server -config="/mnt/vault/config/config.hcl"
```

This will bring up a new container named vault.server from the vault image. We're defining the three volumes that vault will use, we're exposing the address of the server in case we want to access it within the container, exposing the port vault is using and, finally, setting the entry point to the command use to start a vault server. As you can see, the command needs a configuration file that we don't have yet. Let's go to *\<base_folder\>/etc/vault.server/config* and create the config.hcl file with the following content:

```
storage "file" {
        path = "/mnt/vault/data"
}

listener "tcp" {
        address = "0.0.0.0:8200"
        tls_disable = 1
}
```

Using this file, we're saying to Vault that we want a server opened at port 8200, that doesn't use https, and that will use the file system as a storage. If you want, you can use other [storages](https://www.vaultproject.io/docs/configuration/storage/index.html), being Consul the most popular one.

Now, we can bring up the container. Let's go to your working folder and type *docker-compose up*

You should see the result of starting vault in your console.
![Container startup log](/images/server.vault.up.png)

If you want to run the container in detached mode, type *docker-compose up -d*.

Now it's time to configure Vault. By default, Vault creates a token (the defatul authentication mechanism) for root access. We can use this token to make the initial set up of Vault. But first, we need to unseal Vault.

## Unsealing Vault

First of all, we need to initialise Vault. Initializing Vault means creating the root token we've already talked about and, much more important, creating the unsealing keys to, you can image, unseal Vault. to initialise Vault we use the command *vault init*. This will create 5 keys for us and we'll need to provide three of them to unseal Vault. To change this configuration, use the parameters *-key-shares* and *-key-threshold*. Let's then use 5 shares and 2 shares as threshold: *vault init -key-shares=5 -key-threshold=2*.

```
MacBook-Pro:TestVault vga$ vault init -key-shares=5 -key-threshold=2
Error initializing Vault: Put https://127.0.0.1:8200/v1/sys/init: http: server gave HTTP response to HTTPS client
```

Oops! We can't connect to Vault! This is because, by default, the vault cli tries to connect using https to localhost and using port 8200. In our case, localhost and port 8200 are correct, but we've disabled TLS, so we need to connect via http. Let's configure our terminal to connect to the right place: *export VAULT_ADDR=http://127.0.0.1:8200
*

Let's try to run the init command again. Now, you should see something like this:

```
MacBook-Pro:TestVault vga$ vault init -key-shares=5 -key-threshold=2
Unseal Key 1: JjPOZr3C27PkMUT+NMAANsE9LD/EMOxeg4LrazntoL29
Unseal Key 2: HEiabCxFrigrcoKBqKMD0SI2cIQqF8rIai1/7iMynQ2z
Unseal Key 3: GmjlrY/kYko2zwWMvG1Y5Ts3VZWEEg8dsqUx1Fab7R1f
Unseal Key 4: D1GSRtVE3vvw1Inbwiv5ohJK/nmJgAuIcISTQx7+N4za
Unseal Key 5: O4PkY3gMbHMOnSdMenyMyD21Au+zrv8VelihpDQPM+W6
Initial Root Token: f1682479-2a28-d577-c3de-521431116581

Vault initialized with 5 keys and a key threshold of 2. Please
securely distribute the above keys. When the vault is re-sealed,
restarted, or stopped, you must provide at least 2 of these keys
to unseal it again.

Vault does not store the master key. Without at least 2 keys,
your vault will remain permanently sealed.
```

As the message indicates, every time the server is re-sealed (using *vault seal*) or restarted we'll need to provide two of these keys. Save them in a secure place, encrypt them and make an spell to protect them.

And now, finally, we can unseal the server. Type *vault unseal*. You will see
```
MacBook-Pro:TestVault vga$ vault unseal
Key (will be hidden): 
```

Paste there one of the keys and repeat the process as many times as the threshold you used in the initialisation phase.

When you have finally unsealed the server, you should see something like this:

```
MacBook-Pro:TestVault vga$ vault unseal
Key (will be hidden): 
Sealed: false
Key Shares: 5
Key Threshold: 2
Unseal Progress: 0
Unseal Nonce: 
```

Awesome. It's time to configure Vault!

## Configure an admin user

You can use different [authentication backends](https://www.vaultproject.io/docs/auth/index.html) in Vault, like GitHub, Okta or LDAP. In this example, we're going to use username & password.

To make any action in this stage, we need to use the root token. Let's grab it from the result of the init command and use it: *vault auth \<token\>*

```
MacBook-Pro:TestVault vga$ vault auth f1682479-2a28-d577-c3de-521431116581
Successfully authenticated! You are now logged in.
token: f1682479-2a28-d577-c3de-521431116581
token_duration: 0
token_policies: [root]
```

Now we need to enable the authentication backend: *vault auth-enable userpass*

```
MacBook-Pro:TestVault vga$ vault auth-enable userpass
Successfully enabled 'userpass' at 'userpass'!
```

### Policies
Vault uses [policies](https://www.vaultproject.io/docs/concepts/policies.html) to grant and permit access to paths (everything in Vault is path based). So, the first thing we need to do is to create a policy for our new brand admin user. Create a file called adminpolicy.hcl with the following contents:

```
#authorization
path "auth/userpass/*" {
  capabilities = ["create", "read", "update", "delete", "list"]
}

#policies
path "sys/policy/*" {
  capabilities = ["create", "read", "update", "delete", "list"]
}
```

With this policy, a user will be able to manage users and policies, which is what we need by now. Add this policy using the following command:

```
MacBook-Pro:TestVault vga$ vault write sys/policy/admins policy=@"adminpolicy.hcl"
Success! Data written to: sys/policy/admins
```

As we already said, everything is a path in Vault, policies not being an exception. To write a new policy, we need to write in the path *sys/policy/\<name\>*. If we want, we can read the recently created policy:

```
MacBook-Pro:TestVault vga$ vault read sys/policy/admins
Key  	Value
---  	-----
name 	admins
rules	#authorization
	path "auth/userpass/*" {
	  capabilities = ["create", "read", "update", "delete", "list"]
	}

	#policies
	path "sys/policy/*" {
	  capabilities = ["create", "read", "update", "delete", "list"]
	}
```

It's time to create a user linked to this policy.

### Users

To create a user that uses this policy, we need to run the following command:

```
MacBook-Pro:TestVault vga$ vault write auth/userpass/users/admin password=abcd policies=admins
Success! Data written to: auth/userpass/users/admin
```

As always, we're writing to a path.

Now we can use this new user to create other users. Let's try that. First, we need to authenticate to Vault using this user

```
MacBook-Pro:TestVault vga$ vault auth -method=userpass username=admin password=abcd
Successfully authenticated! You are now logged in.
The token below is already saved in the session. You do not
need to "vault auth" again with the token.
token: b54fe4f4-060f-b68b-ea7c-ffddea53e7c2
token_duration: 2764800
token_policies: [admins default]
MacBook-Pro:TestVault vga$ 
```

Time to create a new user. We'd like to create a user with permissions to write secrets in his private space and secretes in a team space. Create a file called *\<username\>.hcl* (in my case username = vgaltes) with the following content (replace vgaltes with your username and team with your team name):

```
#user authentication
path "auth/userpass/users/vgaltes" {
  capabilities = ["update"]
}

#own secrets
path "secret/vgaltes/*" {
  capabilities = ["create", "read", "update", "delete", "list"]
}

#team secrets
path "secret/team/*" {
  capabilities = ["create", "read", "update", "delete", "list"]
}
```

With the first part of the policy, we're allowing the user to change her password. With the second part, we're allowing the user to manage his own space (well, a space named as his username), and with the third one we're allowing the user to manage the team space.

Time to add the policy:

```
MacBook-Pro:TestVault vga$ vault write sys/policy/vgaltes-policy policy=@"vgaltespolicy.hcl"
Success! Data written to: sys/policy/vgaltes-policy
```

Now we can crate a user associated to this policy:

```
MacBook-Pro:TestVault vga$ vault write auth/userpass/users/vgaltes password=abcd policies=vgaltes-policy
Success! Data written to: auth/userpass/users/vgaltes
```

User created!! Let's write and read some secrets.

## Managing secrets
We have now created a new user. We've sent an email to him telling that his user has been created and that his password is *abcd*. The password is not very secure, so we urge him to change it as soon as possible. Let's see what he can do.

First of all he need to authenticate into Vault. Easy peasy, we already know how to do that:

```
MacBook-Pro:TestVault vga$ vault auth -method=userpass username=vgaltes password=abcd
Successfully authenticated! You are now logged in.
The token below is already saved in the session. You do not
need to "vault auth" again with the token.
token: 2a6e67a9-9b3a-39a7-5b29-bc3cecf79151
token_duration: 2764800
token_policies: [default vgaltes-policy]
MacBook-Pro:TestVault vga$ 
```

Time to change the password. As always, we just need to write into a specific path:

```
vault write auth/userpass/users/vgaltes password=abcd1234
```

We can try now to authenticate using the old password. Hopefully it won't work:

```
MacBook-Pro:TestVault vga$ vault auth -method=userpass username=vgaltes password=abcd
Error making API request.

URL: PUT http://127.0.0.1:8200/v1/auth/userpass/login/vgaltes
Code: 400. Errors:

* invalid username or password
```

Awesome! Let's try with the new password. Now, we're not going to provide the password, so the cli will ask for it:

```
MacBook-Pro:TestVault vga$ vault auth -method=userpass username=vgaltes
Password (will be hidden): 
Successfully authenticated! You are now logged in.
The token below is already saved in the session. You do not
need to "vault auth" again with the token.
token: 12de59c3-a23e-7dea-eff0-42dfeb8b73fd
token_duration: 2764799
token_policies: [default vgaltes-policy]
```

Now we're ready to write our first secret. Let's start with something simple:

```
MacBook-Pro:TestVault vga$ vault write secret/vgaltes/hello value=world
Success! Data written to: secret/vgaltes/hello
```

Let's try to read it now:

```
MacBook-Pro:TestVault vga$ vault read secret/vgaltes/hello
Key             	Value
---             	-----
refresh_interval	768h0m0s
value           	world
```

Cool! We can do the same with the team space. Just change \<username\> for \<teamname\>.

Let's try to write to someone else's space:

```
MacBook-Pro:TestVault vga$ vault write secret/peter/hello value=world
Error writing data to secret/peter/hello: Error making API request.

URL: PUT http://127.0.0.1:8200/v1/secret/peter/hello
Code: 403. Errors:

* permission denied
```

As expected, we can't write to that path.

## Summary

We've set up a basic (but operational) Vault server. We know how to create policies and users, and how those users can log in into the system and create and read secrets. In the next article, we'll see how we can use Vault to create temporary users in a MySQL database.
