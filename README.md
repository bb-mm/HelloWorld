# Data.gov.uk

This guide provides scripts to install a copy of data.gov.uk's website to your own server. Rebrand it and you have a fully-featured government open data portal.

#### This guide is from the repo of [datagovk/dgu-vagrant-puppet](https://github.com/datagovuk/dgu-vagrant-puppet/tree/togo) on `Github`, and it only provides the minimal steps to set up the website. For a live development and more detailed information, you could watch the origin [repo]([datagovk/dgu-vagrant-puppet](https://github.com/datagovuk/dgu-vagrant-puppet/tree/togo).

## About

The UK Government has contributed Data.gov.uk To Go to Github to kick-start the use and development of common open data portal software, beyond the basic CKAN. UK wants to develop it in partnership with other providers of Open Data portals, through the usual Open Source / Github model of forking, pull requests, issues etc. that everyone is encouraged to contribute to.

![Demo image](https://raw.githubusercontent.com/bb-mm/Storm_Sample/master/demo.PNG)

## Overview

Here is an overview of the install process:

* Machine preparation - a fresh Ubuntu 12.04 (virtual) machine
* CKAN source - download from Github
* Puppet provision of the main software packages (Apache, Postgres, SOLR etc) and set-up linux users
* CKAN database setup
* Drupal install

## 1. Machine preparation & CKAN install

### Fresh (virtual) Machine Preparation

Using a virtual-machine it is perfectly fine alternative to use a non-virtual machine, freshly installed with Ubuntu 12.04. The Puppet scripts assume the name of the machine is 'ckan', so you need to login to it and rename it:

    sudo hostname ckan
    sudo vim /etc/hosts
    # ^ add "127.0.0.1  ckan" to hosts...

Puppet will assume the home user is called 'co', so create it with some particular options:

    sudo adduser co -u 510

Then, remember to add user `co` to be a `sudoer`. 

	sudo adduser co sudo

All further steps are to be carried out from the ssh session under the user 'co' on this target machine.

You need to install some dependencies:

    sudo apt-get install ruby1.9.3 rubygems git
    sudo ln -sf /usr/bin/ruby1.9.3 /etc/alternatives/ruby
    sudo gem install librarian-puppet -v 1.0.3
	sudo gem install puppet

Clone this repo to the machine in /vagrant (to match the vagrant install) and switch to the 'togo' branch:

    sudo mkdir /vagrant
	sudo chmod 777 /vagrant
    cd /vagrant
    git clone git@github.com:datagovuk/dgu-vagrant-puppet
    cd /vagrant/dgu-vagrant-puppet
    git checkout togo
	cd /vagrant
	cp -r dgu-vagrant-puppet/* ./
	rm -rf dgu-vagrant-puppet/
	
Use the script to clone all the CKAN source repos.

	 sudo ln -s /vagrant/src /src
	 cd /src
     ./git_clone_all.sh

Puppet is used to install and configure the main software packages (Apache, Postgres, SOLR etc) and setup linux users.

To provision an existing machine, install the puppet modules:

    sudo /vagrant/puppet/install_puppet_dependancies.sh

and then execute the site manifest now at /etc/puppet/:

    sudo puppet apply /vagrant/puppet/manifests/site.pp

Provisioning will take a while, and you can ignore warnings that are listed in the section of this document titled 'Puppet warnings'.

To automatically activate your CKAN python virtual environment on log-in, it is recommended to add this line to your .bashrc:

    source ~/ckan/bin/activate && cd /src/ckan
	
Or you could use the this [sample .bashrc file](https://github.com/datagovuk/dgu-vagrant-puppet/blob/togo/.bashrc) to replace yours.


## 2. CKAN Database setup

**IMPORTANT** You must activate the CKAN virtual environment when working on the VM. Eg.:

    source ~/ckan/bin/activate
	
You can see that the virtual environment is activated by the presence of the `(ckan)` prefix in the prompt. e.g.:
	
	(ckan)co@ckan:/src/ckan$
	
And make sure you run paster commands from the `/vagrant/src/ckan` directory.

After running puppet, a fresh database is created for you. If you need to create it again then you can do it like this:

    createdb -O dgu ckan --template template_postgis

### Use test data

Sample data is provided to demonstrate CKAN. It comprises 5 sample datasets and is loaded like this:

    sudo -u www-data /home/co/ckan/bin/paster \
	--plugin=ckanext-dgu create-test-data --config=/var/ckan/ckan.ini

And then, you need change the ownership of some files, or you will encounter some `permissions denied` problems:
	
	sudo chown www-data:www-data -R /home/co/

### Give yourself a CKAN user for debug (optional)

For test purposes you can add a CKAN admin user. Remember to reset the password before making the site live.

    sudo -u www-data /home/co/ckan/bin/paster user add admin \
	email=admin@ckan password=pass --config=/var/ckan/ckan.ini
	
    sudo -u www-data /home/co/ckan/bin/paster sysadmin add admin --config=/var/ckan/ckan.ini

### Try CKAN

You can test CKAN on the command-line:
    
    curl http://localhost/data/search

And try a browser to connect to the machine with the URL: `http://localhost/data/search`

The sample data looks like this:

![Demo image](https://raw.githubusercontent.com/bb-mm/Storm_Sample/master/demo.PNG)

You should get CKAN HTML. It's worth checking the logs for errors too:

    less /var/log/ckan/ckan-apache.error.log

Working correctly you should see something like this:


	[Fri Sep 19 13:43:49 2014] [error] 2014-09-19 13:43:49,484 
	DEBUG [ckanext.spatial.model.package_extent] Spatial tables defined in memory
	[Fri Sep 19 13:43:49 2014] [error] 2014-09-19 13:43:49,491 
	DEBUG [ckanext.spatial.model.package_extent] Spatial tables already exist
	[Fri Sep 19 13:43:49 2014] [error] 2014-09-19 13:43:49,502 
	DEBUG [ckanext.harvest.model] Harvest tables defined in memory
	[Fri Sep 19 13:43:49 2014] [error] 2014-09-19 13:43:49,505 
	DEBUG [ckanext.harvest.model] Harvest tables already exist
	[Fri Sep 19 13:43:50 2014] [error] 2014-09-19 13:43:50,025 
	CRITI [ckan.lib.uploader] Please specify a ckan.storage_path in your config
	[Fri Sep 19 13:43:50 2014] [error]                              for your uploads


## 4. Drupal install

For Drupal you will need to complete the configuration of the LAMP stack and get a working drush installation, as explained below.  For more detailed requirements, please refer to https://drupal.org/requirements .

### Install Drush

For more details about installation of Drush, see here: https://github.com/drush-ops/drush

You can either use `Composer` or `php-pear` to install `Drush`:

#### Option1: Composer

First get Composer:

    curl -sS https://getcomposer.org/installer | php
    sudo mv composer.phar /usr/local/bin/composer

Now install the latest Drush:

    composer global require drush/drush:dev-master

And add it to the path:

    sudo sed -i '$a\export PATH="$HOME/.composer/vendor/bin:$PATH"' $HOME/.bashrc
    source $HOME/.bashrc

#### Option2: PHP-Pear

If you don't have pear installed (command not found), run:

	sudo apt-get install php-pear
	
Then, to install Drush:

	sudo pear channel-discover pear.drush.org
	sudo pear install drush/drush
	
### Install the DGU Drupal Distribution

You can install the DGU Drupal Distribution with the following drush command:


	sudo mkdir /var/www/drupal
	sudo chown co:www-data /var/www/drupal
	cd /src/dgu_d7/
	drush make distro.make /var/www/drupal/dgu
	mysql -u root --execute "CREATE DATABASE dgu;"
	mysql -u root --execute "CREATE USER 'co'@'localhost' IDENTIFIED BY 'pass';"
	mysql -u root --execute "GRANT ALL PRIVILEGES ON *.* TO 'co'@'localhost';"
	cd /var/www/drupal/dgu
	drush --yes --verbose site-install dgu --db-url=mysql://co:pass@localhost/dgu \
	--account-name=admin --account-pass=admin  --site-name='something creative'


This will install Drupal, download all the required modules and configure the system.  In the `site-install` command you can ignore two errors at the end about sending e-mails, due to sendmail being missing. E-mail functionality will need to be fixed for a production system.

After this step completes successfully, you should enable some modules:


	drush --yes en dgu_site_feature
	drush --yes en dgu_app dgu_blog dgu_consultation dgu_data_set dgu_data_set_request dgu_footer dgu_forum \
	dgu_glossary dgu_idea dgu_library dgu_linked_data dgu_location dgu_organogram dgu_promo_items dgu_reply \
	dgu_shared_fields dgu_user dgu_taxonomy ckan dgu_search dgu_services dgu_home_page dgu_moderation


You will need to configure drupal with the url of your CKAN instance.  We use the following drush commands:

	drush vset ckan_url 'http://<your hostname>/api/';
	drush vset ckan_apikey 'xxxxxxxxxxxxxxxxxxxxx';

You may also check and modify these settings in the admin menu: configuration->system->ckan.

Now fix permissions:

	sudo chown -R co:www-data /var/www/drupal/dgu/sites/default/files
	
Otherwise you'll get messages such as "The specified file temporary://fileKrLiDX could not be copied, because the destination directory is not properly configured. This may be caused by a problem with file or directory permissions. More information is available in the system log."

Drupal uses a second SOLR core for the search. The configuration of this is to be provided soon.

## 5. Drupal content

### Sample content

Those evaluating this distribution will probably want to use the sample content, which creates some sample blog posts, apps etc. This is installed like this:

    zcat /src/dgu_d7/sample/dgud7_default_db.sql.gz  | mysql -u root dgu

NB This will delete all other Drupal content and users.

You can now log-in (it is http://<your hostname>/user ) with this user:

    Username: admin
    Password: admin

## 6. Final

Before you reach the final step, you need to make a small change to the `redirect.module` at `/var/www/drupal/dgu/profiles/dgu/modules/contrib/redirect/redirect.module`.

Open this file, and find the line of `572`, it should look like this:

	$rid_query->condition('status', 1);
	
`Delete` or `Comment` this line! Or you will get error when you try to access the home page.

After all the steps done successfully, you can use a browser to see a sample website. It should look like this:

![Demo page](https://raw.githubusercontent.com/bb-mm/Storm_Sample/master/home.PNG)

# Orientation

##Sysadmin User Setting (CKAN)

If you want to manage your DATASETS at CKAN (such as create/delete/edit datasets), you need the following configurations to make you a `CKAN Sysadmin User`:

	cd /vagrant/src/ckan
	sudo -u www-data /home/co/ckan/bin/paster --plugin=ckan sysadmin add user_d1 --config=/var/ckan/ckan.ini

Then, try with the `http://<your hostname>/ckan-admin` to log in. (If you use the default settings, the username and password should both be "admin"). 
After Logging in as the Sysadmin user, click the `System Dashboard`, it should look like this:

![dashboard](https://raw.githubusercontent.com/bb-mm/Storm_Sample/600ff7d4f3ec712ba340b4adf8ad3cda32a849f6/dashboard.PNG) 


## CKAN Paster commands

When running CKAN paster commands, you should ensure that:

* you specify the path to paster in the virtualenv (in the future you might just ensure you've activated CKAN's python virtual environment, but that doesn't work when you sudo)
* you are in the CKAN source directory
* use the www-data user, to avoid the log permissions problem (see section below)

You can see that the virtual environment is activated by the presence of the `(ckan)` prefix in the prompt. e.g.:

    (ckan)co@precise64:/src/ckan$

Note you do need to specify --config because although ckan now gets it from the CKAN_INI environment variable (this is due to a recently introduced change to ckan), that is not available when you sudo.

Examples::

    sudo -u www-data /home/co/ckan/bin/paster search-index rebuild --config=/var/ckan/ckan.ini
    sudo -u www-data /home/co/ckan/bin/paster user user_d1 --config=/var/ckan/ckan.ini
    sudo -u www-data /home/co/ckan/bin/paster --plugin=ckanext-dgu create-test-data --config=/var/ckan/ckan.ini

You can add `--help` to list commands and find out more about one. Find full details of the CKAN paster commands is here: http://docs.ckan.org/en/ckan-2.2/paster.html

## CKAN Config file

The ckan config file is `/var/ckan/ckan.ini`. If you change any options, for them to take effect in the web interface you need to restart apache:

    sudo /etc/init.d/apache2 graceful

## CKAN Logs

The main CKAN log file is: `/var/log/ckan/ckan.log`

Errors go to: `/var/log/ckan/ckan-apache.error.log`

The log levels are set in /var/ckan/ckan.ini, so to get the debug logging from ckan you can change the level in the `logger_ckan` section. i.e. change it to:

	[logger_ckan]
	level = DEBUG
	handlers = console, file
	qualname = ckan
	propagate = 0

(and obviously restart apache to take effect)

## Log permissions

It can happened that you may see CKAN return '500 Internal Server Error' and when looking at the log /var/log/ckan/ckan.log you see this error:

    IOError: [Errno 13] Permission denied: '/var/log/ckan/ckan.log

This can happen when running paster commands and forgetting run them as the `www-data` user as directed. Normally the CKAN logfile is created and written to by apache and hence is owned by user `www-data`. However when running paster commands as the co user it will also write to the log, and if the log happens to roll-over at this time then the co user will now own the logfile. To rectify this, change the ownership:

    sudo chown www-data:www-data /var/log/ckan/ckan.log

The fix for this issue is in the pipeline.

## Grunt and assets

Data.gov.uk uses Grunt to do pre-processing of Javascript and CSS scripts as well as images and it writes timestamps to help with cache versioning.

Puppet will have installed a recent version of NodeJS (0.10.32+) and npm (1.4.28+) plus Grunt. There are two repos with assets which if you change you need to run Grunt before they will be used by CKAN.

Grunt runs on puppet provision, and you can manually run it like this:

    cd /vagrant/src/ckanext-dgu
    grunt
    cd /vagrant/src/shared_dguk_assets
    grunt

There is more about Grunt use here: https://github.com/datagovuk/shared_dguk_assets/blob/master/README.md
