# How to apply fun technologies you learn at work for home
![Servers](../assets/home-server/hp_micro_server.png)

##### originally posted at [IAS medium blog](https://medium.com/ias-tech-blog/how-to-apply-fun-technologies-you-learn-at-work-for-home-576126207125) at Dec 17, 2020

I love technology and I am lucky enough to get paid for pursuing this hobby. So it is no surprise that in my spare time at home I am experimenting with new technologies and leveraging those to make my life at home a bit more convenient. One of the latest projects I worked on at home was rebuilding my home server.

In this new socially distant reality, the tools I use at work (Docker and automation with Git and Jenkins) have helped me build a home server to unify and simplify entertainment, and connect with family and friends via:

- central media server to stream music and video
- personal cloud to backup photos
- firewall for network security
- family blog

## Docker and Docker Compose

At IAS, we use Docker, from development to deployment, both leveraging official images or building our own. Since Docker is so helpful, I decided to use it on my own server. I found the following benefits with this move:

- maintainable (each Docker component is scripted in a docker-compose.yml file)
- upgradable with ease (rebuild the service with the latest image)
- easy to organize and backup (all persistence Docker volumes live in their own directory)
- migratable (though my server crashed months ago, it was easy to rebuild everything on a new virtual machine)

## Automation with Git and Jenkins

Automation is the new mantra in our day-to-day work on the IAS Prime team, with Git and Jenkins playing a big role in that mindset. We automate spinning up environments, running tests, creating bug reports from Slack messages, deploying software updates, and AWS Autoscaling.

My main take-away from working on the AWS onboarding project with the Jenkins pipeline is how convenient it is to set up CICD with code. I was amazed by the results, so I borrowed the same approach for home. Now, I never have to log in to the server to run scripts. With the Jenkins web interface, I can review the job status and reports whenever needed. I run repeatable tasks in my home server with the following:

- Jenkins (in a Docker)
- Git (Gitea, a lightweight git server, in a Docker)

Then I create shell scripts to handle these repeatable maintenance tasks:

- auto-deployment of Docker image (by checking in a new docker-compose.yml file to Gitea which wired to Jenkins for the deployment)
- weekly backup of data (Jenkins Scheduler to pull the back up script from Gitea)
- weekly upgrade of docker images (Jenkins Scheduler to pull the upgrade script from Gitea)

I had a lot of fun rebuilding the server and it was a reminder of what I have learned from my daily work. It’s also a little showcase to my non-techy family members of what I do on the job.

Learning and using technology goes both ways. On weekends, I use an open-source Python project that I created to generate math quiz sheets for family and friends, which in turn have improved my Python skills for work.

Integrating technology into my family life reminded me about why I got into this field — I have a passion for technology. Sometimes I ask myself where all this learning has benefitted me most… is what I’m learning at work helping me at home, or is what I’m learning at home helping me at work?

At Integral Ad Science I love that my job allows for experimentation and learning new technologies. I am proud of the systems we have built over the last few years and I continue to be amazed by the volumes being processed (3B — 5B events per hour) through those systems. I love the team I am working with and feel that I grow as a developer.

At home I love playing with computer hardwares. My new hobby is setting up thin client machines around the house since they are low power (15 W) and silent (fan-less). I use them as a media streamer, for light web browsing and watching O’Reilly training video courses (a benefit of working in the IAS).

In the end, it doesn’t matter because it’s the technology that drives my happiness in my work and my personal life.