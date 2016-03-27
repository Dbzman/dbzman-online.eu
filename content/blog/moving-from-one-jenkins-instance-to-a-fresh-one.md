+++
date = "2014-08-11T21:46:00+01:00"
title = "Moving from one Jenkins instance to a fresh one (preserving configuration)"
list_image = "/images/blog/2014/08/jenkins_logo.png"
aliases = [
  "moving-from-one-jenkins-instance-to-a-fresh-one/"
]

+++

![](/images/blog/2014/08/jenkins_logo.png)

At my current project (.NET web application with Microsoft SQL Server) I am responsible for configuration management. For some time now we were using Jenkins for continuous integration and also for deployment of our test environments. Basically we had the following setup:

- Jenkins (master)
- Jenkins01 (slave)
- Jenkins02 (slave)
- Jenkins03 (slave)

Every of the above mentioned machines (Jenkins03 is just a VM) is based on **Windows Server 2008** with **.NET 4.0** installed.
For an upcoming release we planned to upgrade our application to run on **.NET 4.5**, **SQL Server 2012** and **Windows Server 2012**. As .NET 4.5 is a replacement of .NET 4.0 it's kind of problematic when developing older releases (.NET 4.0) and future releases (.NET 4.5) in parallel. Because .NET 4.5 **replaces** version 4.0 we cannot make sure that everything works as expected when we upgrade all environments to 4.5 and build the older releases. We are therefore forced to strictly isolate the environments. That leads us to the following constellation:

- Jenkins2012 (new master, Windows Server 2012, for everything new)
- Jenkins (**old master**, builds all the old jobs)
- Jenkins01 (slave of Jenkins2012)
- Jenkins02 (slave of Jenkins2012)
- Jenkins03 (slave of Jenkins2012)

I've gone the following path to install a fresh Jenkins instance while preserving the configs:

- installed and started Jenkins on Jenkins2012 (template VM with Windows Server 2012)
- copied old jobs to new Jenkins:

```bash
cp -r X:/Jenkins/jobs/T02*/*.xml C:/ --parents
```

- reloaded the config on the new Jenkins instance (doesn't kill running jobs)
  - all jobs were then available
- checked the config on both Jenkins instances
  - some steps were missing on the new Jenkins because of several plugins which were not yet installed
 - the first "save" on a job with missing plugins removes the steps from the job (*be careful here!*)
- rewired all job connections (downstream jobs weren't triggered at the first try)
- kept the old master for reference

As you can see most of the work went pretty straightforward. Simply copying all .xml files and just reloading the config has saved us many manual steps. The only thing we had to do manually was to rewire all the downstream jobs.
