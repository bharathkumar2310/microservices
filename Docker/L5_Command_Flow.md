Ex : 


    docker run hello-world


Beow is what happened when we run the above command
-----------------------------------------------------------------------------------------------------------------
        docker run hello-world
        Unable to find image 'hello-world:latest' locally
        latest: Pulling from library/hello-world
        4f55086f7dd0: Pull complete
        Digest: sha256:5dd0d3e6e255913fc30f90b9f2b1d359cc2cbdb48090cc4b65f1676e203243cc
        Status: Downloaded newer image for hello-world:latest
        
        Hello from Docker!
        This message shows that your installation appears to be working correctly.
        
        To generate this message, Docker took the following steps:
        1. The Docker client contacted the Docker daemon.
           2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
              (amd64)
           3. The Docker daemon created a new container from that image which runs the
              executable that produces the output you are currently reading.
           4. The Docker daemon streamed that output to the Docker client, which sent it
              to your terminal.
        
        To try something more ambitious, you can run an Ubuntu container with:
        $ docker run -it ubuntu bash
        
        Share images, automate workflows, and more with a free Docker ID:
        https://hub.docker.com/
        
        For more examples and ideas, visit:
        https://docs.docker.com/get-started/

-------------------------------------------------------------------------------------------------------------------------------------

    Here we are trying to pul a docker image hello-world to create a container(which is the running instance)
    First it tries to pick from local if not found it pulls from the docker hub registry where popular tools images are present
    Then it prints the output
    The docker image contains all code for the hello world etc
