---
layout: post
title: "Vagrant &amp; Chef-Zero"
date: 2015-02-15 19:28:36 +0530
comments: true
categories: [infrastructure, chef, vagrant, rails]
---

Chef cookbooks quickly get complicated as with deployment strategies change
over time. You may waste weeks of your time getting things right. One way to
reduce this overhead is to start with a simple one machine build using 
Vagrant and Chef-Zero to provision it, then move to a multi-machine build and
evolve accordingly. As Captain Picard said, we don't jump directly to warp
speed, we start with an impulse then engage to warp drive.

Automating DevOps can be time consuming in the short term, but it is essential
that it be done properly to sustain continuous delivery on a project.
Provisioning machines and making sure that provisioning is idempotent is one
of the more time consuming activities involved in setting up automation.

Vagrant and Chef-Zero are two of the tools we use daily to automate the setup
of deployment environments. Working with these tools reduces our time by a big
factor because of their local runtime and cookbooks caching on developer
machine, so no unnecessary network latency while provisioning. We spend
majority of time in writing cookbook recipes, as cookbook development cycle is
reduced three simple steps write a recipe, provision virtual machine and check
if changes are applied to virtual machine. Initial step this simplified
development cycle requires developer input in writing cookbook recipes whereas
second is automated with vagrant (which requires either chef-solo or chef-zero
to converge chef recipes); and third can be automated using test-kitchen
(an integration tool for developing and testing infrastructure code).

Previous to chef-zero announcement we used chef-solo to provision virtual
machines. Chef-Zero in addition chef-solo features includes improvements such
as in-memory and fast-start Chef server intended for development purposes.
Although Chef Team have no immediate plans to deprecate chef-solo, they will
eventually remove it from Chef. A good reason which made us to shift from
chef-solo to chef-zero.
(source From [Solo to Zero](https://www.chef.io/blog/2014/06/24/from-solo-to-zero-migrating-to-chef-client-local-mode/))


To leverage the true power of chef-zero a compatible vagrant configuration is
needed. We use following Vagrant configuration as our template.

{% highlight ruby %}
VAGRANTFILE_API_VERSION = '2'
 
Vagrant.require_version '>= 1.5.0'
 
Vagrant.configure(VAGRANTFILE_API_VERSION) do |config|
  config.omnibus.chef_version          = :latest if Vagrant.has_plugin?('vagrant-omnibus')
  config.hostmanager.enabled           = true
  config.hostmanager.manage_host       = true
  config.hostmanager.ignore_private_ip = false
  config.hostmanager.include_offline   = true
  config.berkshelf.enabled             = true
  config.ssh.shell                     = 'bash -c "BASH_ENV=/etc/profile exec bash"'
 
  config.vm.define 'application-staging' do |node|
    node.vm.box      = 'ubuntu/trusty64'
    node.vm.hostname = 'application.staging'
    node.vm.network :private_network, ip: '33.33.33.10'
  end
 
  config.vm.provider :virtualbox do |virtualbox|
    virtualbox.name   = 'application-staging'
    virtualbox.memory = 1024
    virtualbox.cpus   = 2
  end
 
  config.vm.provision :chef_zero do |chef|
    chef.node_name      = 'application-staging'
    chef.cookbooks_path = ['.chef/cookbooks']
    chef.nodes_path     = '.chef/nodes'
    chef.roles_path     = '.chef/roles'
    chef.data_bags_path = '.chef/data_bags'
 
    chef.add_recipe 'staging'
  end
end
{% endhighlight %}


This Vagrant configuration requires a some vagrant plugins to be pre-installed.

* vagrant-chef-zero
* vagrant-berkshelf
* vagrant-hostmanager

We use Berkshelf to manage our cookbook dependencies. Using Berkshelf gives us a 
project-like structure to our cookbook.

    staging
    ├── .chef
    │   ├── cookbooks
    │   │   └── .keep
    │   ├── data_bags
    │   │   └── users
    │   │       └── deployer.json
    │   ├── knife.rb
    │   ├── nodes
    │   │   └── .keep
    │   └── roles
    │       └── .keep
    ├── Berksfile
    ├── Gemfile
    ├── README.md
    ├── Vagrantfile
    ├── attributes
    │   └── default.rb
    ├── charms
    ├── files
    │   └── default
    │       ├── deployer.key
    │       ├── deployer.pub
    │       └── nginx.access.log
    ├── libraries
    │   └── .keep
    ├── metadata.rb
    ├── providers
    │   └── .keep
    ├── recipes
    │   ├── application.rb
    │   ├── base.rb
    │   ├── database.rb
    │   ├── default.rb
    │   ├── rails_stack.rb
    │   ├── users.rb
    │   └── web_server.rb
    ├── resources
    │   └── .keep
    └── templates
        └── default
            ├── database.yml.erb
            ├── nginx.conf.erb
            └── secrets.yml.erb


##Conclusion

It's hard to conclude an infrastructure recipe, as software changes so does
infrastructure. Starting with a smaller local build then evolving to complex
remote build requires a lot of changes made to cookbooks. Before you converge
these cookbooks to remote machine, provision a local virtual machine using
Vagrant and Chef-Zero to ensure their correctness. Remember, all big things
have small beginnings.
