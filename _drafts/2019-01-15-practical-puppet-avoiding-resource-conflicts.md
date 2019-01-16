---
layout: single
title:  "Practical Puppet: Avoiding Resource Conflicts"
date:   2019-01-15 19:55:00 -0500
categories: puppet
---

One of the most common questions I get from coworkers new to Puppet is "how can I avoid resource conflicts?".

In an ideal world of [roles and profiles](https://puppet.com/docs/pe/2019.0/designing_system_configs_roles_and_profiles.html) and perfectly coded modules, resource conflicts should never
occur. But, given that we don't live in an ideal world, there are a few helpful functions to avoid the issue.

It is important to note that the functions listed below are still impacted by evaluation order (explained in greater detail later). These functions are not magic "fix all" functions, and
should be avoided if at all possible.

### `defined`

The [defined](https://puppet.com/docs/puppet/6.1/function.html#defined) function is built into Puppet and will return `true` if a given resource is defined within the catalog. The
function may be used within a conditional to check if a particular resource exists (yet).

```puppet
unless defined(File['/tmp/shared']) {
  # do stuff ...
  file { '/tmp/shared':
    ensure => 'directory',
  }
}
```

### `defined_with_params`

This function (and every function hereafter) is provided by the [puppetlabs/stdlib](https://forge.puppet.com/puppetlabs/stdlib) Forge module. As the name suggests, this function checks for a
resource with specific attributes.

```puppet
if defined_with_params(User['johndoe'], {'ensure' => 'present'}) {
  # do stuff ...
  user { 'johndoe':
    ensure => absent,
  }
}
```

Check the [puppetlabs/stdlib README](https://forge.puppet.com/puppetlabs/stdlib#defined_with_params) for more information.

### `ensure_resource`

This function is pure gold. Given a resource type, title, and attribute hash, this function will create a resource in the catalog only if it does not already exist. If the resource does
exist, but has different attributes, a duplicate resource definition error is thrown.

```puppet
ensure_resource('file', '/tmp/shared', {'ensure' => 'directory'})
```

You can also use pass in arrays to create multiple resources with a single function call. Check out the [README](https://forge.puppet.com/puppetlabs/stdlib#ensure_resource) for more info.

### `ensure_resources`

Essentially a souped-up version of the `ensure_resource` function that allows you to pass in a hash of resources. In my opinion, this function quickly makes your code unreadable, and
I tend to avoid it. However, it definitely has its place, particularly when describing resources within Hiera.

```puppet
ensure_resources('cron', {'backupjob' => { 'command' => '/usr/bin/backup', 'user' => 'root', 'hour' => 0 }, 'copyjob' => { 'command' => '/usr/sbin/copystuff', 'minute' => 30 } }, { 'ensure' => 'present' })
```

### `ensure_packages`

Similar to the previous function, but specifically for `package` resources. I use it for dependencies when a node's package manager cannot sort it out on its own (namely Windows).

```puppet
$my_pkgs = {
  'libcurl3' => { 'ensure' => 'present' },
  'zabbix' => { 'ensure' => '3.4.0' }
}

ensure_packages($my_pkgs)

```
## Impact of evaluation order

How and when resources & functions are evaluated during catalog compilation may impact the result of these functions. Puppet does not always evaluate classes and resources in the order in
which they are defined within a manifest. This evaluation-order independence has the potential to change the outcome of these functions and may make it seem like they are... well... broken.

Exactly how catalog compilation works gets into the nitty gritty of Puppet, and I won't pretend to completely understand how it works. Instead I will attempt to explain the issue as I
understand it--feel free to yell at me if I get anything wrong.

```puppet
# Evaluated first
file { '/tmp/shared':
  ensure => 'directory',
}

# Evaluated second
ensure_resource('file', '/tmp/shared', {'ensure' => 'directory'})

# This catalog will compile.
```
In the example above, the catalog will compile successfully, as the `file` resource is evaluated before the `ensure_resource` function. However, if the evaluation order is flipped, the
catalog will fail to compile, as the `ensure_resource` function already added the `file` resource to the catalog before the file resource declaration is evaluated.

```puppet
# Evaluated first
ensure_resource('file', '/tmp/shared', {'ensure' => 'directory'})

# Evaluated second
file { '/tmp/shared':
  ensure => 'directory',
}

# This catalog will fail
```

Evaluation order can quickly become an issue when the same resources are added to the catalog in two completely different, unrelated classes. This evaluation order issue can
largely be avoided by using these handy functions in all classes that define the resource, which may be a hindrance if your Puppet modules are developed by different people on different
teams.

All that said, the best solution is to use roles, profiles, and class/resource relationships correct, and avoid these functions altogether. Easier said than done.
