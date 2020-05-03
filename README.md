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
* 
#### Hiera 
#### Files
#### Metaparameters
#### Templating
#### Defined Types
#### Profiles and Roles
#### Rspec 

#### Resources
* Puppet classes: include vs Require vs Contain: https://www.youtube.com/watch?v=jvDLXykcxiA
