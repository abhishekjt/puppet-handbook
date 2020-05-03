# puppet-handbook (WIP)
### Hands-on session
#### Standard puppet directory structure
* Puppet catalogue of modules https://forge.puppet.com/
* Install ntp(Network time protocol) puppet module:`puppet module install puppetlabs-ntp --version 7.4.0` and explore the directory structure under `/etc/puppetlabs/code/environments/production/modules` using `tree` command. 
#### PDK (puppet development Kit) Installation
```
$ wget https://apt.puppetlabs.com/puppet5-release-bionic.deb
$ sudo dpkg -i puppet5-release-bionic.deb
$ sudo apt update
$ sudo apt install pdk


#Example usage: pdk new class install
```

#### First apache class
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
#### params.pp
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
#### Hiera 
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
#### Files
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
#### Metaparameters
#### Templating
#### Defined Types
#### Profiles and Roles
#### Rspec 

#### Resources
* Puppet classes: include vs Require vs Contain: https://www.youtube.com/watch?v=jvDLXykcxiA
