OpenDKIM Cookbook
=================
[![Cookbook Version](https://img.shields.io/cookbook/v/opendkim.svg?style=flat)](https://supermarket.chef.io/cookbooks/opendkim)
[![Dependency Status](http://img.shields.io/gemnasium/onddo/opendkim-cookbook.svg?style=flat)](https://gemnasium.com/onddo/opendkim-cookbook)
[![Code Climate](http://img.shields.io/codeclimate/github/onddo/opendkim-cookbook.svg?style=flat)](https://codeclimate.com/github/onddo/opendkim-cookbook)
[![Build Status](http://img.shields.io/travis/onddo/opendkim-cookbook.svg?style=flat)](https://travis-ci.org/onddo/opendkim-cookbook)
[![Coverage Status](http://img.shields.io/coveralls/onddo/opendkim-cookbook.svg?style=flat)](https://coveralls.io/r/onddo/opendkim-cookbook?branch=master)
[![Inline docs](http://inch-ci.org/github/onddo/opendkim-cookbook.svg?branch=master&style=flat)](http://inch-ci.org/github/onddo/opendkim-cookbook)

Installs and configures [OpenDKIM](http://www.opendkim.org/): Open source implementation of the DKIM (Domain Keys Identified Mail) sender authentication system.

Requirements
============

## Supported Platforms

This cookbook has been tested on the following platforms:

* Amazon Linux
* CentOS
* Debian
* Fedora
* FreeBSD
* RedHat
* Ubuntu

Please, [let us know](https://github.com/onddo/opendkim-cookbook/issues/new?title=I%20have%20used%20it%20successfully%20on%20...) if you use it successfully on any other platform.

## Required Cookbooks

* [yum-epel](https://supermarket.chef.io/cookbooks/yum-epel)

## Required Applications

* Ruby `1.9.3` or higher.

Attributes
==========

| Attribute                  | Default      | Description                  |
|----------------------------|:------------:|------------------------------|
| `node['opendkim']['conf']` | *calculated* | OpenDKIM configuration hash. |

## Platform Support Related Attributes

Some cookbook attributes are used internally to support the different platforms. Surely you want to change them if you want to support new platforms or want to improve the support of some platforms already supported.

| Attribute                                 | Default               | Description                       |
|-------------------------------------------|:---------------------:|-----------------------------------|
| `node['opendkim']['conf_file']`           | *calculated*          | OpenDKIM Configuration file path.
| `node['opendkim']['require_yum_epel']`    | *calculated*          | Whether to include `yum-epel` recipe.
| `node['opendkim']['service']['name']`     | *calculated*          | OpenDKIM system service name.
| `node['opendkim']['service']['supports']` | *calculated*          | OpenDKIM service supported actions.
| `node['opendkim']['packages']['tools']`   | *calculated*          | OpenDKIM tools package name as array (currently unused).
| `node['opendkim']['packages']['service']` | `%w(opendkim)`        | OpenDKIM daemon package name as array.
| `node['opendkim']['run_dir']`             | `'/var/run/opendkim'` | OpenDKIM run directory used for the pidfile and as home for the system user.
| `node['opendkim']['user']`                | `'opendkim'`          | OpenDKIM system user name.
| `node['opendkim']['group']`               | `'opendkim'`          | OpenDKIM system group.

Recipes
=======

## opendkim::default

Installs and configures OpenDKIM.

Usage Examples
==============

## Including in a Cookbook Recipe

You can simply include it in a recipe:

```ruby
include_recipe 'opendkim'
```

Don't forget to include the `opendkim` cookbook as a dependency in the metadata.

```ruby
# metadata.rb
# [...]

depends 'opendkim'
```

## Including in the Run List

Another alternative is to include the default recipe in your *Run List*:

```json
{
  "name": "mail.onddo.com",
  "[...]": "[...]",
  "run_list": [
    "recipe[opendkim]"
  ]
}
```

## Reading the Key from a Chef Vault Bag

This is a complete example that reads the DKIM key from a chef vault bag using the [`chef-vault`](https://supermarket.chef.io/cookbooks/chef-vault) cookbook. The *txt* field is completely optional.

For more information about this configuration options, see [opendkim.conf(5)](http://www.opendkim.org/opendkim.conf.5.html) and [opendkim(8)](http://www.opendkim.org/opendkim.8.html).

```ruby
main_domain = 'example.com'
selector = '20150512'
key_name = "#{selector}._domainkey.#{main_domain} "\

# Configure OpenDKIM

# Defines a table that will be queried to convert key names to sets of data of
# the form (signing domain, signing selector, private key). The private key can
# either contain a PEM-formatted private key, a base64-encoded DER format
# private key, or a path to a file containing one of those.
default['opendkim']['conf']['KeyTable'] = 'refile:/etc/opendkim/KeyTable'

# Defines a dataset that will be queried for the message sender's address
# to determine which private key(s) (if any) should be used to sign the
# message. The sender is determined from the value of the sender
# header fields as described with SenderHeaders above. The key for this
# lookup should be an address or address pattern that matches senders;
# see the opendkim.conf(5) man page for more information. The value
# of the lookup should return the name of a key found in the KeyTable
# that should be used to sign the message. If MultipleSignatures
# is set, all possible lookup keys will be attempted which may result
# in multiple signatures being applied.
default['opendkim']['conf']['SigningTable'] =
  'refile:/etc/opendkim/SigningTable'

# We create the Key Table and Signing Table files

directory '/etc/opendkim' do
  mode '00755'
end

file '/etc/opendkim/KeyTable' do
  mode '00644'
  content(
    "#{key_name} "\
    "#{main_domain}:#{selector}:"\
    "/etc/opendkim/keys/#{main_domain}/#{selector}.private\n"
  )
end

file '/etc/opendkim/SigningTable' do
  mode '00644'
  content "*@#{main_domain} #{key_name}\n"
end

# Install OpenDKIM

include_recipe 'opendkim'

# Read DKIM keys from chef-vault

# node#save avoids chef-vault chicken & egg problem (a bit tricky)
node.save unless Chef::Config[:solo]
include_recipe 'chef-vault'
key = chef_vault_item('dkim_keys', main_domain)

# Create the credential files

directory "/etc/opendkim/keys/#{main_domain}" do
  owner node['opendkim']['user']
  group node['opendkim']['group']
  recursive true
end

file "/etc/opendkim/keys/#{main_domain}/#{selector}.private" do
  owner node['opendkim']['user']
  group node['opendkim']['group']
  mode '00640'
  sensitive true if Chef::Resource.method_defined?(:sensitive)
  content key['private']
end

# The txt is optional
file "/etc/opendkim/keys/#{main_domain}/#{selector}.txt" do
  owner node['opendkim']['user']
  group node['opendkim']['group']
  mode '00644'
  content key['txt']
  only_if { key['txt'].is_a?(String) }
end
```

The vault bag content example:

```json
{
  "id": "example.com",
  "private": "-----BEGIN RSA PRIVATE KEY-----\n [...] \n-----END RSA PRIVATE KEY-----\n",
  "txt": "v=DKIM1; k=rsa; p=MIGfMA0GCSqGSIb3DQEBAQUAA4GNADCBiQKB [...]"
}
```

The knife command to create the vault bag item:

    $ knife vault create dkim_keys example.com [...]

See the [Chef-Vault documentation](https://github.com/Nordstrom/chef-vault/blob/master/README.md) to learn how to create chef-vault bags.

## Integrate OpenDKIM with Postfix

We are using the [`postfix-full`](https://supermarket.chef.io/cookbooks/postfix-full) cookbook in this example:

```ruby
opendkim_port = 8891

# Configure Postfix
node.default['postfix']['main']['milter_protocol'] = 2
node.default['postfix']['main']['milter_default_action'] = 'accept'
node.default['postfix']['main']['smtpd_milters'] =
  "inet:localhost:#{opendkim_port}"
node.default['postfix']['main']['non_smtpd_milters'] =
  "inet:localhost:#{opendkim_port}"

# [...]
include_recipe 'postfix-full'

# Configure OpenDKIM
default['opendkim']['conf']['Mode'] = 'sv'
default['opendkim']['conf']['Socket'] =
  "inet:#{opendkim_port}@localhost"

# [...]
include_recipe 'opendkim'
```

## DNS Resource Record TXT Example

This a DNS TXT record example based on the examples above:

```
20150512._domainkey.example.com. 21599 IN TXT "v=DKIM1; k=rsa; p=MIGfMA0GCSqGSIb3DQEBAQUAA4GNADCBiQKB [...]"
```

## Testing Your Email DKIM Configuration

You can send an empty email to [check-auth@verifier.port25.com](mailto:check-auth@verifier.port25.com) to check that everything works correctly.

Testing
=======

See [TESTING.md](https://github.com/onddo/opendkim-cookbook/blob/master/TESTING.md).

Contributing
============

Please do not hesitate to [open an issue](https://github.com/onddo/opendkim-cookbook/issues/new) with any questions or problems.

See [CONTRIBUTING.md](https://github.com/onddo/opendkim-cookbook/blob/master/CONTRIBUTING.md).

TODO
====

See [TODO.md](https://github.com/onddo/opendkim-cookbook/blob/master/TODO.md).


License and Author
=====================

|                      |                                          |
|:---------------------|:-----------------------------------------|
| **Author:**          | [Raul Rodriguez](https://github.com/raulr) (<raul@onddo.com>)
| **Author:**          | [Xabier de Zuazo](https://github.com/zuazo) (<xabier@onddo.com>)
| **Copyright:**       | Copyright (c) 2015, Onddo Labs, SL. (www.onddo.com)
| **License:**         | Apache License, Version 2.0

    Licensed under the Apache License, Version 2.0 (the "License");
    you may not use this file except in compliance with the License.
    You may obtain a copy of the License at
    
        http://www.apache.org/licenses/LICENSE-2.0
    
    Unless required by applicable law or agreed to in writing, software
    distributed under the License is distributed on an "AS IS" BASIS,
    WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
    See the License for the specific language governing permissions and
    limitations under the License.
