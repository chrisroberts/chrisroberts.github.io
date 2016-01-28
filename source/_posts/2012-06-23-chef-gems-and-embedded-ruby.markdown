---
title: "Chef gems and embedded ruby"
date: 2012-06-23 11:27
tags: [chef_gem, chef, recipes, ruby, devops]
---

Backporting the Chef Gem.

## ChefGem Resource

The [ChefGem resource](http://wiki.opscode.com/display/chef/Resources#Resources-Differencesbetweenchefgemandgempackageresources)
was introduced in 0.10.9 to provide an easy way to install and make use of gems
within Chef. In the past gems required by recipes had to be installed during
compile time which added a handful of steps that had to be repeated every time
a gem was required by a recipe. The ChefGem resource automates these steps
making it much easier to provide gems required by Chef recipes.

Along with automating the steps required to make the gem useful at runtime, the
ChefGem resource also attempts to install gems into Chef's embedded ruby if it
is running via an omnibus installed Chef. This is done to keep Chef and its
dependencies isolated from the system at large. The ChefGem provider does some
inspection to determine where it is running and based on location will install
ChefGems into the embedded ruby or the system ruby. Likewise, the GemPackage
provider will check if it is running via an omnibus install and ensure that
packages get installed into the system ruby, and falls back to the embedded
ruby if unavailable.

### ChefGem Issues

With the introduction of ChefGem, a few issues are also introduced:

1. The new resource introduction is not backwards compatibile.
2. Proper omnibus detection does not work properly.
3. Gems may be unintentionally installed into embedded ruby

The first issue, and arguably the most important, restricts the ability to
implement recipes using the ChefGem resource until Chef has been upgraded
to at least 0.10.9. It also means that omnibus installs are not viable until
recipes have been converted to use ChefGem, as gems required by recipes will
not be available if a system ruby is in use. This can be problematic as recipe
updates plus Chef upgrades must be a synchronized action.

Also, within the 0.10.10 release, the omnibus detection does not work as expected
due to the moving of the omnibus installation directory. This means ChefGems get
installed into the system ruby if it is available and are not available to an
omnibus chef process. Along with this, if a system ruby becomes available later
in the installation, GemPackages will be installed to the embedded ruby first.

### Solutions

For omnibus based chef installations, it becomes vitally important to ensure
a system ruby is available aside from the embedded ruby shipped with Chef. To
provide this, we can use the [ruby_installer](https://github.com/heavywater/chef-ruby_installer)
cookbook to install a package based ruby or the REE version. By placing the
ruby_installer recipe at the start of the run list, we can ensure that a system
ruby is available for Chef during the first run.

To provide backwards compatibility, proper installation locations based on
resources and proper omnibus detection, the [chef_gem](https://github.com/heavywater/chef-chef_gem)
cookbook will add this support. For versions of Chef under 0.10.9, it will provide
a compat chef_gem resource enabling recipes to be migrated to chef_gem without
requiring an immediate upgrade. For Chef versions under 0.10.11, it will patch
omnibus detection to ensure gem installations happen within the correct ruby
instance. And for Chef versions 10.12.0 and beyond, it becomes a no-op cookbook.

### Resources

* [chef_gem](https://github.com/heavywater/chef-chef_gem)
* [ruby_installer](https://github.com/heavywater/chef-ruby_installer)
* [CHEF-3143](http://tickets.opscode.com/browse/CHEF-3143)
