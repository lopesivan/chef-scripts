Convert Your Bash Scripts With chef-apply



[213c] By Chris Doherty January 16, 2015November 1, 2016

A Brief Note

Before we get started, it's worth mentioning that even though I'm going to pretend
Chef is a scripting language, a Chef runlist describes the desired final state of a
system, rather than providing a list of actions to execute. In other words, each Chef
resource triggers code like this:

if system state already matches what the resource describes
then
don't do anything
else
perform whatever actions are needed to get the system into the correct state

So if I tell Chef I want a file present, and specify its contents:

file "/tmp/foo" do
 content "This will be the file's content."
 end

Chef will check to see if the file is there with that content, and only update it (or
create it) if needed.

For the times when life doesn't fit that model, we have the execute resource to run
some shell code. Even then, we can add a not\_if or only\_if condition to prevent it
from executing on every Chef run.

You're still talking. You told me this would be useful.

Enough! It's Friday afternoon, I want to try out Chef, and I want to get something
done!

At home I run a music server called squeezeboxserver, and because later versions
removed my favorite features, I use an older version that won't run on anything later
than Ubuntu 11. I have an Ubuntu 10 virtual machine, using Vagrant to manage the
VirtualBox VM. squeezeboxserver scans my music library and stores the file and
playlist data in MySQL.

squeezeboxserver is very particular about its dependencies and configuration. I used
its build scripts to build a Debian package, but it doesn't install any dependencies.
It needs MySQL Server installed, but by default it runs its own mysqld rather than
using a system-global mysqld. I originally wrote a Bash script to set the machine up:

rm /etc/localtime
 ln -s /usr/share/zoneinfo/America/Los_Angeles /etc/localtime
 apt-get update
 export DEBIAN_FRONTEND=noninteractive
 apt-get -y install mysql-server-5.1 libmysqlclient16-dev mysql-client-5.1
 dpkg -i /home/chris/squeezebox/squeezeboxserver_7.5.6~32834_all.deb
 tar -C / -xf /home/chris/backups/sbserver_prefs.tar
 /etc/init.d/squeezeboxserver restart

This mostly worked, but it's surprisingly brittle for an 8-line shell script. The VM
still required a lot of manual tweaks, and it was actually fragile enough that I
avoided taking the host server down for maintenance because I dreaded trying to bring
the VM back up. It also wasn't reliable if any of those commands encountered an
error&#8211;this happened far more often than I would have expected&#8211;and it didn't clearly
communicate what it was doing. Deleting and re-creating the VM was similarly
hazardous.

With the most recent maintenance of my host server, I decided I'd had enough, and I
converted it using chef-apply. chef-apply is a tool that ships with Chef, and it
basically lets you pretend Chef is a scripting language: no recipes, no directory
structure, just a single file of Chef resources. You can even shebang a chef-apply
script as you would a shell script:

#!/usr/bin/env chef-apply

I went through my original Bash script line by line to convert it to Chef.

rm /etc/localtime
 ln -s /usr/share/zoneinfo/America/Los_Angeles /etc/localtime

There are a few ways to set your machine's timezone; this just happens to be the one
I'm familiar with. Easy enough, Chef has a link resource.

link "/etc/localtime" do
 to "/usr/share/zoneinfo/America/Los_Angeles"
 end

Here I'm telling Chef &#8220;the correct state of the server has this symlink&#8221;; Chef will
check to see if the symlink already exists, and do nothing if it's already there and
pointing to the right file.

export DEBIAN_FRONTEND=noninteractive
 apt-get -y install mysql-server-5.1 libmysqlclient16-dev mysql-client-5.1

Chef's package resource has us covered.

package mysql-server-5.1
 package libmysqlclient16-dev
 package mysql-client-5.1

Or, if you're comfortable with Ruby and you don't like the repetition, you can do it
with a loop which does the exact same thing.

["mysql-server-5.1", "libmysqlclient16-dev", "mysql-client-5.1"].each do |pkg_name|
 package pkg_name
 end

But wait&#8211;installing mysql-server-5.1 starts the system mysqld. I could stop it
(service mysql stop) but then it will restart whenever I reboot the VM. I'm a software
engineer, not a sysadmin, so I'm far too lazy to figure out how to permanently disable
the system mysqld. Chef's service resource has an action for this.

service "mysql" do
 action :disable
 end

Now mysqld won't start up on system boot.

So much for the prep work. Now to install and configure squeezeboxserver itself.

dpkg -i /home/chris/squeezebox/squeezeboxserver_7.5.6~32834_all.deb

Here we encounter our first diversion, which merits some background. Chef resources
are backed by providers, where the actual heavy lifting of a Chef resource happens.
When we wrote package mysql-client-5.1 above, we didn't have to concern ourselves with
what operating system we were on: Chef will figure it out and call the correct
provider for your system. On a RedHat-based system, this is the yum provider, and on
Debian-based systems, of course, the package resource resolves to the apt provider. As
a result, to install from a .deb file, I had to use the dpkg\_package resource, which
explicitly uses the dpkg provider.

dpkg_package "squeezeboxserver_7.5.6" do
 source "/home/chris/squeezebox/squeezeboxserver_7.5.6~32834_all.deb"
 end

The .deb file leaves the server running, but I've been running squeezeboxserver for a
long time, and I have a .tar of prefs files I want to use.

tar -C / -xf /home/chris/backups/sbserver_prefs.tar

This is not so bad, and in fact there is no core tar resource, so I just move it
verbatim into Chef with the execute resource. (There's a tar cookbook, but it's Friday
afternoon and I don't want anything complicated.) Normally we use Chef resources to
describe our desired final state; sometimes that's not enough, so we have execute,
which essentially shells out a command.

execute "tar -C / -xf /home/chris/backups/squeezeboxserver.tar"

Then,
Now restart the server, and tell it to scan my music collection to populate the MySQL
database.

/etc/init.d/squeezeboxserver restart

Easy enough. Tell Chef to disable, then re-enable the service.

service "squeezeboxserver" do
 action :restart
 end

And&#8230;that's it.

Since I was on a roll, I decided to add some cronjobs, since Chef makes managing
cronjobs trivial. It's such trouble to modify crontabs safely in Bash that I had never
bothered trying.

cron "backup_config" do
 minute "0"
 hour "0"
 user "vagrant"
 command "tar -C / -cf /home/chris/backups/sbserver_prefs.tar /var/lib/squeezeboxserver/prefs"
 end

A couple more like that, and now I have up-to-date backups of the config, the
playlists, and the MySQL data. Boom. I deleted and re-created the VM a few times&#8211;once
or twice, it was just for fun. Chef re-built it identically every time. If something
goes wrong, wonder of wonders, it stops and tells me. It's true, the chef-apply script
has a few more lines in it. It's also more reliable and has more features than the
original.

The deep truth about Chef is that it's not doing anything clever, which is a real
relief to those of us who think &#8220;clever&#8221; means &#8220;hard to debug.&#8221; It's simply doing all
the checking (&#8220;is this file present and with the right contents?&#8221;) and cross-platform
implementation (&#8220;use apt-get on Debian, Yum on RHEL, pkg_add/pkgng on FreeBSD&#8230;&#8221;) that
we would find tedious and error-prone. It does all this always and only in the order
we tell it to.

chef-apply is sort of &#8220;Chef Lite&#8221;: Chef's quick-hack, Get Stuff Done tool. You
definitely don't want to run your whole infrastructure on it. It runs just a single
file, with no cookbooks, file templates, roles, users, orgs, or any of the other Chef
features you'd want in your automation. When you're ready for the full power of Chef,
you have a foundation of cross-platform code you can copy into your recipes.

I've included the full chef-apply script below. Happy hacking!

link "/etc/localtime" do
 to "/usr/share/zoneinfo/America/Los_Angeles"
 end
 package mysql-server-5.1
 package libmysqlclient16-dev
 package mysql-client-5.1
 service "mysql" do
 action :disable
 end
 dpkg_package "squeezeboxserver_7.5.6" do
 source "/home/chris/squeezebox/#{DEB}"
 end
 # o/~ meet the new server / same as the old server o/~
 execute "tar -C / -xf /home/chris/backups/sbserver_prefs.tar"
 service "squeezeboxserver" do
 action :restart
 end
 # ---- end app setup ----
 cron "nightly_rescan" do
 minute "30"
 hour "0"
 user "vagrant"
 command "/usr/sbin/squeezeboxserver-scanner --rescan"
 end
 cron "backup_config" do
 minute "0"
 hour "0"
 user "vagrant"
 command "tar -C / -cf /home/chris/backups/sbserver_prefs.tar /var/lib/squeezeboxserver/prefs"
 end
 cron "backup_mysql" do
 minute "5"
 hour "0"
 user "vagrant"
 command "sudo mysqldump -S #{MYSQL_SOCK} slimserver | gzip -c > /home/chris/backups/sbserver_sql.tgz"
 end
 cron "backup_playlists" do
 minute "10"
 hour "0"
 user "vagrant"
 command "tar -cf /home/chris/backups/playlists.tar /home/chris/music/playlists"
 end

Posted in

community engineering






[213c35badc2a]
Author Chris Doherty

Chris is an itinerant Senior Software Engineer at Chef, sharing the joys of good
engineering practice and the importance of the beginning Chef user's experience.

  * Max Voloshin

    Awesome blog post! After reading it I have built Vagrant VM with chef-apply and I
    understand every line of my recipe :)

ChefConf
Get the

Internet Us



Categories

  * ChefConf
  * Products &#038; Projects
  * Community
  * DevOps
  * News
  * Webinars
  * Partners
  * Cookbooks
  * Learn Chef

Follow @chef

Tweets by @chef

c 2016 Chef Software, Inc. All Right Reserved.




