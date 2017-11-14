---
layout: post
title: How to Write Better Chef Code
tags:
  - Software Engineering
---

Authoring and endlessly refactoring Square's Mac Chef has shaped my opinions of what a node bringup
workflow should look like. Here's what I view as the way to write Chef well for the use case
outlined in the next section.

## Background
Square uses Chef for its embedded build and test nodes. We have many types of readers and they are
tested many ways. Different types of build nodes build sub-parts of the end artifact, resulting in
a many-to-many relationship between build and test workers.

## Goals
* There are two steps for node bringup. Imaging and configuration. Both should take one step each.
* The Chef repo must be structured so that developers that do not know Chef well can contribute.
  This is made possible if the repo achieves both **configuration** and **convention**.
* Decouple as much as possible.

## Non-Goals
* Performance. These nodes are not imaged on-demand. They are meant to sit for a long time and
  update with [Chef client](https://docs.chef.io/chef_client.html).

## Cookbook Principles

### Attributes Provide Sensible Defaults
Every cookbook needs to include default attributes to ensure it can run in without the presence of
other cookbooks, even though these default attributes might make assumptions, such as the presence
of a given user.

### Use Hashes and not Arrays
Hashes will make your cookbooks smarter and decouple your attributes. Chef will probably change the
way it merges attributes at runtime. I would rather avoid all of that and keep consistent behavior
that I tightly control. Here are two attribute structures:

Example 1:

```json
"jenkins_labels": [
  "mac",
  "build",
  "xcode"
]
```

Example 2:

```json
"jenkins_labels": {
  "mac": true,
  "build": true,
  "xcode": true
}
```

Example 2 looks wonky, but Example 1 can easily be overridden somewhere else. If you want "smarter"
cookbooks that are both convention and configuration, use hashes.

Example 3:

```json
"jenkins_labels": {
  "test": true
}
```

Example 3 can easily be merged with example 2 to make a node that's a test node with Xcode
installed.

### One Cookbook, One Task
Keep cookbooks lightweight, but group similar tasks such as installing all Python packages into one
cookbook. The only exception to this rule should be role cookbooks.

### Use Role Cookbooks
Role cookbooks have a single recipe that lists other recipes using `include_recipe`.

### Avoid Running Multiple Versions of Cookbooks in Production
Multiple versions of cookbooks are a crutch when attributes are not properly set up. If two versions
of the same package need to be live, then two roles' attributes should supply these versions, with
the cookbook providing a sensible default. Updating code for each platform and then deploying a new
version is much more sensible.

### Use Guard Clauses
```ruby
execute 'move file from src to dst if dst exists' do
  command "mv #{src} #{dst}"
  not_if  "ls #{dst}"
end
```

### Control Flow with `notifies` Wisely
`notifies` can introduce uncomfortable dependencies into your cookbooks.

```ruby
command 'do something' do
  notifies :restart, "service['service-in-recipe-two']"
end
```

This recipe now depends on `recipe-two`, which isn't ideal if we want to do a simple litmus test of
whether or not the recipe works in a vacuum. How do we fix that?

```ruby
command 'do something' do
  notifies :restart, "service['service-in-another-recipe']", :delayed if node.recipe? 'recipe-two'
end
```

Make sure to include guard clauses for your `notifies` statements.

## Role Principles

### Do Not Avoid Roles
Just because the Chef community is infatuated with role cookbooks does not mean roles themselves
should be discarded. The role cookbook itself provides versioning, but git also provides versioning
for roles.

On one hand, the advantage of having roles is that they provide another layer of attribute
precedence in the merge hierarchy. On the other hand, I care about readability and giving a complete
picture of what is going on more than the extra layer. Roles provide a way to specify all of the
settings needed to configure a node. Role cookbooks can only paint part of the picture by specifying
attributes at runtime, but the role itself can tell the developer much more with the role settings
found [here](https://docs.chef.io/roles.html).

### One Role, One Role Cookbook
Keep a simple 1:1 relationship in your many-to-many madness.

### Role Inheritance
```
                              --------------------------
                       |------| mac-build-machine.json |
-----------------      |      --------------------------
| mac-base.json |<-----|
-----------------      |      -------------------------
                       |------| mac-test-machine.json |
                              -------------------------
```
Keep roles simple but nested. Having parent roles helps ensure consistency across your stack. We
wouldn't want different authentication methods to our nodes, for example.

## Don't Forget Environments
Use environments to setup a wide net of attribute settings that encompasses multiple roles. For
example, hardware nodes belong in the hardware CI environment, where they can access the info for
the hardware Jenkins instance. Being able to specify a `-E` flag during `knife bootstrap` keeps me
sane as well.

## Testing Principles

### Complete Integration Tests with [Test Kitchen](https://github.com/test-kitchen/test-kitchen)
Chef unit tests serve to enforce that cookbook syntax is correct and the developer is not making
operating system-related mistakes. Real Chef testing needs to include end-to-end integration tests,
where the entire role.

I prefer to use [Serverspec](https://github.com/test-kitchen/busser-serverspec).
