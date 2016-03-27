+++
date = "2015-03-25T22:41:00+01:00"
title = "Typo3 Neos: Automated deployment (without Surf)"
aliases = [
  "typo3-neos-automated-deployment/"
]

+++

For the new version of my own website I've planned to use Typo3 Neos. Since Neos is relatively new it includes a more versatile deployment workflow compared to traditional PHP applications which are often only installable / deployable via a clickable installer. Traditional installers not only create pain when trying to setup consistent development environments with Vagrant they are also more error prone due to the nature of frontends which are likely change the markup over time.

# Why Neos?
Neos solves this problem as it is based on the Flow3 Framework which brings a command line utility for executing a ton of tasks like creating backups or migrating the database. There is also a smart tool called "Surf" which is basically a set of PHP scripts suitable for deploying. In this article I will talk about automating the deployment of Neos with the command line tools while specifically avoiding Surf. This is because I am using a shared hosting plan. More on that in a second.

# A Typo3 system on a shared hosting environment. Are you serious?
Short answer: Yes.
Longer answer: Even if Typo3 Neos may not run in an optimal way on a shared hosting environment where script execution limits (more on that later) and only limited system access is provided it was the most obvious choice since I've been using such an environment for years now and most of the times I've found an adequate solution for hosting my applications there. It is also rather stable and proofed for running several websites of my own. For instance one website is currently running a Typo3 CMS 6.2 system.

# What I am using
The hosting plan I use is called "Managed Hosting Pro" by Domainfactory featuring a script execution limit of 120 seconds and limited shell access. I luckily have the ability to edit the PHP.ini file for specific domains but I cannot avoid my scripts being killed after 120 seconds. This is due to a background daemon which is watching for long running tasks. For most of the more simpler websites such managed hosting plans are fine. But this plan steps a bit forward as it additionally provides limited ssh access and even cronjobs.
So to come back to a point I mentioned before: Why I intentionally avoiding Surf is because the deployment with that tool is just one big call of that script. It is very unlikely that the deployment is finished within under 2 minutes. (compiling sass, installing composer and migrating the database all takes time) So if I would split up the tasks I could safely execute them one after another. This is what I've done and this is how Jenkins helped me. For this deployment process I am using a Jenkins [plugin](https://wiki.jenkins-ci.org/display/JENKINS/Publish+Over+SSH+Plugin) to run commands over SSH and also to upload files via SCP.

# More on script execution limits
For installing Neos via composer I had to increase the timeout of some the actual steps in the jobs which are using that SSH plugin since it could be that it takes more than 120 seconds. Wait a minute! You said that you have this script execution limit of 120 seconds. How could that make sense?
Theoretically it could happen that you could have nearly twice the amount of time available for executing one command. This is due to the background daemon which only checks every 120 seconds so that way it could be that it checks after 119 seconds and then again 120 seconds later. So in the best case you could have 239 seconds available. But still that's not enough for doing a complete deployment.

# The steps
In the following steps I explain what I did to deploy an instance from scratch while first creating backups.
I assume the following variables are set. Since I am using Jenkins you will find everything in that syntax:

- **${backup_dir}**: location where your site backups should be stored
- **${neos_dir}**: neos root directory
- **${package_key}**: name of your site you are going to use, for example "TL.DbzmanOnline"
- **${site_node}**: name of the root node for your site, in my case just "dbzmanonline"
- **${neos_user}**: name of the admin user
- **${neos_password}**: password for the admin user
- **${full_name}**: full name of the admin user
- **${domain}**: domain for your site

And some database settings:

- **${db_user}**
- **${db_password}**
- **${db_host}**
- **${db_name}**

The coherent commands in each step are all wrapped in a seperate "SSH transfer set" in Jenkins to decrease the chance of timeouts. That's also the reason why you see that "export FLOW_CONTEXT.." more than once in a step. Not that also this setup is just for deploying **one instance** with **one site** (one whole website tree). If you need to deploy more than one site you have to repeat the steps in #4 for each one of you sites.
But lets start deploying!

## 1. Create backups
```
# As it could be that the resource folder of the backup location is not empty we remove it
# Important: Make sure that the following "Resources" folder exists
# so that images can be backed up.
rm -rf ${backup_dir}/${package_key}/Resources/*
# set the context to production
export FLOW_CONTEXT=Production
cd ${neos_dir}
./flow site:export --site-node ${site_node} --filename ${backup_dir}/${package_key}/Sites.xml`
```

## 2. Cleanup the installation
```
# Recreate database with whatever tool you prefer. Could be like that:
mysql -u ${db_user} -p${db_password} -h ${db_host} -e "drop database ${db_name};"
mysql -u ${db_user} -p${db_password} -h ${db_host} -e "create database ${db_name};"
# Purge the current neos installation
rm -rf ${neos_dir}
```

## 3. Configure your installation
```
# This is the most critical part in terms of duration.
# Here I am using the latest stable CLI tool of PHP (specific to Domainfactory)
# The "require-dev" part is skipped and dist packages are prefered
# to speed up composer installation.
# Sometimes it could be, depending on the server load or internet connection,
# that this part fails.
export FLOW_CONTEXT=Production
/usr/local/bin/php5-56STABLE-CLI composer.phar create-project --no-dev --prefer-dist typo3/neos-base-distribution ${neos_dir}

# Since swiftmailer is used for the contact form we need
# to include it in the composer.json and install it
# Note: If you have issues with swiftmailer
# while submitting the form ("...Call to a member function send() on a non-object")
# have a look at: https://jira.typo3.org/browse/NEOS-270
cd ${neos_dir}
# This could be solved more elegant but is enough for now
sed -i 's/\"require\": {/\"require\": {\"typo3\/swiftmailer\": \"5.3.1\",/g' composer.json
/usr/local/bin/php5-56STABLE-CLI ~/composer.phar update --no-dev

# !!! IMPORTANT !!!
# Here I upload the settings of the neos installation (these are commited in a seperate repository for the time being)
# You could do that however you like

## Migrate the database and create an initial user
export FLOW_CONTEXT=Production
cd ${neos_dir}
./flow doctrine:migrate
./flow user:create --roles Administrator ${neos_user} ${neos_password} Your Name
```

### 4. Prepare your site instance

Upload the code of your site instance here. Here I compile my sass files and upload relevant data to the following directory on the server:
```
${neos_dir}/Packages/Sites/${package_key}
```

### 5. Restore backups
```
# Since I had issues with restoring from a file outside of the neos directoy
# I use a workaround here: The backups are copied to the folder
# "${neos_dir}/Packages/Sites/${package_key}/Resources/Private/Content" 
# before they are restored.
mkdir ${neos_dir}/Packages/Sites/${package_key}/Resources/Private/Content
cp -r ${backup_dir}/${package_key}/* ${neos_dir}/Packages/Sites/${package_key}/Resources/Private/Content

cd ${neos_dir}
./flow site:import --filename ${neos_dir}/Packages/Sites/${package_key}/Resources/Private/Content/Sites.xml
./flow domain:add ${site_node} ${domain}
# Add nodes that are missing in the database but specified in NodeTypes.yml
./flow node:repair
```

## 6. Warmup caches
```
export FLOW_CONTEXT=Production
cd ${neos_dir}
./flow flow:cache:flush
./flow cache:warmup
```

These are basically all the steps needed to setup a fresh Typo3 neos instance from scratch. At first I had issues with the FLOW_CONTEXT when restoring and creating backups but I solved them with some workarounds. (like creating the "Resource" folder first) It took some time to get it all right and to have a stable repeatable deployment workflow but in the end I managed to get it working. Someday I may take a deeper look into Surf but for now the commands above are working perfectly. You see it is quite cool to have the ability to deploy a CMS in an automated and repeatable way out of the box while only using Jenkins and some standard Linux commands. I hope that this was helpful for you. If you ever encounter my [website](http://dbzman-online.eu) not being available this is most likely due to a running deployment. ;)
