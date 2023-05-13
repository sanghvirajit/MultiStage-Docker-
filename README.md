# MultiStage-Docker

Compact multi-stage Docker images for production-grade Python projects

# Concept

The main idea of this method is to create a virtualenv for your package using a heavy full-powered image (which contains commonly used headers, libraries, compilers, etc. which are only needed during build time), and then copy it into a base image with a Python version of choice. Any build artifacts that are unneeded for runtime won't be copied into the final image, further reducing image size.

# Reasoning

Why so complex? You could just COPY directory with your python project into Docker container, and for the first point of view this seems to be reasonable.

But just copying a directory with a python project can cause several problems:

Generated on different operating system .pyc files can be put into Docker image accidentally. Thus, python would try to rewrite .pyc with correct ones each time when Docker image would be started. If you would run Docker image in read-only mode - your application would break.

Large possibility that you would also pack garbage files: pytest and tox cache, developer's virtualenv and other files, that just increase the size of the resulting image.

The multi-stage virtualenv approach has several advantages:

Only the libraries necessary for runtime will be present in the final image, reducing the final image size by potentially hundreds of MBs.

No unnecessary build artifacts and temporary files from the pip install will be present in the final image, only the resulting virtualenv is copied.

No need for python3-dev in the final stage, the virtualenv can run on python3-minimal.

# Usefull tools

## apt-install

Pretty simple bash script. The main purpose is removing the apt cache and temporary files after installation when you want to install something through apt-get install.

Otherwise, you have to write something like this

<code> apt-get update && \
  apt-get install -y --no-upgrade --auto-remove tcpdump && \
  rm -fr /var/lib/apt/lists /var/lib/cache/* /var/log/* </code>
  
 It might be replaced like this:

``
apt-install tcpdump
``

The ``--no-upgrade`` flag avoids build failures when passing held packages (like cudnn8 installed and held in the cuda base image), e.g. originating from the ``find-libdeps`` command below.

## find-libdeps

A shell script which find binary *.so files and resolve required system package for install library dependencies.

Save required packages

``find-libdeps /usr/share/python3/app > /usr/share/python3/app/pkgdeps.txt``

Install saved packages

``xargs -ra /usr/share/python3/app/pkgdeps.txt apt-install``
