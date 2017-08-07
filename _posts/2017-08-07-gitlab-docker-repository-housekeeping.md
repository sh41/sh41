---
title: Gitlab Omnibus with Docker Repository Housekeeping
layout: default
tags: gitlab docker
---
I do love a bit of [Github](https://github.com), but for our commercial work we needed a local install so we've been using an internal installation of [Gitlab-CE Omnibus edition](https://about.gitlab.com/installation/) for the past year and generally it works quite well for our purposes. 

For the most part Gitlab-CE just works and doesn't require a lot of administration or oversight. We've got a reasonably tight configuration defined in Puppet using [vshn's puppet-gitlab module](https://github.com/vshn/puppet-gitlab) and a cron job that does a daily local backup which is then shifted offsite by our usual backup system. 

A couple of months ago we started a new project which allowed us to have a crack at getting CI up and running and rather than bring in another tool to learn like Jenkins we decided to give Gitlab-CI a spin. For our purposes it works perfectly and we're now happily hosting a local Docker Registry on the same box as Gitlab and have acouple of Docker Swarms (staging and production) that use the built images. 

All was well for the first couple of months, but this morning something wasn't quite right, pushes were failing and logging in to the Gitlab web interface was taking much longer than usual. Fortunately our monitoring system already had the answer for us: We had run out of disk space on the server. Not ideal, but no big deal, there must be temp files or something that can be removed. 

To cut a long story short we narrow it down to the nightly backups. Each nightly backup was coming in at 6GiB and keeping 7 days of backup online meant that over 40GiB of space was being used by the backups. I removed the oldest backup and the service was back up and running again, but 6GiB seemed a bit high to me so I dug further. 

The root cause appears to be that docker's registry installed with Gitlab-CE Omnibus edition never cleans up old images, so for every build pushed to the registry - even if it overwrites a build of the same tag, the space is not reclaimed. 

There is a `/usr/bin/gitlab-ctl registry-garbage-collect` command that stops the registry service invokes the docker built in garbage collection and then starts the registry service again, but unless you have deleted the reference to the image in the Gitlab UI nothing will be GC'd and if you've overwritten an image with one of the same name, you can't actually trigger a delete.

The solution ended up being slightyly more long winded than I'd prefer (and scarily marked as expermimental), but it appears to work for now, so there you go: 
Install [Docker Distribution Pruner](https://gitlab.com/gitlab-org/docker-distribution-pruner). Checkout https://gitlab.com/gitlab-org/docker-distribution-pruner/issues/2 for a step by step of getting go installed. 

Add the pruner and the Garbage Collection scripts to run before the backup cronjob: 
```
0 23 * * * /usr/share/go-1.6/src/gitlab.com/gitlab-org/docker-distribution-pruner/docker-distribution-pruner -config=/var/opt/gitlab/registry/config.yml -delete -soft-delete=true && /usr/bin/gitlab-ctl registry-garbage-collect && /opt/gitlab/bin/gitlab-rake gitlab:backup:create
```
So at 23:00 every evening we run the pruner (in soft delete mode) then the garbage collection then the actual backup. If you are brave you can change to `-soft-delete=false` which will actually delete the files rather than just move them... there are some pretty dire warnings on the project page though, so you may want to hold fire on that to start off with. 

Now the backups won't be bloated by totally unused layers from the registry and if you are happy with the pruning you can manually delete the files in `/var/opt/gitlab/gitlab-rails/shared/registry/docker_backup/`. 
