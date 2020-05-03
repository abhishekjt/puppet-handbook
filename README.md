# puppet-handbook
## Hands-on session
____
### Standard puppet directory structure
* Puppet catalogue of modules https://forge.puppet.com/
* Install ntp(Network time protocol) puppet module:`puppet module install puppetlabs-ntp --version 7.4.0` and explore the directory structure under `/etc/puppetlabs/code/environments/production/modules` using `tree` command. 
### PDK (puppet development Kit) Installation
```
$ wget https://apt.puppetlabs.com/puppet5-release-bionic.deb
$ sudo dpkg -i puppet5-release-bionic.deb
$ sudo apt update
$ sudo apt install pdk


#Example usage: pdk new class install
```

### First apache class

* Create new module Apache
```
$ cd /etc/puppetlabs/code/environments/production/modules
$ pdk new module apache
```
* create new manifest named install  
```
$ cd /etc/puppetlabs/code/environments/production/modules/apache
$ pdk new class install
```
* open the manifest and add the contents
```
$ vim manifests/install.pp

class apache::install {
  package { 'httpd':
    ensure => present,
  }
}
```
* Create init.pp
```
 $pdk new class apache
 $ vim manifests/init.pp
 
 class apache {
  include apache::install
}
```
* Add the below content to main manifest file `manifests/site.pp`
```
node abhishekjthackeray2c.mylabserver.com {

  include apache

}
```
* Test changes on `abhishekjthackeray2c.mylabserver.com` with a dry run: `puppet agent -t --noop`

### params.pp

* Create params.pp 
```
$ pdk new class params

class apache::params {
  $install_ensure = 'present'

  case $::osfamily {
    'RedHat': {
      $install_name = 'httpd'
    }
    'Debian': {
      $install_name = 'apache2'
    }
  }
}
```
* update install.pp 
```
class apache::install (
  $install_name   = $apache::params::install_name,
  $install_ensure = $apache::params::install_ensure,
) inherits apache::params {
  package { "${install_name}":
    ensure => $install_ensure,
  }
}
```
* update site.pp
```
node abhishekjthackeray3c.mylabserver.com {
  include apache
}
```
* Converge both the managed nodes

### Hiera 

* Edit hiera.yaml
```
$ cd /etc/puppetlabs/code/environments/production/modules/apache
$  hiera.yaml

---
version: 5

defaults:  # Used for any hierarchy level that omits these keys.
  datadir: data         # This path is relative to hiera.yaml's directory.
  data_hash: yaml_data  # Use the built-in YAML backend.

hierarchy:
  - name: 'Operating System Family'
    path: '%{facts.os.family}-family.yaml'

  - name: 'common'
    path: 'common.yaml'
```
* Edit OS specific yaml files 
```
$vim data/RedHat-family.yaml

---
apache::install_name: 'httpd'
```
```
$ vim data/Debian-family.yaml

---
apache::install_name: 'apache2'
```
* Edit common.yaml
```
$ vim data/common.yaml

---
apache::install_ensure: present
apache::install_name: 'apache2'
```
* Update init.pp
```
class apache  (
  String $install_name,
  String $install_ensure,
) {
  include apache::install
}
```
* update install.pp
```
class apache::install {
  package { "${apache::install_name}":
    ensure => $apache::install_ensure,
  }
}
```

### Files

* Download the required conf files
```
$ cd /etc/puppetlabs/code/environments/production/modules/apache

$ curl https://raw.githubusercontent.com/linuxacademy/content-ppt206-extra/master/apache2.conf -o files/Debian.conf
$ curl https://raw.githubusercontent.com/linuxacademy/content-ppt206-extra/master/httpd.conf -o files/RedHat.conf
```
* Create new config class
```
$ pdk new class config
$ vim manifests/config.pp

class apache::config {
  file { 'apache_config':
    ensure => $apache::config_ensure,
    path   => $apache::config_path,
    source => "puppet:///modules/apache/${osfamily}.conf",
    mode   => '0644',
    owner  => 'root',
    group  => 'root',
  }
}
```
* Edit OS family files and common.yaml accordingly 
```
$vim data/RedHat-family.yaml

apache::config_path: '/etc/httpd/conf/httpd.conf'


$ vim data/Debian-family.yaml:

apache::config_path: '/etc/apache2/apache2.conf'


$ vim data/common.yaml

apache::config_ensure: 'file'
apache::config_path: '/etc/apache2/apache2.conf'
```
* Update init.pp
```
class apache  (
  String $install_name,
  String $install_ensure,
  String $config_ensure,
  String $config_path,
) {
  contain apache::install
  contain apache::config

  Class['::apache::install']
  -> Class['::apache::config']
}
```

### Metaparameters

* Add a new Service class 
```
# @summary
#   Allows for the Apache service to restart when triggered
class apache::service {
  service { "${apache::service_name}":
    ensure     => $apache::service_ensure,
    enable     => $apache::service_enable,
    hasrestart => true,
  }
}
```
* Edit the Hiera data accordingly
```
$ vim data/RedHat-family.yaml:
apache::service_name: 'httpd'

$ vim data/Debian-family.yaml:
apache::service_name: 'apache2'

$ vim data/common.yaml
apache::service_name: 'apache2'
apache::service_ensure: 'running'
apache::service_enable: true
```
* Modify init file 
```class apache  (
  String $install_name,
  String $install_ensure,
  String $config_ensure,
  String $config_path,
  String $service_name,
  Enum["running", "stopped"] $service_ensure,
  Boolean $service_enable,
) {
  contain apache::install
  contain apache::config
  contain apache::service

  Class['::apache::install']
  -> Class['::apache::config']
  ~> Class['::apache::service']
}
```
* Test converge on any of the boxes running puppet agent 

### Templating and Defined Types

* Desired template for setting up virtual hosts in Apache
```
<VirtualHost *:80>
    ServerName subdomain.mylabserver.com
    ServerAlias subdomain
    ServerAdmin admin@mylabserver.com
    DocumentRoot /var/www/subdomain/html
</VirtualHost>
```
* Create a template called `vhosts.conf.epp` 
```
vim templates/vhosts.conf.epp

<VirtualHost *:<%= $port %>>
    ServerName <%= $subdomain %>.<%= $facts[fqdn] %>
    ServerAlias <%= $subdomain %>
    <% if $admin =~ String[1] { -%>
      ServerAdmin <%= $admin %>
    <% } -%>
    DocumentRoot <%= $docroot %>
</VirtualHost>
```
* Create defined type ```pdk new defined_type vhosts``` with the below contents
```
define apache::vhosts (
  Integer $port,
  String $subdomain,
  String $admin,
  String[1] $docroot,
) {
  file { "${docroot}":
    ensure => 'directory',
    owner  => $apache::vhosts_owner,
    group  => $apache::vhosts_group,
  }
  file { "${apache::vhosts_dir}/${subdomain}.conf":
    ensure  => 'file',
    owner   => $apache::vhosts_owner,
    group   => $apache::vhosts_group,
    mode    => '0644',
    content => epp('apache/vhosts.conf.epp', {'port' => $port, 'subdomain' => $subdomain, 'admin' => $admin, 'docroot' => $docroot}),
    notify  => Service["${apache::service_name}"],
  }
}
```
* Add Hiera data, all of which is OS family-specific
```
# data/RedHat-family.yaml
apache::vhosts_dir: '/etc/httpd/conf.d'
apache::vhosts_owner: 'apache'
apache::vhosts_group: 'apache'

# data/Debian-family.yaml
apache::vhosts_dir: '/etc/apache2/sites-available'
apache::vhosts_owner: 'www-data'
apache::vhosts_group: 'www-data'

# data/common.yaml
apache::vhosts_dir: '/etc/apache2/sites-available'
apache::vhosts_owner: 'www-data'
apache::vhosts_group: 'www-data'
```
* Make changed in init.pp
```
class apache  (
  String $install_name,
  String $install_ensure,
  String $config_ensure,
  String $config_path,
  String $service_name,
  Enum["running", "stopped"] $service_ensure,
  Boolean $service_enable,
  String[1] $vhosts_dir,
  String[1] $vhosts_owner,
  String[1] $vhosts_group,
) {
  contain apache::install
  contain apache::config
  contain apache::service

  Class['::apache::install']
  -> Class['::apache::config']
  ~> Class['::apache::service']
}

```
* Make changed to the main manifest : site.pp
```
node abhishekjthackeray3c.mylabserver.com {

  include apache

  apache::vhosts { 'puppet_project':
    port      => 80,
    subdomain => 'puppetproject',
    admin     => 'admin@mylabserver.com',
    docroot   => '/var/www/html/puppetproject',
  }



  apache::vhosts { 'puppet_project_dev':
    port      => 8081,
    subdomain => 'puppetproject-dev',
    admin     => '',
    docroot   => '/var/www/html/puppetproject-dev',
  }
}

```
* Test it out on Ubuntu managed node

### Profiles and Roles

* install the below puppet modules
```
cd /etc/puppetlabs/code/environments/production/modules/

puppet module install puppetlabs-motd
puppet module install puppet-php
puppet module install puppetlabs-mysql

```
* Create a new profile module and base class 
```
pdk new module profile
pdk new class base
vim manifests/base.pp

class profile::base {
  include ::ntp

  class { '::motd':
    content => "This host is managed by Puppet!\n",
  }

}
```
* Add apache profile 
```
pdk new class apache
vim manifests/apache.pp

class profile::apache {
  include ::apache
  class { '::php':
    pear => true,
  }
}

```
* create mysql-server profile
```
pdk new class mysql::server
vim manifests/mysql/server.pp

class profile::mysql::server {
  class { '::mysql::server':
    root_password           => 'passwordhash',
    remove_default_accounts => true,
  }
}
```
* Create Role (Roles should only contain profiles and nothing else.) to include lamp stack
```
cd /etc/puppetlabs/code/environments/production/modules/
pdk new module role

cd role
pdk new class lamp

vim manifests/lamp.pp
class role::lamp {
  include profile::base
  include profile::apache
  include profile::mysql::server
}
```
* Make changes in the main manifest file `site.pp`
```
node abhishekjthackeray2c.mylabserver.com {
  include role::lamp
}

node abhishekjthackeray3c.mylabserver.com {
  include role::lamp
}
```
NOTE: mysql seems to have some dependency issue in ubuntu , run `sudo apt --fix-broken install` and try to converge again

#### Rspec

* Test exisiting default spec 
```
pdk test unit --tests=spec/classes/apache_spec.rb
```
Edit the apache_spec.rb to include the installed classes and rerun the test 
```
require 'spec_helper'

describe 'apache' do
  on_supported_os.each do |os, os_facts|
    context "on #{os}" do
      let(:facts) { os_facts }

      it { is_expected.to contain_class('apache::install') }
      it { is_expected.to contain_class('apache::config') }
      it { is_expected.to contain_class('apache::service') }
      it { is_expected.to compile }
    end
  end
end
```

#### Resources
* Puppet classes: include vs Require vs Contain: https://www.youtube.com/watch?v=jvDLXykcxiA
