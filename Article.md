Using Serf to control a cluster of Docker servers.
=================

This year's introduction of [Docker](http://www.docker.io) has been huge for sysadmins everywhere. Whether or not you already understand what Docker can do for you - let me assure you - it has the potential to change how we think about servers.

At [nonfiction](http://www.nonfiction.ca), we host a large number of web applications for customers. Some of those web applications were developed for a specific purpose and because they're often not *business critical*, they don't get a lot of regular updates. They don't have the budget or the desire to continue working on them year after year - upgrading as techonology matures. As a result, we have a number of pretty old web applications that work pretty well but are not based on current technology.

From a sysadmin perspective, deploying these old applications can be pretty complicated - the dependancies can be pretty hairy and are downright fickle. Even though we use [Opscode Chef](http://www.opscode.com/chef/) to manage our servers, on occasions when we can't use [Heroku](https://www.heroku.com/) for the application, we occasionally still end up having:

1. Ruby 1.8.x Server
2. Ruby 1.9.x Server
3. Specialized application Server for "that" project.
4. Node.js 0.8.x Server
5. Really old RHEL Server for that 11 year old PHP 4.x application. \(Yes - really.\)

That's not awesome at all. More servers - especially servers that run a very low number of low-traffic apps - seems like a waste of resources, money and time:

![CPU for the last month for an app server](https://github.com/darron/sysadvent-docker/master/article-support/cpu-for-the-last-month.png)

Yes - I know about RVM, but haven't been a huge fan of using it on the server.

Docker can change all of that.

With Docker you can run [containers](http://docs.docker.io/en/latest/terms/container/#container-def) for any of your applications and run all of those applications \(and more\) on a single server. No crazy gem / Ruby version problems - everything in it's own self-contained container.



Servers:

1. Compile
2. INDEX / Storage
3. Web Serve