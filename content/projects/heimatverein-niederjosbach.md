+++
date = "2016-03-13T22:07:12+01:00"
title = "Heimatverein Niederjosbach"
image = "nhgv.png"
website = "http://heimatverein-niederjosbach.de/"
time = "2007 - today"
+++

This website has been done for a local history club I am also a member of. It has a rather long history and had switched it's foundation quite often. Starting with Joomla, over to Typolight (now known as Contao), to Wordpress, Typo3 (because the company I work for is a Typo3 expert) and now to Hugo. I've chosen Hugo (which is a static site generator) in the at the end because the performance of Typo3 was really bad on the server and also a CMS in general was not needed since the content doesn't change that often and we only have one guy maintaining the site. In a couple of weeks I completely ported the Typo3 version over to Hugo. Also the gallery was fairly easy to implement. With this relaunch I also decided to open source the whole website on Github and TravisCI to check and deploy assets. I've also written a small tool which handles the deployment (generate the content) on the target server called [hookdeploy](https://github.com/Dbzman/hookdeploy). All it does is executing a script on the filesystem when you call it with a valid API key. Maintaining the site - like adding content (in Markdown) - became a piece of cake. Since the site is only generated during deployment and once a day (to get events listings right) the performance gain is massive because the server now only has to serve static pre-generated HTML files.
