Using Serf to control a cluster of Docker servers.
=================

First - Docker
--------

This year's introduction of [Docker](http://www.docker.io) has been huge for sysadmins everywhere. Whether or not you already understand what Docker can do for you - let me assure you - it has the potential to change how we think about servers.

<a href="http:/www.docker.io"><img src="http://github.froese.org/assets/sysadvent-2013/docker.png" align="right" width="150" border="0" /></a>

At [nonfiction](http://www.nonfiction.ca), we host a large number of web applications for customers. Some of those web applications were developed for a specific purpose and because they're often not *business critical*, they don't get a lot of regular updates. They don't have the budget or the desire to continue working on them year after year - upgrading as techonology matures. As a result, we have a number of pretty old web applications that work pretty well but are not based on current technology.

From a sysadmin perspective, deploying these old applications can be pretty complicated - the dependancies can be pretty hairy and are downright fickle. Because we can't always use [Heroku](https://www.heroku.com/) for the application, we end up having:

1. Ruby 1.8.x Server
2. Ruby 1.9.x Server
3. Specialized application Server for "that" project.
4. Node.js 0.8.x Server
5. Really old RHEL Server for that 11 year old PHP 4.x application. \(Yes - really.\)

That's not awesome at all.

More servers - especially servers that run a very low number of low-traffic apps - seems like a waste of resources, money and time. For example, here's the cpu usage for an old Ruby 1.8.x application server:

![CPU for the last month for an app server](http://github.froese.org/assets/sysadvent-2013/cpu-for-the-last-month.png)

Kind of a waste. Docker can change all of that.

With Docker you can run [containers](http://docs.docker.io/en/latest/terms/container/#container-def) for any of your applications and run all of those applications \(and more\) on a single server. No crazy gem / Ruby version problems - everything in it's own self-contained container. No more "I don't have python 3.3 on that server." or "Sorry - can't compile that version of Node on that old box - gotta move it."

We like Docker so much, that we're building a new service with it providing the backend infrastructure. Our backend is named [octohost](https://github.com/octohost/octohost) and is available on [Github](https://github.com/octohost).

Enter - Serf
---------

The [Serf](http://www.serfdom.io/) website bills it as:

<a href="http://www.serfdom.io/"><img src="http://github.froese.org/assets/sysadvent-2013/serf.png" align="right" border="0" /></a>

>Serf is a decentralized solution for service discovery 
>and orchestration that is lightweight, highly available, 
>and fault tolerant.

Basically, Serf is a new system built to pass messages around and trigger events from server to server - some [examples are listed on the website](http://www.serfdom.io/intro/use-cases.html). Instead of building your own messaging system or inventing a new daemon, you can connect a number of servers together using Serf and use it to trigger "events".

We're going to use it to connect a number of Docker servers together:

1. Compile server - this server compiles the software into a Docker container and pushes it to the Registry server.
2. Registry server - this server receives and stores the container.
3. Web server - this server pulls the container once it's ready to download and makes it available on the web.

We're going to kick this off from another server - by sending Serf events and making custom handlers for each "type" of server.

Serf has the concept of [Roles](http://www.serfdom.io/docs/agent/options.html) where you can tell a particular member of the cluster that it's a "\{insert-role-here\}" and only the events that apply to that role will be executed.

We're going to create some roles for our servers:

1. build
2. store
3. serve
4. master

Let's launch these servers:

`ec2-run-instances --key dfroese-naw -g sg-1a3b0e2a --user-data-file user-data-file/master ami-38204508 --region us-west-2`

Once we have the IP for that server, we'll launch the others and get them to join the serf cluster:

```
ec2-run-instances --key dfroese-naw -g sg-1a3b0e2a --user-data-file user-data-file/build ami-38204508 --region us-west-2
ec2-run-instances --key dfroese-naw -g sg-1a3b0e2a --user-data-file user-data-file/store ami-38204508 --region us-west-2
ec2-run-instances --key dfroese-naw -g sg-1a3b0e2a --user-data-file user-data-file/serve ami-38204508 --region us-west-2
```

We're using [Amazon's User Data](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/AESDG-chapter-instancedata.html) system to:

1. Set the system's Serf role.
2. Download the [Serf event handlers](https://github.com/darron/serf-docker-events).
3. [Activate](https://github.com/octohost/octohost/blob/master/config/serf.conf#L23-L32) those handlers.
4. Join the cluster by connecting to the first 'master' system.
5. Any additional setup for that role as needed.

Now that we've got the systems connected - let's send some test events.

`serf event role-check`

When that event is sent, each system executes [/etc/serf/handlers/role-check.sh](https://github.com/darron/serf-docker-events/blob/master/role-check.sh) - this is some of the output:

```
2013/12/02 23:07:21 Requesting user event send: role-check. Coalesced: true. Payload: ""
2013/12/02 23:07:22 [INFO] agent: Received event: user-event: role-check
2013/12/02 23:07:22 [DEBUG] Event 'user' script output: ip-10-250-69-116 role is master
2013/12/02 23:07:22 [DEBUG] Event 'user' script output: ip-10-250-65-99 role is store
2013/12/02 23:07:22 [DEBUG] Event 'user' script output: ip-10-225-185-80 role is serve
2013/12/02 23:07:22 [DEBUG] Event 'user' script output: ip-10-227-14-222 role is build
```

You can also watch what's going on with the entire cluster:

`serf monitor`

```
2013/12/02 23:07:21 Requesting user event send: role-check. Coalesced: true. Payload: ""
2013/12/02 23:07:21 [DEBUG] serf-delegate: messageUserEventType: role-check
2013/12/02 23:07:21 [DEBUG] serf-delegate: messageUserEventType: role-check
2013/12/02 23:07:21 [DEBUG] serf-delegate: messageUserEventType: role-check
2013/12/02 23:07:21 [DEBUG] serf-delegate: messageUserEventType: role-check
2013/12/02 23:07:21 [DEBUG] serf-delegate: messageUserEventType: role-check
2013/12/02 23:07:22 [INFO] agent: Received event: user-event: role-check
2013/12/02 23:07:22 [DEBUG] Event 'user' script output: ip-10-250-69-116 role is master
2013/12/02 23:07:23 [INFO] serf: EventMemberFailed: ip-10-225-185-80 10.225.185.80
2013/12/02 23:07:24 [INFO] agent: Received event: member-failed
2013/12/02 23:07:27 [INFO] Responding to push/pull sync with: 10.250.65.99:33341
2013/12/02 23:07:27 [INFO] serf: EventMemberJoin: ip-10-225-185-80 10.225.185.80
2013/12/02 23:07:28 [INFO] agent: Received event: member-join
2013/12/02 23:07:51 [INFO] Initiating push/pull sync with: 10.225.185.80:7946
```

Now let's do something useful.
----------

Let's tell the build server to compile a git repo.

`serf event compile|giturl`

Once it's done, it pushes it to the registry.

Once that's done, the serve system pulls it from the registry and makes it visible on the web.

You can add all sorts of other event handlers.
----------

`serf event delete-container`

`serf event disable-container`

Extra credit if I have time
----------

Connect this to [hubot](http://hubot.github.com/) through [capitoshka](https://github.com/darron/capitoshka) already running on [Heroku](http://capitoshka.herokuapp.com/projects).










