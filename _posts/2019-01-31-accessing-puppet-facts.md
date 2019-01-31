---
layout: single
classes: wide
title:  "The better way to access Puppet facts"
date:   2019-01-31 00:16:00 -0500
categories: puppet
---

I spend a *lot* of time working with facts in Puppet. I've probably spent more time figuring out the best way to manage custom external facts unique to our environment (node lifecycle
status, datacenter, etc.) this past year than anything else in the Puppet universe. So I was rather surprised to find out there are better ways to access node facts within manfiests than
using the standard `$facts` hash.

Most of the information that follows came from a recent [office-hours](https://puppet.com/community/office-hours) conversation on the Puppet Community Slack. If you haven't already, I highly
suggest you check out the Puppet Community Slack channel. It is a great place to bounce ideas off of Puppet engineers and other members of the community.

## The fact variables

### Classic top-scope variables

Node facts have always been accessible in Puppet using top scope variables. For example, the fact `kernel` is automatically available within a manifests in the variable `$kernel`.

```puppet
if $kernel == 'Linux' {
  # do stuff...
}
```

The downside of using variables is that it isn't immediately obvious you are using a node fact in your manifest. May not be an issue for common well-known facts, but for custom facts it'll
cause your manifest to be difficult to read. Many developers will explicity call a top-scope variable (ie. `$::kernel`) as a hint, but it still doesn't completely resolve the issue.

### The `$facts` hash

In Puppet 3.5 (I think), PuppetLabs added the `$facts` hash. Node facts are merged together into a single hash, with may be accessed using the fact name as the key.

```puppet
unless $facts['kernel'] == 'Linux' {
  # do non-Linux stuff...
}
```

This is how I've always accessed facts within my manifests, and so far I haven't had any trouble. *However,* while this approach works for most facts, it does not work well for structured facts.

Take a node that has the following custom structured fact:

```json
{
  "cmdb_data": {
    "datacenter": "virginia"
  }
}
```

If I wanted to retrieve the node's datacenter within my manifest, I could simply use the variable `$facts['cmdb_data']['datacenter']`. On nodes that do *not* have the custom structured
fact (a newly provisioned node, for example), I had assumed the variable would simply have the value `undef`. Turns out this is not the case, in reality the catalog will blow up and fail to
compile.

## The functions that help

Luckily there are a few functions available to help you access structured facts.

### `dig`

Similar to Ruby's built-in `dig` function, the Puppet variant allows you to "dig" into complex data structures. Refer to [Puppet's function
documentation](https://puppet.com/docs/puppet/6.2/function.html#dig) for more information.

Example fact:

```json
{
  "node_meta": {
    "owner": {
      "name": "johndoe"
    },
    "datacenter": "virginia"
  }
}
```

```puppet
$facts.dig('node_meta', 'owner', 'name') # returns: 'johndoe'
$facts.dig('node_meta', 'enclosure', 'rack') # returns: undef

$facts['node_meta']['enclosure']['rack'] # fails with catalog compilation error
```

### `fact`

A relatively new entry to the [puppetlabs/stdlib forge module](https://forge.puppet.com/puppetlabs/stdlib#fact), the `fact` function behaves similarly to the `dig` function, but is explicity
used for facts. The function also supports dot notation, making it relatively compact and easy to read.

```puppet
$server_owner = fact('node_meta.owner.name')
```

### `get`

`get` is a new function added in Puppet 6. It provides generally the same functionality as `dig` and `fact`, but has the ability to return a default value if the value isn't present
in the data structure. Like the `fact` function, this function uses dot notation for navigation values.

As an aside, for those who are huge fans of stdlib's `pick` function, the `get` function has the potential to replace `pick` in many situations.

```puppet
$facts.get('node_meta.owner.name', 'no_owner_information')
$server_owner = get($facts, 'node_meta.owner.name', 'no_owner_information')
```

The `get` function also has an optional lambda for more advanced error handing should the value not exist. More information is available in [Puppet's function
documentation](https://puppet.com/docs/puppet/6.2/function.html#get).

## TL;DR

You should probably be using the `dig`, `fact`, or `get` functions to retrieve fact values rather than top-scope variables or the `$facts` hash.

If you are on Puppet 6, the `get` function is a no-brainer in my opinion. If you can't make the jump to Puppet 6, my vote would be the `fact` function available in puppetlabs/stdlib.
`dig` is my least favorite of the three (not a fan of the array notation), but it will still get the job done.
