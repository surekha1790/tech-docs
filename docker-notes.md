# DOCKER COMMANDS

<li><b>To view docker images: </b></li>
&emsp;&emsp;&emsp;&emsp;&emsp; 
docker images
<li><b> To build and run docker image: </b></li>
&emsp;&emsp;&emsp;&emsp;&emsp;
docker build -t image_name dockerfile_location
<br>&emsp;&emsp;&emsp;&emsp;&emsp; Ex: docker build -t test-image  .
<br>&emsp;&emsp;&emsp;&emsp;&emsp; docker run –rm –name (container_name) (image_name)  (--rm option used to remove container after existing from container)

<li><b>To remove docker images:</b></li>
&emsp;&emsp;&emsp;&emsp;&emsp;docker rmi -f (image id)
<li><b>To view containers list:</b></li>
&emsp;&emsp;&emsp;&emsp;&emsp;docker ps -a  
To login to docker container
docker exec -it (container_name) /bin/bash

<li><b>To remove exited containers:</b></li>
&emsp;&emsp;&emsp;&emsp;&emsp;Get the id from Docker ps -a then docker rm (container_id)
<li><b>To remove all exited containers at a time:</b></li>
&emsp;&emsp;&emsp;&emsp;&emsp;docker rm $(docker ps -a -f -q status=exited)
<li><b>Volumes:</b></li>
&emsp;&emsp;&emsp;&emsp;&emsp;When run a docker image results will be stored in docker container. We cannot access docker container once it is exited.
<li><b>To mount container destination directory in the host machine we need to use volumes</b></li>
&emsp;&emsp;&emsp;&emsp; docker run –name (container_name) -v (host_machine_dir):(container_dir) (gscc_image)

&emsp;&emsp;&emsp;&emsp; Ex: docker run –name test-run -v “$(pwd)”/dir:/tmp test-image
Note: absolute path should be used for volumes.
<li><b>To view container details:</b></li>
docker inspect (container)
<li><b>To push docker images to docker hub:</b></li>

## Example docker file
FROM openjdk:7
<br>COPY (host_dir) (docker_dir)
<br>CMD/ENTRYPOINT (script) (input)


FROM java:8
<br>COPY . /
<br>RUN javac Hello.java
<br>CMD ["java", "Hello"]
