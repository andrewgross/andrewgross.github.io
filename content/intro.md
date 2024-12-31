Title: Docker and Inodes
Date: 2024-12-29 2:38

Trying out Pelican as a static site generator. An interesting thing I noticed with my development environment is that docker is not great at seeing inode changes in mounted volumes. My setup is the following:

1. Write changes in VSCode (laptop)
2. VSCode Remote-SSH Session (server) handles syncing those changes from my laptop
3. I have the files in a Git Repo (server)
4. I have the Git Repo as a mounted volume in a docker container
5. The Pelican server runs in the docker container with `--autoreload`
6. The Pelican server does not notice changes to files and pick them up with autoreload.

My best guess is that docker doesn't get to see inode events for its mounted drives, at least in my default configuration.  A little digging shows [this](https://forums.docker.com/t/inotify-is-not-triggering-any-events-in-docker-when-we-mount-it-from-the-host/104570) issue from 2021. Seems like some folks don't see this same issue. Not worth the deep dive at this time, I just opened a second shell on the server and ran `watch -n 1` with the generation command.