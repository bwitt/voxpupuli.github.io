---
layout: post
title: Module best practices
date: 2026-11-15
summary: Preferred patterns for new and modernized Puppet modules, and the older patterns they replace.
---

Preferred patterns for new code and for modernizing existing modules. Rules
already enforced by CI and the [lint plugins][lint-plugins] aren't repeated
here. See also the [pull request review guide](/docs/reviewing_pr/).

| Old | New |
| --- | --- |
| `inherits foo::params` | Module data in `data/` |
| `hiera()`, `hiera_hash()`, `hiera_array()` | Automatic parameter lookup |
| ERB templates | EPP templates with typed parameters |
| A parameter per config option | A `Hash` rendered with `stdlib::to_yaml` and friends |
| `String` everywhere | `Stdlib::Port`, `Stdlib::Absolutepath`, `Enum`, ... |
| `include` for internal subclasses | `contain` |
| `create_resources()` | `each` |
| Plain `String` secrets | `Sensitive` |
| `exec` with `curl` or `wget` | [puppet/archive][archive] |

## Data in modules instead of params.pp

Add a `hiera.yaml` to the module root:

```yaml
---
version: 5
defaults:
  datadir: 'data'
  data_hash: 'yaml_data'
hierarchy:
  - name: 'Operating system major release'
    path: '%{facts.os.name}-%{facts.os.release.major}.yaml'
  - name: 'Operating system'
    path: '%{facts.os.name}.yaml'
  - name: 'Operating system family'
    path: '%{facts.os.family}.yaml'
  - name: 'Common'
    path: 'common.yaml'
```

Put OS-specific values in `data/`:

```yaml
# data/RedHat.yaml
---
foo::package_name: 'foo-server'
```

[openvox-strings][openvox-strings] 7.0+ documents defaults from the class
signature and `data/common.yaml`, but not from OS-specific files. Describe
those in the `@param` text:

```puppet
# @param package_name
#   Name of the package. Defaults to `foo` on Debian and `foo-server` on RedHat.
# @param config_dir
#   Configuration directory.
# @param config_file
#   Main configuration file.
class foo (
  String[1]            $package_name,
  Stdlib::Absolutepath $config_dir  = '/etc/foo',
  Stdlib::Absolutepath $config_file = "${config_dir}/foo.conf",
) {
  # ...
}
```

* Values derived from other parameters belong in the class, not in Hiera.
* Declare merge behaviour for `Hash` and `Array` parameters with
  `lookup_options` in `data/common.yaml`:

  ```yaml
  ---
  lookup_options:
    foo::config:
      merge: deep
  ```

* Don't call `lookup()` or `hiera*()` in manifests.
* Removing `params.pp` is `backwards-incompatible` if anything references
  `$foo::params::*`.

## EPP instead of ERB

Declare typed parameters and pass values explicitly. Don't read class variables
or facts inside the template:

```erb
<%- |
  Stdlib::Port     $port,
  Array[String[1]] $listen,
| -%>
port = <%= $port %>
<% $listen.each |$address| { -%>
listen = <%= $address %>
<% } -%>
```

```puppet
file { $foo::config_file:
  ensure  => file,
  content => epp('foo/foo.conf.epp', {
    'port'   => $foo::port,
    'listen' => $foo::listen,
  }),
}
```

When converting an existing ERB template, keep the output identical to avoid
config changes and service restarts.

## Structured config from data

For YAML, JSON or TOML config files, take a `Hash` and serialize it with
`stdlib::to_yaml`, `stdlib::to_json_pretty` or `stdlib::to_toml`. This
supports every upstream option without a new parameter for each:

```puppet
class foo (
  Hash[String[1], Any] $config = {},
) {
  file { $foo::config_file:
    ensure  => file,
    content => stdlib::to_yaml($config),
  }
}
```

## Specific data types

Use the most specific type that fits:

* [stdlib types][stdlib-types] such as `Stdlib::Port`, `Stdlib::Absolutepath`
  and `Stdlib::HTTPUrl`
* `Enum` with the values the service accepts
* Type aliases in `types/` for complex, reused structures

## contain

Use `contain` for subclasses so that ordering against `Class['foo']` applies
to everything inside it:

```puppet
class foo {
  contain foo::install
  contain foo::config
  contain foo::service

  Class['foo::install']
  -> Class['foo::config']
  ~> Class['foo::service']
}
```

## Iteration instead of create_resources

```puppet
class foo (
  Hash[String[1], Hash] $vhosts = {},
) {
  $vhosts.each |$name, $attributes| {
    foo::vhost { $name:
      * => $attributes,
    }
  }
}
```

## Sensitive

Use `Sensitive` for passwords, tokens and keys. Unwrap only where used, and
wrap the result:

```puppet
class foo (
  Sensitive[String[1]] $password,
) {
  file { $foo::config_file:
    ensure  => file,
    content => Sensitive(epp('foo/foo.conf.epp', {
      'password' => $password.unwrap,
    })),
  }
}
```

Users can convert plain Hiera values:

```yaml
lookup_options:
  foo::password:
    convert_to: 'Sensitive'
```

## Avoid

* `defined()`: depends on evaluation order. Offer a `manage_*` parameter
  instead.
* Resource collectors (`Package <| |>`): realize all matching virtual
  resources and can override other modules' attributes.
* `exec` with `curl` or `wget`: use [puppet/archive][archive].

## Acceptance tests

Add acceptance tests for new features and bug fixes where possible.
[voxpupuli-acceptance][acceptance] provides an idempotency check:

```ruby
require 'spec_helper_acceptance'

describe 'foo' do
  it_behaves_like 'an idempotent resource' do
    let(:manifest) { 'include foo' }
  end

  describe service('foo') do
    it { is_expected.to be_enabled }
    it { is_expected.to be_running }
  end
end
```

[lint-plugins]: https://github.com/voxpupuli/voxpupuli-puppet-lint-plugins
[openvox-strings]: https://github.com/voxpupuli/openvox-strings
[stdlib-types]: https://github.com/puppetlabs/puppetlabs-stdlib/blob/main/REFERENCE.md#data-types
[archive]: https://github.com/voxpupuli/puppet-archive
[acceptance]: https://github.com/voxpupuli/voxpupuli-acceptance
