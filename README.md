# Installation guide for GitLab 7.9-stable on OS X 10.10



## Requirements
- ruby 2.1.5
- vim
- Mac OS X 10.10
- User group `git` and user `git` in this group
- Enable remote login for `git` user

Run the following commands in order to create the group and user `git`:

```bash
LastUserID=$(dscl . -list /Users UniqueID | awk '{print $2}' | sort -n | tail -1)
NextUserID=$((LastUserID + 1))
sudo dscl . create /Users/git
sudo dscl . create /Users/git RealName "Git Lab"
sudo dscl . create /Users/git hint "Password Hint"
sudo dscl . create /Users/git UniqueID $NextUserID
LastGroupID=$(dscl . readall /Groups | grep PrimaryGroupID | awk '{ print $2 }' | sort -n | tail -1)
NextGroupID=$(($LastGroupID + 1 ))
sudo dscl . create /Groups/git
sudo dscl . create /Groups/git RealName "Git Lab"
sudo dscl . create /Groups/git passwd "*"
sudo dscl . create /Groups/git gid $NextGroupID
sudo dscl . create /Users/git PrimaryGroupID $NextGroupID
sudo dscl . create /Users/git UserShell $(which bash)
sudo dscl . create /Users/git NFSHomeDirectory /Users/git
sudo cp -R /System/Library/User\ Template/English.lproj /Users/git
sudo chown -R git:git /Users/git
```

Warning!!! git user should be admin


Hide the git user from the login screen:

	 sudo defaults write /Library/Preferences/com.apple.loginwindow HiddenUsersList -array-add git

Unhide:

	sudo defaults delete /Library/Preferences/com.apple.loginwindow HiddenUsersList

## Recommendation

Use Parallels Deskop for first install.

## Note

If you find any issues, please let me know or send PR with fix ;-) Thank you!

## Install instructions

1. [Install command line tools](#1-install-command-line-tools)
2. [Install Homebrew](#2-install-homebrew)
3. [Install some prerequisites](#3-install-some-prerequisites)
4. [Install database](#4-install-database)
5. [Setup database](#5-setup-database)
6. [Install ruby](#6-install-ruby)
7. [Install Gitlab Shell](#7-install-gitlab-shell)
8. [Install GitLab](#8-install-gitlab)
9. [Check Installation](#9-check-installation)
10. [Configuring SMTP](#12-configuring-smtp)

### 1. Install command line tools

	xcode-select --install #xcode command line tools

### 2. Install Homebrew

	ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
	brew doctor

### 3. Install some prerequisites

	brew install icu4c git logrotate redis libxml2 cmake pkg-config

	ln -sfv /usr/local/opt/logrotate/*.plist ~/Library/LaunchAgents
	launchctl load ~/Library/LaunchAgents/homebrew.mxcl.logrotate.plist
	ln -sfv /usr/local/opt/redis/*.plist ~/Library/LaunchAgents
	launchctl load ~/Library/LaunchAgents/homebrew.mxcl.redis.plist

Make sure you have python 2.5+ (gitlab don’t support python 3.x)

Confirm python 2.5+

	python --version

GitLab looks for python2

	sudo ln -s /usr/bin/python /usr/bin/python2

Some more dependices

	sudo easy_install pip
	sudo pip install pygments

Install `docutils` from http://sourceforge.net/projects/docutils/files/latest/download?source=files

	curl -O http://heanet.dl.sourceforge.net/project/docutils/docutils/0.11/docutils-0.11.tar.gz
	gunzip -c docutils-0.11.tar.gz | tar xopf -
	cd docutils-0.11
	sudo python setup.py install

### 4. Install database

> Official installation documentation recommend to use postgresql, see [http://doc.gitlab.com/ce/install/installation.html](http://doc.gitlab.com/ce/install/installation.html). But I prefer MySQL.

#### mysql

	brew install mysql
	ln -sfv /usr/local/opt/mysql/*.plist ~/Library/LaunchAgents
	launchctl load ~/Library/LaunchAgents/homebrew.mxcl.mysql.plist


### 5. Setup database

#### mysql

Run `mysql_secure_installation` and set a root password, disallow remote root login, remove the test database, and reload the privelege tables.

	mysql_secure_installation

Now login in mysql

	mysql -u root -pPASSWORD_HERE

Create a new user for our gitlab setup 'git'

	CREATE USER 'git'@'localhost' IDENTIFIED BY 'PASSWORD_HERE';

Create database

	CREATE DATABASE IF NOT EXISTS `gitlabhq_production` DEFAULT CHARACTER SET `utf8` COLLATE `utf8_unicode_ci`;

Grant the GitLab user necessary permissions on the table.

	GRANT SELECT, LOCK TABLES, INSERT, UPDATE, DELETE, CREATE, DROP, INDEX, ALTER ON `gitlabhq_production`.* TO 'git'@'localhost';

Quit the database session

	\q

Try connecting to the new database with the new user

	sudo -u git -H mysql -u git -pPASSWORD_HERE -D gitlabhq_production


### 6. Install ruby

OS X 10.10 has ruby 2.0.

rudy 2.1.5 can use rbenv or rvm .The installation please google.


### 7. Install Gitlab Shell

	cd /Users/git
	sudo -u git git clone https://github.com/gitlabhq/gitlab-shell.git
	cd gitlab-shell
	sudo -u git git checkout v2.6.0
	sudo -u git cp config.yml.example config.yml

Now open `config.yml` file and edit it


Use `/Users` instead of `/home`, and change redis-cli path to homebrew’s redis-cli

	sudo -u git sed -i "" "s/\/home\//\/Users\//g" config.yml
	sudo -u git sed -i "" "s/\/usr\/bin\/redis-cli/\/usr\/local\/bin\/redis-cli/" config.yml

Do setup

	sudo -u git -H ./bin/install

### 8. Install Gitlab

#### Download Gitlab

	cd /Users/git
	sudo -u git git clone https://github.com/gitlabhq/gitlabhq.git gitlab
	cd gitlab
	sudo -u git git checkout 7-9-stable

#### Configuring GitLab

	sudo -u git cp config/gitlab.yml.example config/gitlab.yml
	sudo -u git sed -i "" "s/\/usr\/bin\/git/\/usr\/local\/bin\/git/g" config/gitlab.yml
	sudo -u git sed -i "" "s/\/home/\/Users/g" config/gitlab.yml

Make sure GitLab can write to the `log/` and `tmp/` directories

	sudo chown -R git log/
	sudo chown -R git tmp/
	sudo chmod -R u+rwX  log/
	sudo chmod -R u+rwX  tmp/

Create directories for repositories make sure GitLab can write to them

	sudo -u git mkdir /Users/git/repositories
	sudo chmod -R u+rwX  /Users/git/repositories/

Create directory for satellites

	sudo -u git mkdir /Users/git/gitlab-satellites

Create directories for sockets/pids and make sure GitLab can write to them

	sudo -u git mkdir tmp/pids/
	sudo -u git mkdir tmp/sockets/

	sudo chmod -R u+rwX  tmp/pids/
	sudo chmod -R u+rwX  tmp/sockets/

Create public/uploads directory otherwise backup will fail

	sudo -u git mkdir public/uploads
	sudo chmod -R u+rwX  public/uploads

Fix

	sudo chown -R git:git /Users/git/repositories/
	sudo chmod -R ug+rwX,o-rwx /Users/git/repositories/
	sudo chmod -R ug-s /Users/git/repositories/
	sudo find /Users/git/repositories/ -type d -print0 | sudo xargs -0 chmod g+s

Copy the example Unicorn config

	sudo -u git cp config/unicorn.rb.example config/unicorn.rb
	sudo -u git sed -i "" "s/\/home/\/Users/g" config/unicorn.rb

Comment out `listen "/Users/git/gitlab/tmp/sockets/gitlab.socket", :backlog => 64` in `unicorn.rb`.

Configure Git global settings for git user, useful when editing via web

	sudo -u git -H git config --global user.name "GitLab"
	sudo -u git -H git config --global user.email "gitlab@domain.com"

Copy rack attack middleware config

	sudo -u git -H cp config/initializers/rack_attack.rb.example config/initializers/rack_attack.rb

Set up logrotate

	sudo mkdir /etc/logrotate.d/
	sudo cp lib/support/logrotate/gitlab /etc/logrotate.d/gitlab
	sudo sed -i "" "s/\/home/\/Users/g" /etc/logrotate.d/gitlab

#### Gitlab Database Config

##### mysql

	sudo -u git cp config/database.yml.mysql config/database.yml
	sudo -u git sed -i "" "s/secure password/PASSWORD_HERE/g" config/database.yml


#### Install Gems
RubyGems taobao mirror

	$ gem sources --remove https://rubygems.org/
	$ gem sources -a https://ruby.taobao.org/
	$ gem sources -l
	*** CURRENT SOURCES ***

	https://ruby.taobao.org

change first line in Gemfile

	source 'https://ruby.taobao.org/'


In case if you are using mysql as database:

	sudo gem install bundler
	sudo -u git -H bundle install --deployment --without development test postgres aws

If you can't build nokogiri 1.6.5 do this:

	brew install libxml2 libxslt libiconv
	brew link libxml2 libxslt

	sudo gem install nokogiri -- --use-system-libraries --with-xml2-include=/usr/include/libxml2 --with-xml2-lib=/usr/lib/

	bundle config build.nokogiri --use-system-libraries --with-xml2-include=/usr/include/libxml2 --with-xml2-lib=/usr/lib/

Then

	sudo -u git -H bundle install --deployment --without development test postgres aws





#### Initialize Database and Activate Advanced Features

	sudo -u git -H bundle exec rake gitlab:setup RAILS_ENV=production

Here is your admin login credentials:

	login: root
	password: 5iveL!fe

#### Precompile assets

```
sudo -u git -H bundle exec rake assets:precompile RAILS_ENV=production
```

#### Setup Redis Socket

Redis config is located in `/usr/local/etc/redis.conf`. Make o copy:

```
cp /usr/local/etc/redis.conf /usr/local/etc/redis.conf.orig
```

Edit file and set `port 0` (instead of 6379) and uncomment:

```
unixsocket /tmp/redis.sock
unixsocketperm 777
```

*Warning: permission 777 could be insecure? We need to find better solution. `redis.sock` has `wheel` group. Is there any way how to permanently change it?*

Restart redis (see how at `brew info redis`):

```
launchctl unload ~/Library/LaunchAgents/homebrew.mxcl.redis.plist
launchctl load ~/Library/LaunchAgents/homebrew.mxcl.redis.plist
```

Configure Redis connection settings:

```
sudo -u git -H cp config/resque.yml.example config/resque.yml
```

Change the Redis socket path to `unix:/tmp/redis.sock`:

```
sudo -u git -H vim config/resque.yml
```
production: unix:/tmp/redis.sock

Configure gitlab-shell to use Redis sockets (`/tmp/redis.sock`):

```
sudo -u git vim /Users/git/gitlab-shell/config.yml
```
socket: /tmp/redis.sock

Start background_jobs

	sudo -u git -H RAILS_ENV=production bin/background_jobs start


Next step will setup services which will keep Gitlab up and running

	sudo -u git -H rails s -e production -p 8080

### 9. Check Installation

Check gitlab-shell

	sudo -u git /Users/git/gitlab-shell/bin/check


Double-check environment configuration

	sudo -u git -H bundle exec rake gitlab:env:info RAILS_ENV=production

Do a thorough check. Make sure everything is green.

	sudo -u git -H bundle exec rake gitlab:check RAILS_ENV=production

The script complained about the init script not being up-to-date, but I assume that’s because it was modified to use /Users instead of /home. You can safely ignore that warning.


### 10. Configuring SMTP

Copy config file

	sudo -u git -H cp config/initializers/smtp_settings.rb.sample config/initializers/smtp_settings.rb

Edit `config/initializers/smtp_settings.rb` with your settings (see [ActionMailer::Base - Configuration options](http://api.rubyonrails.org/classes/ActionMailer/Base.html))
