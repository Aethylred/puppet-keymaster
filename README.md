# Keymaster

[![Build Status](https://travis-ci.org/Aethylred/puppet-keymaster.svg?branch=master)](https://travis-ci.org/Aethylred/puppet-keymaster)

The Keymaster Puppet module is intended to manage the deployment and redeployment of keys, certificate, and other security tokens across Puppet nodes, services, applications, and users.

Keymaster will generate self-signed keys and deploy them, or can deploy keys that have been pre-generated and seeded into it's keystore.

Initially Keymaster will handle SSH keys and certificates for users and host SSH keys and certificates for nodes. The intent is to be sure that these keys are well known to the Puppet manifest and can be used and deployed through the Puppet infrastructure.

It is the intention that using Keymaster means that the content of a key or certificate not be stored in a Puppet manifest or Hiera data store.

This module does not install the ssh client, sshd service, or the OpenSSH packages. The [saz-ssh](https://forge.puppetlabs.com/saz/ssh) module is recommended for managing the ssh client and service.

## Background

This module implements and expands the OpenSSH key store and generation as described in this [article on ssh::auth](http://projects.puppetlabs.com/projects/1/wiki/Module_Ssh_Auth_Patterns).

This module is mostly derived from the good work that Ashley Gould did on [puppet-sshauth](https://github.com/ashleygould/puppet-sshauth), but a rework was required to meet the following goals:

* Update style to meed now Puppet standards as enforced by [puppet-lint](http://puppet-lint.com/)
* Use new code features in Puppet and the [Puppetlabs `stdlib`](https://forge.puppetlabs.com/puppetlabs/stdlib)
* To include a full suite of unit tests with [rspec-puppet](http://rspec-puppet.com/) and [Travis-CI](https://travis-ci.org/)
* To move SSH keys into a their own name space to allow planned features
* Explicitly failing with messages rather than letting exceptions fall through for Puppet to handle.

# Usage

## Installing the Module

Install the keymaster module into a Puppet manifest by:

* With [librarian-puppet](https://github.com/rodjek/librarian-puppet)
* The command line

    ```
    $ cd /etc/puppet/modules
    $ puppet module install Aethylred-keymaster
    ```

* Cloning from the git repository

    ```
    $ git clone https://github.com/Aethylred/puppet-keymaster.git /etc/puppet/modules/keymaster
    ```

## Setting up the Keymaster

The Keymaster module requires that a Puppetmaster be set up as the Keymaster by including the `keymaster` class. All other operations with the Keymaster library require that a Keymaster is defined somewhere in a Puppet manifest. The minimum requirement would be:

```puppet
include keymaster
```

## Defining an OpenSSH Key

An OpenSSH key can be defined anywhere as it creates a collection of stored and virtual resources that can be realised where they are required. This includes the use of keys stored on the Keymaster, and generating them when they do not exist. Declaring the key _only_ makes a key ready for use, and does not install or otherwise copy it to a node. A minimal key declaration would be:

```puppet
keymaster::openssh::key('key_name': )
```

## Assigning an OpenSSH Key to a User

A previously defined key (i.e. a key that has been defined in any part of a Puppet manifest) can be installed into a user's account. This copies the key's public and private parts to the user's `.ssh` directory. The minimal manifest to install a key in a user's account would be:

```puppet
user{'your_name_here': }
keymaster::openssh::key{'key_name': }
keymaster::openssh::deploy_pair{'key_name':
  user => 'your_name_here',
}
```

## Authorizing a Key to Access a User Account

The public part of a previously defined key can be injected into a user's `~/.ssh/authorized_keys` file to allow holders of the private part of that key to use SSH to access that user account. The minimal manifest to authorize a key would be:

```puppet
user{'your_name_here': }
keymaster::openssh::key{'key_name': }
keymaster::openssh::authorize{'key_name':
  user => 'your_name_here',
}
```

# Seeding Keys

If a key is not present in the Keymaster Server's key store, the `keymaster` class will generate self-signed keys as required. These keys will persist and be reissued where ever a Puppet manifest says they should be deployed. This makes it possible to be sure that the same key is deployed without defining, or even knowing, its contents.

In those cases where a known and previously generated key is required, create a directory that represents the key's name (`this_key` in the example below) in the keystore hierarchy and copy in the private and public parts of the key. For an OpenSSH key (where the private part is the file `key` and the public part is the file `key.pub`) the result should be similar to:

```
/var/lib/keymaster/openssh-+-a_key
                           |
                           +-another_key
                           |
                           +-this_key-+-key
                                      |
                                      +-key.pub
```

Once the keystore is pre-seeded with the known key then in the puppet manifest the following entry in any node manifest would define the resource (the parameters should be used to make puppet aware of the nature of the key's content):

```puppet
keymaster::openssh::key{'this_key': }
```

**NOTE:** When using a pre-seeded key it is not advisable to use the `force`, `mindate` or `maxdays` parameter otherwise Keymaster will revoke and regenerate these keys when they exceed the age defined by these parameters.

**NOTE:** When creating the directory that holds the key pair, substitute `_at_` for the `@` character. The `@` character may not be safely supported in some file systems.

# Classes

## `keymaster`

The `keymaster` class prepares a Puppet master to act as a Keymaster key store. This class prepares and secures a directory (`/var/lib/keymaster` by default) to store the files used by the other resources defined in this module. The `keymaster` class realises the virtual and stored classes created by the resources defined in this module, making them available to be deployed by Puppet.

### Parameters

#### `keystore_base`

This parameter sets the base directory where the Keymaster will store its collection of keys. The default is `/var/lib/keymaster`.

#### `keystore_openssh`

This parameter sets the base directory where the Keymaster will store its collection of OpenSSH keys. The default is `/var/lib/keymaster/openssh`.

#### `user`

This is the user that runs the Puppetmaster service, by default this is set to `puppet`. If Puppetmaster is being run under Apache or ngnix then this must be changed to match the user running these services.

#### `group`

This is the primary group of the user that runs the Puppetmaster service, by default this is set to `puppet`. If Puppetmaster is being run under Apache or ngnix then this must be changed to match the primary group of the user running these services.

# Public Resources

These resources are used in Puppet manifests and node definitions to deploy keys that have been defined with the Keymaster module.

## `keymaster::openssh::authorize`

This will deploy the public part of a key pair in to the targeted user's `~/.ssh/authorized_keys` file, which will allow user accounts holding the private part of the key SSH access to the targeted user's account.

The name of this resource must match the name of a `keymaster::openssh::key` resourced defined elsewhere in the Puppet manifest.

### Parameters

#### `user`

This parameter targets the user which will receive the public part of the key into their `~/.ssh/authorized_keys` file. The named user must be defined in a node's Puppet manifest. This is a required parameter.

#### `ensure`

This parameter ensures if the key authorization entry is present or absent in the targeted user's `~/.ssh/authorized_keys` file. The default is `present`. This may not work as expected if the named key ensure parameter is set as absent.

#### `options`

This sets the option string that can be set when a key is authorized to access a user account or service. The default is undefined, which sets no option string. This parameter will override the `option` parameter of the previously defined `keymaster::openssh::key` being authorized.

## `keymaster::openssh::deploy_pair`

This resource will deploy both the public and private parts of a key pair into the targeted user's `~/.ssh` directory. This will grant the targeted user access to other user accounts that have the public part of the key in their `~/.ssh/authorized_keys` file.

The name of this resource must match the name of a `keymaster::openssh::key` resourced defined elsewhere in the Puppet manifest.

### Parameters

#### `user`

This parameter targets the user which will receive the key pair into their `~/.ssh` directory. The named user must be defined in a node's Puppet manifest. This is a required parameter.

#### `ensure`

This parameter ensures that a key pair is present or absent in the targeted user's `~/.ssh` directory. The default is `present`. This may not work as expected if the named key ensure parameter is set as absent.

#### `filename`

This parameter sets the base file name that the key pair uses when deployed to nodes. The private part uses the file name, while the public part appends the `.pub` extension. The default is undefined, in which case the file name is derived from the keytype (either `id_rsa` or `id_dsa`). This parameter will override the `filname` parameter of the previously defined `keymaster::openssh::key` being deployed.

## `keymaster::openssh::key`

This resource defines an OpenSSH key and creates the virtual resources that are used to generate and deploy the key. In order to deploy this key to nodes defined in the Puppet manifest, there must be matching `keymaster::openssh::deploy_pair` or `keymaster::openssh::authorize` definitions with the same resource name.

### Parameters

#### `ensure`

This parameter defines if a key should be `present` or `absent`. When revoking a key, switching the ensure parameter to `absent` will set Puppet to actively remove keys from nodes. Just deleting or commenting out keymaster classes and resources will stop Puppet from managing those keys, but will not make puppet remove them. Accepted values are `present` and `absent`, the default value is present.

#### `filename`

This parameter sets the base file name that the key pair uses when deployed to nodes. The private part uses the file name, while the public part appends the `.pub` extension. The default is undefined, in which case the file name is derived from the keytype (either `id_rsa` or `id_dsa`).

#### `keytype`

This parameter accepts `rsa` or `dsa` to define key type. The `rsa` key type is recommended and the default value is `rsa`.

#### `length`

This sets the bit length of the key. Values of less than 2048 are not recommended. When the key type is set to `dsa` this value is overridden and set to 1024. The default value is 2048.

#### `maxdays`

When set, the Keymaster server will regenerate a key when it exceeds the age set by the `maxage` parameter on the next puppet run. Nodes that deploy the key will update the key on the next puppet run. There may be an interim period where systems and services may be disrupted as the key redeploys across managed systems. The default is undefined which does not trigger a key refresh. It is not recommended that the `maxdays` parameter is used with pre-seeded keys.

#### `mindate`

Setting `mindate` will regenerate the key if it was created before the specified date. There may be an interim period where systems and services may be disrupted as the key redeploys across managed systems. The default is undefined which does not trigger a key refresh. It is not recommended that the `mindate` parameter is used with pre-seeded keys.

#### `force`

This parameter forces a key to be regenerated by the Keymaster. There may be an interim period where systems and services may be disrupted as the key redeploys across managed systems. The default is `false` which does not trigger a key refresh. It is not recommended that the `force` parameter is used with pre-seeded keys.

#### `options`

This sets the option string that can be set when a key is authorized to access a user account or service. The default is undefined, which sets no option string.


# Private Resources

### Parameters

These private resources are used internally within the other Keymaster classes and resources. Calling them directly is unlikely to result in predictable outcomes.

## `keymaster::openssh::key::authorized_key`

This private resource is created as a stored resource by the `keymaster::openssh::key` resource that is realised by the `keymaster::openssh::authorize` resource and inserts the public part of a key in to a user's `~/.ssh/authorized_keys` file.

## `keymaster::openssh::key::deploy`

This private resource is created as a stored resource by the `keymaster::openssh::key` resource that is realised by the `keymaster::openss::deploy_pair` resource where it deploys the public and private parts of a key into a user's `~/.ssh` directory.

## `keymaster::openssh::key::generate`

This private resource is created as a stored resource by the `keymaster::openssh::key` resource that is realised by the `keymaster` class where it verifies if a key exists in the Keymaster keystore, and generates a key if the named key does not exist.

# To Do

* Manage host certificates for SSH (with saz-ssh)
* Manage X509 certificates

# References

* [ssh::auth](http://projects.puppetlabs.com/projects/1/wiki/Module_Ssh_Auth_Patterns) 
* [saz-ssh](https://forge.puppetlabs.com/saz/ssh) This module is recommended for installing and configuring `ssh` and the `sshd` service. This module has been used to test the Keymaster module.
* [ashleygould puppet-sshauth](https://github.com/ashleygould/puppet-sshauth)
* [Aethylred puppet-sshauth](https://github.com/aethylred/puppet-sshauth)
* [vurbia pppet-sshauth](https://github.com/vurbia/puppet-sshauth)

# Attribution

## puppet-blank

This module is derived from the [puppet-blank](https://github.com/Aethylred/puppet-blank) module by Aaron Hicks (aethylred@gmail.com)

This module has been developed for the use with Open Source Puppet (Apache 2.0 license) for automating server & service deployment.

* http://puppetlabs.com/puppet/puppet-open-source/