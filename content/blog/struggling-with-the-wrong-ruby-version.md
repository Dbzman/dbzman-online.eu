+++
date = "2013-11-13T07:53:00+01:00"
title = "Struggling with the wrong Ruby version"

+++

At work I've recently setup [Dashing](http://shopify.github.io/dashing/) on an Ubuntu VM. I thought it would be nice to have a decent looking dashboard showing the build status and deployment times of our test environments. We are currently using Jenkins for CI and deployment which has a nice API to consume. 

Over time it turned out that many small fixes had to be made in the jobs (at the dashboard) to fit our environment perfectly. Everytime a change to the dashboard was made and commited I had to manually copy the code over to the dashboard server and restart thin. I began to look for a more elegant solution and decided to deploy Dashing via Jenkins. 

There were several start/stop scripts aleady available specifically for dashing. That was nice. I chose the one which had a restart option present. But that doesn't seem to work. There were no errors when I tried to start dashing via "sudo service dashboard start". After some time I found out that 
the command which starts the deamon supressed all errors. I changed that. The error was very clear as I specified the GEM_HOME of the Ruby 1.9.3 installation but it was executed with Ruby 1.8.7 which resulted in a segmentation fault. As the VM was deployed via chef and had two versions of ruby installed I tried to bend the system to use 1.9.3 which was kind of hacky. Don't *ever* do that. I then installed rvm to please my conscience and somehow get it working more elegantly. It indeed worked but only when being logged in via ssh in an interactive shell where I then had the correct ruby version loaded.

Excited about how Jenkins would react I've setup my Jenkins jobs only to find out that non-interactive shells (used when deploying via Jenkins) work completely different. ".bash_profile" or ".bashrc" do not get executed and therefore the contents of the PATH environment variable is never as expected. I solved the problem by specifing the Ruby 1.9.3 bin folder in the PATH directly in the script as you can see in the following excerpt:

    # Must be a valid filename
    NAME=dashing
    DASHING_DIR=/home/dashboard
    PIDFILE="$DASHING_DIR/$NAME.pid"
    DAEMON=/opt/chef/embedded/bin/$NAME
    
    export PATH=/opt/chef/embedded/bin:$PATH
    export GEM_HOME=/opt/chef/embedded/lib/ruby/gems/1.9.1
    export GEM_PATH=/opt/chef/embedded/lib/ruby/gems/1.9.1/gems
    
    DASHING_PORT=3030
    DAEMON_OPTS="start -d -p $DASHING_PORT -P $PIDFILE --tag $NAME -D"
*I am using the embedded version of ruby 1.9.3 which chef provided.*

Now I can easily deploy changes via Jenkins - the way it should be.
