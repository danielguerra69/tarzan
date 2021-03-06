# DEPRECATED

This repository started as a good idea but I didn't have enough time or desire to work on it.  So, it's left here for historical / educational purposes, but be advised it's essentially dead code.  If anyone else wants to fork and develop / maintain it, be my guest :)

tarzan
======

### Self-hosted Automated Docker Builds that Run Rather Fast

![king of the apes](https://github.com/nathanleclaire/tarzan/blob/master/static/img/tarzan.jpg)

[Automated Builds](http://docs.docker.com/docker-hub/builds/) are one of the best features of the [Docker Hub](https://hub.docker.com).  They allow you to automatically re-create your Docker images on source control push, and they allow other people to find the `Dockerfile` used to create your image to inspect and play around with before pulling, running, or modifying it.

However, there is a big problem incorporating Automated Builds in a real-life workflow in their current form. 

- Because Docker's build robot runs Automated Builds using the `--no-cache` option (the infrastructure costs of not doing so would be prohibitive), all of the image layers are created from scratch each time.  
- This ends up in an Automated Build process which could take ten minutes or more (for an operation which would take seconds locally) and does not use the [Docker build cache](http://thenewstack.io/understanding-the-docker-cache-for-faster-builds/) (one of the best, most oft-cited features of `docker build`) at all.
- Because the layers are completely new, Docker's build robot pushes _all new layers_ when it pushes the built image back to Docker Hub, slowing the Automated Build down even more (the familiar `"image layer already exists, skipping"` message is nowhere to be found).
- When end users go to `docker pull` the image built using an Automated Build, they _always_ get new layers even if they have pulled that same image before.  This means that even if you only changed one character in one line of source code, you will most likely have to pull down anywhere from 80 megabytes to a gigabyte or more of Docker image layers.  This makes Automated Builds look very unattractive for real production deployments.

Therefore, it would be highly preferable to have an automated build robot which runs builds using the Docker cache.

`tarzan` is a naive implementation of such an automated build robot, written in [Go](http://golang.org).  I say it is "naive" because it doesn't attempt to do anything particularly clever (largely it shells out to `docker` commands) and is inherently meant to run on a single host (though this may change in the future).  However, it could still be turn out to be a useful tool for automating Docker image re-builds and deploys.

# Installation

You can install `tarzan` using either:

```
go get github.com/nathanleclaire/tarzan
```

Or install the (64bit Linux) binary directly using something like:

```
curl -SsL https://github.com/nathanleclaire/tarzan/releases/download/v0.0.1/tarzan | sudo tee /usr/local/bin/tarzan 2>&1>/dev/null; chmod +x /usr/local/bin/tarzan
```

Make sure you have `docker` and `git` binaries installed as well.  On Debian based systems, the following should work:

```
sudo apt-get install -y git-core
curl get.docker.com | sudo sh
```

As mentioned in the next section, `tarzan` is also available to run in a Docker container.

# Getting Started

First you should get a virtual private server of the appropriate size (a small server at, say, [Digital Ocean](http://digitalocean.com) will probably do nicely to start - they have a built-in Docker image as well).

![](/static/img/digitalocean.png)

Alternatively, if you just want to try `tarzan` out the "fast and dirty" way, you could use a proxy tunneling service such as the excellent [ngrok](http://ngrok.com) and run it locally on your computer.

Install `tarzan` on the worker box and start it.  Make sure that the port you're exposing on is accessible to the outside world.  `80` will do fine if you plan on using this box only for `tarzan`.

```
tarzan -p 80
[negroni] listening on :80
```

Tarzan is also available to run as a Docker container (built, of course, using `tarzan`):

```
docker run -d -p 80:3000 \
    -v /var/run/docker.sock:/var/run/docker.sock \
    -v $HOME/.dockercfg:/home/gobuddy/.dockercfg \
    -v $(which docker):/usr/bin/docker \
    nathanleclaire/tarzan
```

On Github, click on "Settings" (the bottom-most element) in the right hand panel, then click on "Webhooks & Services".

![](/static/img/webhooks.png)

Click the "Add webhook" button, verify your identity, and paste the IP address of your `tarzan` server at the `/build` endpoint in the URL box.  If you like, you can also choose a secret (to prevent spamming of your webhook endpoint) and pass it to `tarzan` with the `--secret` option:

```
tarzan --secret mySecret -p 80
```

Or (since `tarzan` is set as a `ENTRYPOINT` in the Docker image):

```
docker run -d -p 80:3000 \
    -v /var/run/docker.sock:/var/run/docker.sock \
    -v $HOME/.dockercfg:/home/gobuddy/.dockercfg \
    -v $(which docker):/usr/bin/docker \
    nathanleclaire/tarzan
        --secret mySecret
```

Now when you push code to the `master` branch, your image will automatically be rebuilt and pushed to Docker Hub.  By default, `tarzan` assumes that you have the same Docker Hub username as your Github username, but you can specify a different one using `--hub-name`: 

```
tarzan --hub-name myalias -p 80
```

Tarzan runs using the build cache, so this process is relatively speedy and will work with as many images/repos as your disk has space for.  

Naturally, the beefier that your server is, the faster the builds will run, and the fatter that your pipes are, the faster images will get moved to and fro `(Git|Docker) Hub`.

As noted in the FAQ, you can also use `fig` to bootstrap your own version of the registry running alongside `tarzan` (using the `--alt-registry` option) in case you want to use that instead of Docker Hub to push and pull images from. 

# Working on tarzan

If you want to hack on `tarzan` simply:

- Make sure you have `docker` installed on your system (only Linux supported for now)
- Run `make` in the project's root

`tarzan` will compile with all needed dependencies (they are vendored) inside of a container and spit out a binary to the host.

# FAQ

#### Q: Is this compatible with Docker Hub's automated builds?

Yes and no.  Docker Hub does not allow users to push to automated builds manually (using `docker push`), so it is impossible to use a official Automated Build Docker Hub repository as a backend for your `tarzan` build.  However, nothing is stopping you from creating _two_ Docker Hub repositories (and two separate webhooks on Github) and using one as a normal automated build (allowing for `Dockerfile` discovery) and one as a `tarzan` build (allowing for fast build and pull). 

#### Q: Can I run this using my own registry instead of using Docker Hub as a backend?

Yes.  Provided in this repository is a `fig.yml` file which will allow you to run `fig up` in the project's directory and bootstrap an instance of `tarzan` running alongside a local instance of the [Docker open-source registry](https://github.com/docker/docker-registry) as a backend.  That way, you can also push and pull images from the same host where you are running `tarzan` using the Docker `image.location.com/imagename` format.

#### Q: What do I do if my system's Docker binary is called `docker.io` instead of `docker`?

Simply use the `--docker-binary-name` flag:

```
tarzan --docker-binary-name docker.io -p 80
```

#### Q: Will this work with Docker Hub private repositories?

Yep!  As long as the user you've authenticated as can push to the corresponding private Docker Hub repositories, `tarzan` will be able to push to them.  The same caveats about true Automated Builds still apply of course.

#### Q: Is there a shiny GUI or web interface so I can see the status of my builds?

For now, STDOUT/STDERR logging is what `tarzan` offers.  Eventually I want to expand it to have a (mostly pretty minimalistic, but useful) web interface.

#### Q: Can I build [docker/docker](https://github.com/docker/docker) with it and do all kinds of weird stuff that Docker Hub won't let me do?

Yuuuuuuup!  It's your own server, do whatever you want.

#### Q: Why is the project called `tarzan`?

Partially because it is meant to be a wild and feral tool, refusing to be tamed by the confines of civilazation, but there is also a secret meaning and I will buy you a beer, coffee, cookie etc. if you figure it out.
