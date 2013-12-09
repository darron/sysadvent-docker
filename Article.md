Using Serf to control a cluster of Docker servers.
=================

First - Docker and Dependancy Management
--------

This year's introduction of [Docker](http://www.docker.io) has been huge for sysadmins everywhere. Whether or not you already understand what Docker can do for you - let me assure you - it has the potential to change how we think, work and build services.

<a href="http:/www.docker.io"><img src="http://github.froese.org/assets/sysadvent-2013/docker.png" align="right" width="150" border="0" /></a>

At [nonfiction](http://www.nonfiction.ca), we host a large number of web applications for customers. Some of those web applications were developed for a specific purpose and because they're often not *business critical*, they don't get a lot of regular updates. They don't have the budget or the desire to continue working on them year after year - upgrading as techonology matures. As a result, we have a number of pretty old web applications that work pretty well but are not based on current technology.

From a sysadmin perspective, deploying these old applications can be pretty complicated - the dependancies can be pretty hairy and are downright fickle. Although we often us [Heroku](https://www.heroku.com/) for many applications, we stil end up having:

1. Ruby 1.8.x Server
2. Ruby 1.9.x Server
3. Specialized application Server for "that" project.
4. Node.js 0.8.x Server
5. Really old RHEL Server for that 11 year old PHP 4.x application. \(Yes - really.\)

That's not awesome at all.

More servers - especially servers that run a very low number of low-traffic apps - seems like a waste of resources, money and time. We are paying for too much capacity in order to get cleaner dependency management. For example, here's the cpu usage for an old Ruby 1.8.x application server:

![CPU for the last month for an app server](http://github.froese.org/assets/sysadvent-2013/cpu-for-the-last-month.png)

What a waste.

Docker changes all of that.

With Docker you can run [containers](http://docs.docker.io/en/latest/terms/container/#container-def) for any of your applications and run all of those applications \(and more\) on a single server. No crazy gem / Ruby version problems - everything in it's own self-contained container. No more "I don't have python 3.3 on that server." or "Sorry - can't compile that version of Node on that old box - gotta move it."

We like Docker so much, that we're building a new service with it providing the backend infrastructure. Our backend is named [octohost](https://github.com/octohost/octohost) and is available on [Github](https://github.com/octohost).

Enter - Serf
---------

The [Serf](http://www.serfdom.io/) website bills it as:

<a href="http://www.serfdom.io/"><img src="http://github.froese.org/assets/sysadvent-2013/serf.png" align="right" border="0" /></a>

>Serf is a decentralized solution for service discovery 
>and orchestration that is lightweight, highly available, 
>and fault tolerant.

In short, Serf is a system built to pass messages around and trigger events from server to server - some [examples are listed on the website](http://www.serfdom.io/intro/use-cases.html). Instead of building your own messaging system or inventing a new daemon, you can connect a number of servers together using Serf and use it to trigger "events".

We're going to use it to connect some Docker servers together:

1. Compile server - this server compiles the software into a Docker container and pushes it to the Registry server.
2. Registry server - this server receives and stores the container. \(We are cheating and using the regular Docker INDEX for this.\)
3. Web server - this server pulls the container once it's ready to download and makes it available on the web.

Serf has the concept of [Roles](http://www.serfdom.io/docs/agent/options.html) where you can tell a particular member of the cluster that it's a "\{insert-role-here\}" and only the events that apply to that role will be executed.

We're going to create some roles for our servers:

1. build
2. serve
3. master

Let's launch these servers:

`ec2-run-instances --key dfroese-naw -g sg-1a3b0e2a --user-data-file user-data-file/master ami-38204508 --region us-west-2`

Once we have the IP for that server, we'll launch the others and get them to join the serf cluster:

```
ec2-run-instances --key dfroese-naw -g sg-1a3b0e2a --user-data-file user-data-file/build ami-38204508 --region us-west-2
ec2-run-instances --key dfroese-naw -g sg-1a3b0e2a --user-data-file user-data-file/serve ami-38204508 --region us-west-2
```

We're using [Amazon's User Data](http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/AESDG-chapter-instancedata.html) system to:

1. Set the system's Serf role.
2. Download the [Serf event handlers](https://github.com/darron/serf-docker-events).
3. [Activate](https://github.com/octohost/octohost/blob/master/config/serf.conf#L23-L32) those handlers.
4. Join the cluster by connecting to the first 'master' system.
5. Any additional setup for that role as needed. [Take a look at the user-data-files here.](https://github.com/darron/sysadvent-docker/tree/master/user-data-file)

Now that we've got the systems connected - let's send some test events.

`serf event role-check`

When that event is sent, each system executes [/etc/serf/handlers/role-check.sh](https://github.com/darron/serf-docker-events/blob/master/role-check.sh) - this is some of the output:

```
2013/12/02 23:07:21 Requesting user event send: role-check. Coalesced: true. Payload: ""
2013/12/02 23:07:22 [INFO] agent: Received event: user-event: role-check
2013/12/02 23:07:22 [DEBUG] Event 'user' script output: ip-10-250-69-116 role is master
2013/12/02 23:07:22 [DEBUG] Event 'user' script output: ip-10-225-185-80 role is serve
2013/12/02 23:07:22 [DEBUG] Event 'user' script output: ip-10-227-14-222 role is build
```

You can also watch what's going on through the entire cluster:

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

Let's tell the Docker server cluster to:

1. Compile a git repo.
2. Push it to the Docker INDEX
3. Have another server pull that container.
4. Then launch it so it's viewable from the web.

Here's how we kick it off:

`serf event build https://github.com/darron/sysadvent-harp.git,sysadvent/harp-example`

Which outputs:

```
Event 'build' dispatched! Coalescing enabled: true
2013/12/03 21:07:23 [INFO] Initiating push/pull sync with: 10.249.27.132:7946
2013/12/03 21:07:23 [DEBUG] serf-delegate: messageUserEventType: build
2013/12/03 21:07:23 [DEBUG] serf-delegate: messageUserEventType: build
2013/12/03 21:07:23 [DEBUG] serf-delegate: messageUserEventType: build
2013/12/03 21:07:23 [DEBUG] serf-delegate: messageUserEventType: build
2013/12/03 21:07:24 [INFO] agent: Received event: user-event: build
2013/12/03 21:08:51 Requesting user event send: pull. Coalesced: true. Payload: "sysadvent/harp-example"
2013/12/03 21:08:51 [DEBUG] Event 'user' script output: Build: https://github.com/darron/sysadvent-harp.git as sysadvent/harp-example in /tmp/tmp.WcrVsKOn1m
```

The [build starts](https://github.com/darron/serf-docker-events/blob/master/build.sh):

```
Cloning into '/tmp/tmp.WcrVsKOn1m'...
/usr/bin/docker build -t sysadvent/harp-example /tmp/tmp.WcrVsKOn1m
Uploading context 317440 bytes
Step 1 : FROM octohost/nodejs
 ---> 62108a2c615f
Step 2 : ADD . /srv/www
 ---> 5b01d85275dd
Step 3 : RUN cd /srv/www; npm install
 ---> Running in b136131fcb72
npm WARN package.json Harp@1.0.0 No repository field.
npm http GET https://registry.npmjs.org/harp
npm http 200 https://registry.npmjs.org/harp
# Removed lots of npm output.
harp@0.8.13 node_modules/harp
├── mime@1.2.9
├── async@0.2.9
├── mkdirp@0.3.4
├── commander@1.1.1 (keypress@0.1.0)
├── fs-extra@0.3.2 (jsonfile@0.0.1, ncp@0.2.7, rimraf@2.0.3)
├── jade@0.27.7 (commander@0.6.1, coffee-script@1.4.0)
├── less@1.3.1
├── connect@2.7.0 (fresh@0.1.0, cookie-signature@0.0.1, debug@0.7.4, pause@0.0.1, cookie@0.0.5, bytes@0.1.0, crc@0.2.0, formidable@1.0.11, qs@0.5.1, send@0.1.0)
└── terraform@0.4.12 (lru-cache@2.3.0, marked@0.2.8, ejs@0.8.4, coffee-script@1.6.3, jade@0.28.2, stylus@0.33.1, less@1.3.3)
 ---> af20e73caec4
Step 4 : EXPOSE 5000
 ---> Running in 2fce89dcbaa9
 ---> 688c52c7f926
Step 5 : CMD cd /srv/www; /usr/bin/node server.js
 ---> Running in 193e082814c4
 ---> 5a69fee1c103
Successfully built 5a69fee1c103
```

The build server pushes the [built container to the registry and kicks off the pull](https://github.com/darron/serf-docker-events/blob/master/build.sh#L19-L21):

```
Login Succeeded
The push refers to a repository [sysadvent/harp-example] (len: 1)
Sending image list
Pushing repository sysadvent/harp-example (1 tags)
# Lots of output removed.
```

Then the server in the 'serve' role [pulls the container](https://github.com/darron/serf-docker-events/blob/master/pull.sh#L10-L11):

```
Event 'pull' dispatched! Coalescing enabled: true
2013/12/03 21:08:51 [DEBUG] serf-delegate: messageUserEventType: pull
2013/12/03 21:08:52 [DEBUG] serf-delegate: messageUserEventType: pull
2013/12/03 21:08:52 [INFO] agent: Received event: user-event: pull
2013/12/03 21:08:52 [DEBUG] Event 'user' script output: Pull: sysadvent/harp-example
Pulling repository sysadvent/harp-example
5a69fee1c103: Pulling image (latest) from sysadvent/harp-example5a69fee1c103: Pulling image (latest) from sysadvent/harp-example, endpoint: https://cdn-registry-1.docker.io/v1/5a69fee1c103: Pulling dependent layers
# More output removed.
```

So that [it can run it](https://github.com/darron/serf-docker-events/blob/master/run.sh):

```
2013/12/03 21:10:16 [DEBUG] serf-delegate: messageUserEventType: run
2013/12/03 21:10:16 [DEBUG] serf-delegate: messageUserEventType: run
2013/12/03 21:10:16 [DEBUG] serf-delegate: messageUserEventType: run
2013/12/03 21:10:17 [INFO] agent: Received event: user-event: run
2013/12/03 21:10:23 [INFO] Responding to push/pull sync with: 10.231.7.223:60001
2013/12/03 21:10:24 [WARN] Potential blocking operation. Last command took 46.701165ms
2013/12/03 21:10:29 [DEBUG] Event 'user' script output: Run: sysadvent/harp-example
```

At the end of this process, the site was available at: http://harp-example.54.202.94.50.xip.io/

<a href="http://sysadvent-harp.octohost.io/"><img src="http://github.froese.org/assets/sysadvent-2013/sysadvent-web.jpg" align="right" width="400" border="0" /></a>

Which looked like this: 

To sum up
----------

Serf is a new tool that has been added to our toolboxes as sysadmins.

It's very powerful, simple to setup and can be extended in almost limitless ways.

Give Serf a [try](http://www.serfdom.io/). My example Serf handlers are all available [here](https://github.com/darron/serf-docker-events). You can even use the same AMI that I used for this article - ami-38204508.

[Let me know](http://darron.froese.org/) if you've got any questions!



