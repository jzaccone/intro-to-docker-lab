# Lab 2- Adding Value to CI_CD with docker Images

© Copyright IBM Corporation 2017

IBM, the IBM logo and ibm.com are trademarks of International Business Machines Corp., registered in many jurisdictions worldwide. Other product and service names might be trademarks of IBM or other companies. A current list of IBM trademarks is available on the Web at &quot;Cfopyright and trademark information&quot; at www.ibm.com/legal/copytrade.shtml.

This document is current as of the initial date of publication and may be changed by IBM at any time.

The information contained in these materials is provided for informational purposes only, and is provided AS IS without warranty of any kind, express or implied. IBM shall not be responsible for any damages arising out of the use of, or otherwise related to, these materials. Nothing contained in these materials is intended to, nor shall have the effect of, creating any warranties or representations from IBM or its suppliers or licensors, or altering the terms and conditions of the applicable license agreement governing the use of IBM software. References in these materials to IBM products, programs, or services do not imply that they will be available in all countries in which IBM operates. This information is based on current IBM product plans and strategy, which are subject to change by IBM without notice. Product release dates and/or capabilities referenced in these materials may change at any time at IBM&#39;s sole discretion based on market opportunities or other factors, and are not intended to be a commitment to future product or feature availability in any way.

# Overview

In this lab, we build on our knowledge from lab 1. You will start to add value by building your own custom built Docker images. You will create your own Dockerfile, and learn how to push an image to a central registry. All of these concepts can be automated into a CI/CD pipeline to enable continuous delivery of dockerized applications.

## Prerequisites

You must have docker installed, or be using http://play-with-docker.com


# Step 1: Create a python app (without using Docker)

1. Create `app.py` with the following contents

```python
from flask import Flask

app = Flask(__name__)

@app.route("/")
def hello():
    return "hello world!"


if __name__ == "__main__":
    app.run(host='0.0.0.0')
```

This is a simple python app that uses flask to expose a http web server on port 5000. Don't worry if you are not too familiar with python or flask, these concepts can be applied to an application written in any language.

Optionally, If you have python and pip installed, you can run this app locally. If not, move on to the next step.

```sh
$ python --version
Python 2.7.13
$ pip --version
pip 9.0.1 from /usr/local/lib/python2.7/site-packages (python 2.7)
$ pip install python
Requirement already satisfied: python in /usr/local/Cellar/python/2.7.13/Frameworks/Python.framework/Versions/2.7/lib/python2.7/lib-dynload
$ python app.py 
 * Running on http://0.0.0.0:5000/ (Press CTRL+C to quit)
```

# Step 2: Create and build the Docker Image

If you don't have python install locally, don't worry! Because you don't need it. One of the advantages of using Docker containers is that you can build python into your containers, without having python installed on your host. 

1. Create a `Dockerfile` with the following contents
```sh
FROM python:2.7-alpine
RUN pip install flask
CMD ["python","app.py"]
COPY app.py /app.py
```

A Dockerfile lists the commands needed to build an Docker image. Let's go through the above file line by line.

**FROM pythong:2.7-alpine**
This is the starting point for your Dockerfile. Docker images are layered, and you need to select a starting image to build your layers on top of. In this case, we are selecting the `python:2.7-alpine` base layer since it already has the version of python and pip that we need to run our application. The `alpine`, version means that it uses the alpine base image, which is much smaller and lightweight than an alternative flavor of linux. A smaller image means it will build much faster, and it also has advantages for security because it has a smaller attack surface.

For security reasons, it is very important to understand the layers that you build your docker image on top of. For that reason, it is highly recommended to only use "official" images found in the [docker hub](https://hub.docker.com/), or non-community images found in the docker-store. You can find more information about this [python base image](https://store.docker.com/images/python), as well as all other images that you can use, on the [docker store](https://store.docker.com/).

**RUN pip install flask**
Use the `RUN` command to execute commands needed to set up your image for your application. In this case we are installing flask. The `RUN` commands are executed at build time, and are added to the layers of your image. 

**CMD ["python","app.py"]**
`CMD` (not to be confused with `RUN`) is the command that is executed when you start your container. We can only specify one of these. Here we are using `CMD` to run our python app.

**COPY app.py /app.py**
This copies the app.py in the local directory (where you will run `docker build`) into a new layer of the image. It seems counter-intuitive to put this after the `CMD ["python","app.py"]` line. Remember, the `CMD` line is executed only when the container is started, so we won't get a `file not found` error here. Ordering in this way takes advantage of the build cache, which we will demonstrate a little later in this lab.

And there you have it: is a very simple Dockerfile. A full list of commands you can put into a Dockerfile can be found [here](https://docs.docker.com/engine/reference/builder/). Now that we defined our Dockerfile, let's use it to build our custom docker image.

2. Build the docker image. 

Pass in `-t` to name your image `hello-world`.

```sh
$ docker build -t hello-world .
Sending build context to Docker daemon  33.28kB
Step 1/4 : FROM python:2.7-alpine
2.7-alpine: Pulling from library/python
79650cf9cc01: Pull complete 
581a2604819e: Pull complete 
a7826377ee0a: Pull complete 
406f1af34710: Pull complete 
Digest: sha256:a362956d36a22747e0a67687d58cab7e8c30be178fda1bfa6c2caa15bbf9f5cb
Status: Downloaded newer image for python:2.7-alpine
 ---> 29c1b4c9b149
Step 2/4 : RUN pip install flask
 ---> Running in 2274ff106d17
Collecting flask
  Downloading Flask-0.12.1-py2.py3-none-any.whl (82kB)
Collecting itsdangerous>=0.21 (from flask)
  Downloading itsdangerous-0.24.tar.gz (46kB)
Collecting click>=2.0 (from flask)
  Downloading click-6.7-py2.py3-none-any.whl (71kB)
Collecting Jinja2>=2.4 (from flask)
  Downloading Jinja2-2.9.6-py2.py3-none-any.whl (340kB)
Collecting Werkzeug>=0.7 (from flask)
  Downloading Werkzeug-0.12.1-py2.py3-none-any.whl (312kB)
Collecting MarkupSafe>=0.23 (from Jinja2>=2.4->flask)
  Downloading MarkupSafe-1.0.tar.gz
Building wheels for collected packages: itsdangerous, MarkupSafe
  Running setup.py bdist_wheel for itsdangerous: started
  Running setup.py bdist_wheel for itsdangerous: finished with status 'done'
  Stored in directory: /root/.cache/pip/wheels/fc/a8/66/24d655233c757e178d45dea2de22a04c6d92766abfb741129a
  Running setup.py bdist_wheel for MarkupSafe: started
  Running setup.py bdist_wheel for MarkupSafe: finished with status 'done'
  Stored in directory: /root/.cache/pip/wheels/88/a7/30/e39a54a87bcbe25308fa3ca64e8ddc75d9b3e5afa21ee32d57
Successfully built itsdangerous MarkupSafe
Installing collected packages: itsdangerous, click, MarkupSafe, Jinja2, Werkzeug, flask
Successfully installed Jinja2-2.9.6 MarkupSafe-1.0 Werkzeug-0.12.1 click-6.7 flask-0.12.1 itsdangerous-0.24
 ---> e18d56f632b7
Removing intermediate container 2274ff106d17
Step 3/4 : COPY app.py /app.py
 ---> e60dd8252e16
Removing intermediate container 36cf4dd13360
Step 4/4 : CMD python app.py
 ---> Running in 0015d00055cc
 ---> 967c8b52dda9
Removing intermediate container 0015d00055cc
Successfully built 967c8b52dda9
Successfully tagged hello-world:latest
```

Verify that your image shows up in your image list via `docker image ls`.

```sh
$ docker image ls
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
hello-world         latest              9c7a01569645        4 seconds ago       83.4MB
python              2.7-alpine          29c1b4c9b149        3 days ago          72MB
```

Notice that your base image, python:2.7-alpine, is also in your list.

# Step 3: Run the Docker image

Now that you have built the image, you can run it to see that it works.

1. Run the Docker image
```sh
$ docker run -p 5001:5000 -d hello-world
d47d0c97f5d8df5e5f978fbfde12094c6611a8fd4286897804ea8b8dce3c724e
```

The `-p` flag maps a port running inside the container to your host. In this case, we are mapping the python app running on port 5000 inside the container, to port 5001 on your host. Note that if port 5001 is already in use by another application on your host, you may have to replace 5001 with another value, such as 5002.

2. Navigate to http://localhost:5001 in a browser to see the results

You should see "Hello World" on your browser.

3. Check the log output of the container.

If you want to see logs from your application you can use the `docker container logs` command. By default, `docker container logs` prints out what is sent to standard out by your application.
```sh
$ docker container logs [container id] # Use 'docker container ls' to find the id for your running container
Server running at http://localhost:4000
```

The Dockerfile is how you create reproducible builds for your application. A common workflow is to have your CI/CD automation run `docker build` as part of its build process. Once images are built, they will be sent to a central registry, where it can be accessed by all environments (such as a test environment) that need to run instances of that application. In the next step, we will push are custom image to the docker hub, where it can be consumed by other developers and operators.

# Step 4: Push to a central registry

1. Navigate to https://hub.docker.com and create an account if you haven't already

For this lab we will be using the docker hub as out central registry. Docker hub is a free service to store publicly available images, or you can pay to store private images. Go to the [DockerHub](https://hub.docker.com) website and create a free account.

Most organizations that use docker heavily will set up their own registry internally. To simplify things, we will be using the Docker Hub, but the following steps and commands apply to any registry.

2. Tag your image with your username

The Docker Hub naming convention is to tag your image with [dockerhub username]/[image name]. To do this, we are going to tag our previously created image `hello-world` to fit that format.

```sh
$ docker tag hello-world [your dockerhub username]/hello-world
```

3. Push your image to the registry

Once we have a properly tagged image, we can use the `docker push` command to push our image to the Docker Hub registry.

```sh
$ docker push [your dockerhub username]/hello-world
The push refers to a repository [docker.io/jzaccone/hello-world]
b306d469a1f0: Pushed 
e8d05a943765: Pushed 
60229a7f5055: Mounted from library/node 
149b98a4e8a0: Mounted from library/node 
9f8566ee5135: Mounted from library/node 
latest: digest: sha256:c99e5eba484d51868fc1be55f3a85cdb290c633618469862649ab9dce5bd1927 size: 1366
```

4. Check out your image on docker hub in your browser

Navigate to https://hub.docker.com and go to your profile to see your newly uploaded image.

Now that your image is on Docker Hub, other developers and operations can use the `docker pull` command to deploy your image to other environments. 

# Step 5: Deploying a Change
The "Hello World" application is over-rated, let's update the app so that it says "Hello Beautiful World!" instead.

1. Update `app.py`

Replace the string "Hello World" with "Hello Beautiful World!" in `app.py`. Your file should have the following contents:

```python
from flask import Flask

app = Flask(__name__)

@app.route("/")
def hello():
    return "Hello Beautiful World!"


if __name__ == "__main__":
    app.run(host='0.0.0.0')
```

2. Rebuild and push your image

Now that your app is updated, you need repeat the steps above to rebuild your app and push it to the Docker Hub registry.

First rebuild, this time use your Docker Hub username in the build command.:


```sh
$ docker build -t [dockerhub username]/hello-world .
Sending build context to Docker daemon  33.28kB
Step 1/4 : FROM python:2.7-alpine
 ---> 29c1b4c9b149
Step 2/4 : RUN pip install flask
 ---> Using cache
 ---> 95c3d46bc3da
Step 3/4 : CMD python app.py
 ---> Using cache
 ---> 775e4a13b353
Step 4/4 : COPY app.py /app.py
 ---> 32ecbd8b73f2
Removing intermediate container c7f5e1c62338
Successfully built 32ecbd8b73f2
Successfully tagged jzaccone/hello-world:latest
```

```sh
$ docker push [dockerhub username]/hello-world
The push refers to a repository [docker.io/jzaccone/hello-world]
cc3cdc4631a0: Pushed 
91c3b6a99b35: Layer already exists 
ea6fb20ac5c0: Layer already exists 
24b5c72b5972: Layer already exists 
590266e37bf8: Layer already exists 
ba2cc2690e31: Layer already exists 
latest: digest: sha256:0c281ad8e259ce1edc3ad67163c08c93039c9916a690da72c0f4aa79cba8163c size: 1579
```

Notice that for both `docker build` and `docker push`, only the last layer of the image is built/pushed. This is because we optimized our docker file so that our `COPY app.py /app.py` line was last (thus reusing the previous layers of the image). This comes in handy for CI/CD processes, where you want your automation to run as fast as possible.

# Step 6: Clean up

Completing this lab results in a bunch of running containers on your host. Let's clean these up.

1. Run `docker container stop [container id]` for each container that is running

First get a list of the containers running using `docker container ls`.
```sh
$ docker container ls
CONTAINER ID        IMAGE               COMMAND                  CREATED              STATUS              PORTS               NAMES
7872fd96ea46        nginx               "nginx -g 'daemon ..."   About a minute ago   Up About a minute   80/tcp              laughing_northcutt
60abd5ee65b1        nginx               "nginx -g 'daemon ..."   About a minute ago   Up About a minute   80/tcp              xenodochial_johnson
31617fdd8e5f        nginx               "nginx -g 'daemon ..."   About a minute ago   Up About a minute   80/tcp              elegant_thompson
5e1bf0e6b926        nginx               "nginx -g 'daemon ..."   About a minute ago   Up About a minute   80/tcp              kind_stonebraker
```
Then run `docker container stop [container id]` for each container in the list.
```sh
$ docker container stop 787 60a 316 5e1
787
60a
316
5e1
```

2. Remove the stopped containers

`docker system prune` is a really handy command to clean up your system. It will remove any stopped containers, unused volumes and networks, and dangling images.

```sh
$ docker system prune
WARNING! This will remove:
        - all stopped containers
        - all volumes not used by at least one container
        - all networks not used by at least one container
        - all dangling images
Are you sure you want to continue? [y/N] y
Deleted Containers:
7872fd96ea4695795c41150a06067d605f69702dbcb9ce49492c9029f0e1b44b
60abd5ee65b1e2732ddc02b971a86e22de1c1c446dab165462a08b037ef7835c
31617fdd8e5f584c51ce182757e24a1c9620257027665c20be75aa3ab6591740
5e1bf0e6b926bd73a66f98b3cbe23d04189c16a43d55dd46b8486359f6fdf048
4fb6b0acc681bf72dfd81e81c5945d7e430d1957494bff2dbaa5561f312b087d

Total reclaimed space: 12B
```

# Summary

In this lab, you started adding value with your own custom docker containers. You built an image using a Dockerfile, and pushed that image to a central registry. You learned how to integrate Docker into a CI/CD pipeline by creating Dockerfiles for reproducible builds, and by using the registry to make images available from other environments. You learned about the layered caching mechanism of Docker and why the order of Dockerfile lines makes a big difference when building fast CI/CD pipeline.
