## 2.0 Webapps with Docker
Great! So you have now looked at `docker run`, played with a Docker container and also got the hang of some terminology. Armed with all this knowledge, you are now ready to get to the real stuff &#8212; deploying web applications with Docker.

### 2.1 Run a static website in a container
>**Note:** Code for this section is in this repo in the [static-site directory](https://github.com/docker/labs/tree/master/beginner/static-site).

Let's start by taking baby-steps. First, we'll use Docker to run a static website in a container. The website is based on an existing image. We'll pull a Docker image from Docker Store, run the container, and see how easy it is to set up a web server.

The image that you are going to use is a single-page website that was already created for this demo and is available on the Docker Store as [`dockersamples/static-site`](https://store.docker.com/community/images/dockersamples/static-site). You can download and run the image directly in one go using `docker run` as follows.

```bash
$ docker run -d dockersamples/static-site
```

>**Note:** The current version of this image doesn't run without the `-d` flag. The `-d` flag enables **detached mode**, which detaches the running container from the terminal/shell and returns your prompt after the container starts. We are debugging the problem with this image but for now, use `-d` even for this first example.

So, what happens when you run this command?

Since the image doesn't exist on your Docker host, the Docker daemon first fetches it from the registry and then runs it as a container.

Now that the server is running, do you see the website? What port is it running on? And more importantly, how do you access the container directly from our host machine?

Actually, you probably won't be able to answer any of these questions yet! &#9786; In this case, the client didn't tell the Docker Engine to publish any of the ports, so you need to re-run the `docker run` command to add this instruction.

Let's re-run the command with some new flags to publish ports and pass your name to the container to customize the message displayed. We'll use the *-d* option again to run the container in detached mode.

First, stop the container that you have just launched. In order to do this, we need the container ID.

Since we ran the container in detached mode, we don't have to launch another terminal to do this. Run `docker ps` to view the running containers.

```bash
$ docker ps
CONTAINER ID        IMAGE                  COMMAND                  CREATED             STATUS              PORTS               NAMES
a7a0e504ca3e        dockersamples/static-site   "/bin/sh -c 'cd /usr/"   28 seconds ago      Up 26 seconds       80/tcp, 443/tcp     stupefied_mahavira
```

Check out the `CONTAINER ID` column. You will need to use this `CONTAINER ID` value, a long sequence of characters, to identify the container you want to stop, and then to remove it. The example below provides the `CONTAINER ID` on our system; you should use the value that you see in your terminal.

```bash
$ docker stop a7a0e504ca3e
$ docker rm   a7a0e504ca3e
```

>**Note:** A cool feature is that you do not need to specify the entire `CONTAINER ID`. You can just specify a few starting characters and if it is unique among all the containers that you have launched, the Docker client will intelligently pick it up.

Now, let's launch a container in **detached** mode as shown below:

```bash
$ docker run --name static-site -e AUTHOR="Your Name" -d -P dockersamples/static-site
e61d12292d69556eabe2a44c16cbd54486b2527e2ce4f95438e504afb7b02810
```

In the above command:

*  `-d` will create a container with the process detached from our terminal
* `-P` will publish all the exposed container ports to random ports on the Docker host
* `-e` is how you pass environment variables to the container
* `--name` allows you to specify a container name
* `AUTHOR` is the environment variable name and `Your Name` is the value that you can pass

Now you can see the ports by running the `docker port` command.

```bash
$ docker port static-site
443/tcp -> 0.0.0.0:32772
80/tcp -> 0.0.0.0:32773
```

If you are running [Docker for Mac](https://docs.docker.com/docker-for-mac/), [Docker for Windows](https://docs.docker.com/docker-for-windows/), or Docker on Linux, you can open `http://localhost:[YOUR_PORT_FOR 80/tcp]`. For our example this is `http://localhost:32773`.

If you are using Docker Machine on Mac or Windows, you can find the hostname on the command line using `docker-machine` as follows (assuming you are using the `default` machine).

```bash
$ docker-machine ip default
192.168.99.100
```
You can now open `http://<YOUR_IPADDRESS>:[YOUR_PORT_FOR 80/tcp]` to see your site live! For our example, this is: `http://192.168.99.100:32773`.

You can also run a second webserver at the same time, specifying a custom host port mapping to the container's webserver.

```bash
$ docker run --name static-site-2 -e AUTHOR="Your Name" -d -p 8888:80 dockersamples/static-site
```

<img src="../images/static.png" title="static">

To deploy this on a real server you would just need to install Docker, and run the above `docker` command(as in this case you can see the `AUTHOR` is Docker which we passed as an environment variable).

Now that you've seen how to run a webserver inside a Docker container, how do you create your own Docker image? This is the question we'll explore in the next section.

But first, let's stop and remove the containers since you won't be using them anymore.

```bash
$ docker stop static-site
$ docker rm static-site
```

Let's use a shortcut to remove the second site:

```bash
$ docker rm -f static-site-2
```

Run `docker ps` to make sure the containers are gone.

```bash
$ docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
```

### 2.2 Docker Images

In this section, let's dive deeper into what Docker images are. You will build your own image, use that image to run an application locally, and finally, push some of your own images to Docker Cloud.

Docker images are the basis of containers. In the previous example, you **pulled** the *dockersamples/static-site* image from the registry and asked the Docker client to run a container **based** on that image. To see the list of images that are available locally on your system, run the `docker images` command.

```bash
$ docker images
REPOSITORY             TAG                 IMAGE ID            CREATED             SIZE
dockersamples/static-site   latest              92a386b6e686        2 hours ago        190.5 MB
nginx                  latest              af4b3d7d5401        3 hours ago        190.5 MB
python                 2.7                 1c32174fd534        14 hours ago        676.8 MB
postgres               9.4                 88d845ac7a88        14 hours ago        263.6 MB
containous/traefik     latest              27b4e0c6b2fd        4 days ago          20.75 MB
node                   0.10                42426a5cba5f        6 days ago          633.7 MB
redis                  latest              4f5f397d4b7c        7 days ago          177.5 MB
mongo                  latest              467eb21035a8        7 days ago          309.7 MB
alpine                 3.3                 70c557e50ed6        8 days ago          4.794 MB
java                   7                   21f6ce84e43c        8 days ago          587.7 MB
```

Above is a list of images that I've pulled from the registry and those I've created myself (we'll shortly see how). You will have a different list of images on your machine. The `TAG` refers to a particular snapshot of the image and the `ID` is the corresponding unique identifier for that image.

For simplicity, you can think of an image akin to a git repository - images can be [committed](https://docs.docker.com/engine/reference/commandline/commit/) with changes and have multiple versions. When you do not provide a specific version number, the client defaults to `latest`.

For example you could pull a specific version of `ubuntu` image as follows:

```bash
$ docker pull ubuntu:12.04
```

If you do not specify the version number of the image then, as mentioned, the Docker client will default to a version named `latest`.

So for example, the `docker pull` command given below will pull an image named `ubuntu:latest`:

```bash
$ docker pull ubuntu
```

To get a new Docker image you can either get it from a registry (such as the Docker Store) or create your own. There are hundreds of thousands of images available on [Docker Store](https://store.docker.com). You can also search for images directly from the command line using `docker search`.

An important distinction with regard to images is between _base images_ and _child images_.

- **Base images** are images that have no parent images, usually images with an OS like ubuntu, alpine or debian.

- **Child images** are images that build on base images and add additional functionality.

Another key concept is the idea of _official images_ and _user images_. (Both of which can be base images or child images.)

- **Official images** are Docker sanctioned images. Docker, Inc. sponsors a dedicated team that is responsible for reviewing and publishing all Official Repositories content. This team works in collaboration with upstream software maintainers, security experts, and the broader Docker community. These are not prefixed by an organization or user name. In the list of images above, the `python`, `node`, `alpine` and `nginx` images are official (base) images. To find out more about them, check out the [Official Images Documentation](https://docs.docker.com/docker-hub/official_repos/).

- **User images** are images created and shared by users like you. They build on base images and add additional functionality. Typically these are formatted as `user/image-name`. The `user` value in the image name is your Docker Store user or organization name.

### 2.3 Create your first image

Now that you have a better understanding of images, it's time to create your own. Our goal here is to create an image that sandboxes a small Javascript application.

**The goal of this exercise is to create a Docker image which will run a Javascript website app.**

We'll do this by first pulling together the components for our website, then _dockerizing_ it by writing a _Dockerfile_. Finally, we'll build the image, and then run it.

### 2.3.1 Download the Javascript contents

For the purposes of this workshop, we've created altered a fun little game and dockerized it slightly/

Start by creating a directory called ```floppy-bird``` where we'll download a github repository to:

```bash
$ mkdir floppy-bird
$ cd floppy-bird
```

If you don't already have it installed, you will need to a tool to download the github repository files to the folder - on Ubuntu we can install the following:

```bash
$ sudo apt install -yyq subversion
```

Now you can download the website files - they will come down into a folder called "trunk":

```bash
$ svn export https://github.com/tallmantim/floppybird/trunk
```

### 2.3.2 Write a Dockerfile

We want to create a Docker image with this web app. As mentioned above, all user images are based on a _base image_. Since our application is written in Javascript, we will build our own image based on [Nginx](https://store.docker.com/images/nginx). We'll do that using a **Dockerfile**.

A [Dockerfile](https://docs.docker.com/engine/reference/builder/) is a text file that contains a list of commands that the Docker daemon calls while creating an image. The Dockerfile contains all the information that Docker needs to know to run the app &#8212; a base Docker image to run from, location of your project code, any dependencies it has, and what commands to run at start-up. It is a simple way to automate the image creation process. The best part is that the [commands](https://docs.docker.com/engine/reference/builder/) you write in a Dockerfile are *almost* identical to their equivalent Linux commands. This means you don't really have to learn new syntax to create your own Dockerfiles.


1. Create a file called **Dockerfile**, and add content to it as described below.

  We'll start by specifying our base image, using the `FROM` keyword:

  ```
  FROM ubuntu:latest
  ```

2. We'll update the application registry and install our web server software now (Nginx):

  ```
  RUN apt update && apt install -yyq nginx
  ```

3. Let's add the files that make up the Javascript Application.

  ```
  COPY trunk/ /usr/share/nginx/html/
  ```

4. Specify the port number which needs to be exposed. Since our flask app is running as a web server on `80` that's what we'll expose.

  ```
  EXPOSE 80
  ```

5. The last step is the command for running the application which is simply - `nginx` with a couple of command line options. Use the [CMD](https://docs.docker.com/engine/reference/builder/#cmd) command to do that:

  ```
  CMD ["nginx", "-g", "daemon off;"]
  ```

  The primary purpose of `CMD` is to tell the container which command it should run by default when it is started.

6. Verify your Dockerfile.

  Our `Dockerfile` is now ready. This is how it looks:

  ```
  # our base image
  FROM ubuntu:latest

  # Install nginx
  RUN apt update && apt install -yyq nginx

  # copy files required for the app to run
  COPY trunk/ /var/www/html/

  # tell the port number the container should expose
  EXPOSE 80

  # run the application
  CMD ["nginx", "-g", "daemon off;"]
  ```

### 2.3.3 Build the image

Now that you have your `Dockerfile`, you can build your image. The `docker build` command does the heavy-lifting of creating a docker image from a `Dockerfile`.

When you run the `docker build` command given below, make sure to replace `<YOUR_USERNAME>` with your username. This username should be the same one you created when registering on [Docker Cloud](https://cloud.docker.com). If you haven't done that yet, please go ahead and create an account.

The `docker build` command is quite simple - it takes an optional tag name with the `-t` flag, and the location of the directory containing the `Dockerfile` - the `.` indicates the current directory:

```
$ docker build -t <YOUR_USERNAME>/myfirstapp .
Sending build context to Docker daemon  546.3kB
Step 1/5 : FROM ubuntu:latest
 ---> 2dc39ba059dc
Step 2/5 : RUN apt update && apt install -yyq nginx
 ---> Running in 7564956801b2

WARNING: apt does not have a stable CLI interface. Use with caution in scripts.

Get:1 http://archive.ubuntu.com/ubuntu jammy InRelease [270 kB]
....
....
....
Removing intermediate container 7564956801b2
 ---> fc3c175689b0
Step 3/5 : COPY trunk/ /var/www/html/
 ---> 2fbba6d76dfb
Step 4/5 : EXPOSE 80
 ---> Running in 4d76cc90d3ef
Removing intermediate container 4d76cc90d3ef
 ---> 514b1b463656
Step 5/5 : CMD ["nginx", "-g", "daemon off;"]
 ---> Running in 95601e78f6c0
Removing intermediate container 95601e78f6c0
 ---> faaae15761de
Successfully built faaae15761de
Successfully tagged myfirstapp:latest

```

If you don't have the `ubuntu:latest` image, the client will first pull the image and then create your image. Therefore, your output on running the command will look different from mine. If everything went well, your image should be ready! Run `docker images` and see if your image (`<YOUR_USERNAME>/myfirstapp`) shows.

### 2.3.4 Run your image
The next step in this section is to run the image and see if it actually works.

```bash
$ docker run -p 8888:80 --name myfirstapp YOUR_USERNAME/myfirstapp
 * Running on http://0.0.0.0:80/ (Press CTRL+C to quit)
```

Head over to `http://localhost:8888` or `http://<ipaddress>:8888`and your app should be live. **Note** If you are using Docker Machine, you may need to open up another terminal and determine the container ip address using `docker-machine ip default`.

### 2.3.4 Push your image
Now that you've created and tested your image, you can push it to [Docker Cloud](https://cloud.docker.com).

First you have to login to your Docker Cloud account, to do that:

```bash
docker login
```

Enter `YOUR_USERNAME` and `password` when prompted. 

Now all you have to do is:

```bash
docker push YOUR_USERNAME/myfirstapp
```

Now that you are done with this container, stop and remove it since you won't be using it again.

Open another terminal window and execute the following commands:

```bash
$ docker stop myfirstapp
$ docker rm myfirstapp
```

or

```bash
$ docker rm -f myfirstapp
```

### 2.3.5 Dockerfile commands summary

Here's a quick summary of the few basic commands we used in our Dockerfile.

* `FROM` starts the Dockerfile. It is a requirement that the Dockerfile must start with the `FROM` command. Images are created in layers, which means you can use another image as the base image for your own. The `FROM` command defines your base layer. As arguments, it takes the name of the image. Optionally, you can add the Docker Cloud username of the maintainer and image version, in the format `username/imagename:version`.

* `RUN` is used to build up the Image you're creating. For each `RUN` command, Docker will run the command then create a new layer of the image. This way you can roll back your image to previous states easily. The syntax for a `RUN` instruction is to place the full text of the shell command after the `RUN` (e.g., `RUN mkdir /user/local/foo`). This will automatically run in a `/bin/sh` shell. You can define a different shell like this: `RUN /bin/bash -c 'mkdir /user/local/foo'`

* `COPY` copies local files into the container.

* `CMD` defines the commands that will run on the Image at start-up. Unlike a `RUN`, this does not create a new layer for the Image, but simply runs the command. There can only be one `CMD` per a Dockerfile/Image. If you need to run multiple commands, the best way to do that is to have the `CMD` run a script. `CMD` requires that you tell it where to run the command, unlike `RUN`. So example `CMD` commands would be:

```
  CMD ["python", "./app.py"]

  CMD ["/bin/bash", "echo", "Hello World"]
```

* `EXPOSE` creates a hint for users of an image which ports provide services. It is included in the information which
 can be retrieved via `$ docker inspect <container-id>`.     

>**Note:** The `EXPOSE` command does not actually make any ports accessible to the host! Instead, this requires 
publishing ports by means of the `-p` flag when using `$ docker run`.  

* `PUSH` pushes your image to Docker Cloud, or alternately to a [private registry](https://docs.docker.com/registry/)

>**Note:** If you want to learn more about Dockerfiles, check out [Best practices for writing Dockerfiles](https://docs.docker.com/engine/userguide/eng-image/dockerfile_best-practices/).

## Next Steps
For the next step in the tutorial head over to [3.0 Deploying an app to a Swarm](./votingapp.md)
