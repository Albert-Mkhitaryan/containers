# containers

## Definitions of Container From Docker
A container is a standard unit of software that packages up code and all its dependencies so the application runs quickly and reliably from one computing environment to another. A Docker container image is a lightweight, standalone, executable package of software that includes everything needed to run an application: code, runtime, system tools, system libraries and settings

[Docker](https://www.docker.com/resources/what-container/)
## Well Known Picture
![Well Known Picture](images/docker.webp?raw=true "Container VS VM")

## What Does Docker engine doing

![Well Known Picture](images/docker1.jpeg?raw=true "Docker Engine")
# Lets create container using Linux features
Containers are a technology that allows us to run the process in an independent/isolated environment with other processes on the same computer

### The containers are built from a few features of the Linux kernel, and two main features are 
* [Namespaces](https://man7.org/linux/man-pages/man7/namespaces.7.html): Provides isolation
* [Cgroups](https://man7.org/linux/man-pages/man7/cgroups.7.html): Provides limitation

## Namespaces
The main feature of Namespaces is to makes our process completely separated/isolated from the other processes. 
 * pid: provides process isolation
 * net: The networking namespace allows us to run the program on any port without conflict with other processes running on the same computer.
 * mnt: The mount namespace is used to isolate mount points such that processes in different namespaces cannot view each others files
 * ipc : provides separation of shared memory segments
 * uts : Unix Timesharing System namespace allows processes to have own domain name and hostname
 
 ```
 lsns
 ```

For the first we need to have a filesystem, let's extract the ubuntu docker image filesystem end use it
```
mkdir -p /opt/containers/ubuntu/rootfs
cd /opt/containers
docker pull ubuntu
docker run --name test-ubuntu ubuntu
docker export test-ubuntu | tar -C ubuntu/rootfs/ -xf-
ls -l ubuntu/rootfs
```
Now we are going to use Linux namespaces to create our container. For that ushare command will be used.

```
unshare --mount --uts --ipc --net --pid --fork bash
```

Now we have isolated process, let's check for example UTS namespace
```
hostname
hostname my-test-container
exec bash 
hostname
```
Now open new terminal on the same machine and check the hostname
```
hostname
```
now we are going to check PID namespace
```
ps
kill -9 <pid of unshare>
ps -ef 
mount -t proc proc /proc
ps 
ps -ef 
umount /proc
```
Now we will use [pivot_root](https://man7.org/linux/man-pages/man2/pivot_root.2.html) to change our root directory
```
mkdir /opt/containers/ubuntu/rootfs/oldroot/
mkdir /my-ubuntu-container
mount --bind /opt/containers/ubuntu/rootfs/ /my-ubuntu-container
cd /my-ubuntu-container 
ls -l 
pivot_root . oldroot
cd /
ls -l
ls -l /oldroot
```
Fixing our mount points
```
mount -t proc proc /proc
mount 
umount -l /oldroot
mount 
ls -l 
ls -l /oldroot
```
 


 ## Cgroups
 Cgroups allow you to allocate resources — such as CPU time, system memory, network bandwidth, or combinations of these resources — among user-defined groups of tasks (processes) running on a system. 
```
cd /sys/fs/cgroup/memory
ls -l docker
docker run -m 512m -d --name my_nginx nginx
ls -l docker
docker inspect my_nginx | grep Id 
cgget -n -r memory.limit_in_bytes /docker/<id>
```

 ## Overlay filesystem
 https://www.kernel.org/doc/html/latest/filesystems/overlayfs.html

 Overlay filesystems (also called union filesystems) allow creating a union of two or more directories: a list of lower directories and an upper directory. The lower directories of the filesystem are read only, whereas the upper directory can be used for both reads and writes

 ```
 cd /opt/containers/
 mkdir overlay
 mkdir overlay/mount overlay/layer1 overlay/layer2 overlay/layer3 overlay/layer4 workdir
 ls -l overlay
 echo "layer1 file" > ./overlay/layer1/file1
 echo "layer2 file" > ./overlay/layer2/file2
 echo "layer3 file" > ./overlay/layer3/file3
 tree overlay
 cat ./overlay/layer2/file2
 mount -t overlay overlay \
-o lowerdir=/opt/containers/overlay/layer1:/opt/containers/overlay/layer2:/opt/containers/overlay/layer3,upperdir=/opt/containers/overlay/layer4,workdir=/opt/containers/overlay/workdir \
/opt/containers/overlay/mount

tree overlay/mount/
cd overlay
layer2 file
echo "new file" > mount/new_file
tree
rm -rf mount/file3
tree
ls -l layer4/
```
## What's a docker Image?
A docker image basically is a tar file with a root file system and some metadata. 

Everyone may know that every line in a Dockerfile creates a new layer. For example the following snippet we will end up with an image with 2 additional layers.
```
FROM alpine
COPY . /app
ADD my_fole /
CMD ["/my_app"]
```
When we run "docker build" Docker downloads the tarballs from which consist of our image, then unpacks each layer into a separate directory. While building new image additional layers will be created for each COPY, ADD, RUN command. 
When running a container Docker with the help of the Overlay filesystem combines all layers together with an empty upper directory, in which our container will write its changes.
When the container removed, Docker will perform cleans up on the upper folder - our changes on the container will not persist.



