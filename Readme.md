# Installing Jenkins in Docker

from https://www.jenkins.io/doc/book/installing/docker/


### Downloading and running Jenkins in Docker

There are several Docker images of Jenkins available.

Use the recommended official [`jenkins/jenkins` image](https://hub.docker.com/r/jenkins/jenkins/) from the [Docker Hub repository](https://hub.docker.com/). This image contains the current [Long-Term Support (LTS) release of Jenkins](https://www.jenkins.io/download), which is production-ready. However, this image doesn’t contain Docker CLI, and is not bundled with the frequently used Blue Ocean plugins and its features. To use the full power of Jenkins and Docker, you may want to go through the installation process described below.


|  | A new`jenkins/jenkins` image is published each time a new release of Jenkins Docker is published. You can see a list of previously published versions of the `jenkins/jenkins` image on the [tags](https://hub.docker.com/r/jenkins/jenkins/tags/) page. |
| - | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |

### On macOS and Linux

1. Open up a terminal window.
2. Create a [bridge network](https://docs.docker.com/network/bridge/) in Docker using the following [`docker network create`](https://docs.docker.com/engine/reference/commandline/network_create/) command:

   ```
   docker network create jenkins
   ```
3. In order to execute Docker commands inside Jenkins nodes, download and run the `docker:dind` Docker image using the following [`docker run`](https://docs.docker.com/engine/reference/run/) command:

   ```
   docker run \
     --name jenkins-docker \
     --rm \
     --detach \
     --privileged \
     --network jenkins \
     --network-alias docker \
     --env DOCKER_TLS_CERTDIR=/certs \
     --volume jenkins-docker-certs:/certs/client \
     --volume jenkins-data:/var/jenkins_home \
     docker:dind \
     --storage-driver overlay2
   ```


   |    | (*Optional* ) Specifies the Docker container name to use for running the image. By default, Docker generates a unique name for the container.                                                                                                                                 |
   | -- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
   |    | (*Optional* ) Automatically removes the Docker container (the instance of the Docker image) when it is shut down.                                                                                                                                                             |
   |    | (*Optional* ) Runs the Docker container in the background. You can stop this instance by running `docker stop jenkins-docker`.                                                                                                                                                |
   |    | Running Docker in Docker currently requires privileged access to function properly. This requirement may be relaxed with newer Linux kernel versions.                                                                                                                         |
   |    | This corresponds with the network created in the earlier step.                                                                                                                                                                                                                |
   |    | Makes the Docker in Docker container available as the hostname`docker` within the `jenkins` network.                                                                                                                                                                          |
   |    | Enables the use of TLS in the Docker server. Due to the use of a privileged container, this is recommended, though it requires the use of the shared volume described below. This environment variable controls the root directory where Docker TLS certificates are managed. |
   |    | Maps the`/certs/client` directory inside the container to a Docker volume named `jenkins-docker-certs` as created above.                                                                                                                                                      |
   |    | Maps the`/var/jenkins_home` directory inside the container to the Docker volume named `jenkins-data`. This allows for other Docker containers controlled by this Docker container’s Docker daemon to mount data from Jenkins.                                                |
   |    | (*Optional* ) Exposes the Docker daemon port on the host machine. This is useful for executing `docker` commands on the host machine to control this inner Docker daemon.                                                                                                     |
   |    | The`docker:dind` image itself. Download this image before running, by using the command: `docker image pull docker:dind`.                                                                                                                                                     |
   |    | The storage driver for the Docker volume. Refer to the[Docker storage drivers](https://docs.docker.com/storage/storagedriver/select-storage-driver) documentation for supported options.                                                                                      |
   |    | If you have problems copying and pasting the above command snippet, use the annotation-free version below:                                                                                                                                                                    |
   | -- | ------------------------------------------------------------------------------------------------------------                                                                                                                                                                  |


   ```
   docker run --name jenkins-docker --rm --detach \
     --privileged --network jenkins --network-alias docker \
     --env DOCKER_TLS_CERTDIR=/certs \
     --volume jenkins-docker-certs:/certs/client \
     --volume jenkins-data:/var/jenkins_home \
     --publish 2376:2376 \
     docker:dind --storage-driver overlay2
   ```
4. Customize the official Jenkins Docker image, by executing the following two steps:

   1. Create a Dockerfile with the following content:

      ```
      FROM jenkins/jenkins:2.452.2-jdk17
      USER root
      RUN apt-get update && apt-get install -y lsb-release
      RUN curl -fsSLo /usr/share/keyrings/docker-archive-keyring.asc \
        https://download.docker.com/linux/debian/gpg
      RUN echo "deb [arch=$(dpkg --print-architecture) \
        signed-by=/usr/share/keyrings/docker-archive-keyring.asc] \
        https://download.docker.com/linux/debian \
        $(lsb_release -cs) stable" > /etc/apt/sources.list.d/docker.list
      RUN apt-get update && apt-get install -y docker-ce-cli
      USER jenkins
      RUN jenkins-plugin-cli --plugins "blueocean docker-workflow"
      ```
   2. Build a new docker image from this Dockerfile, and assign the image a meaningful name, such as "myjenkins-blueocean:2.452.2-1":

      ```
      docker build -t myjenkins-blueocean:2.452.2-1 .
      ```

      If you have not yet downloaded the official Jenkins Docker image, the above process automatically downloads it for you.
5. Run your own `myjenkins-blueocean:2.452.2-1` image as a container in Docker using the following [`docker run`](https://docs.docker.com/engine/reference/run/) command:

   ```
   docker run \
     --name jenkins-blueocean \
     --restart=on-failure \
     --detach \
     --network jenkins \
     --env DOCKER_HOST=tcp://docker:2376 \
     --env DOCKER_CERT_PATH=/certs/client \
     --env DOCKER_TLS_VERIFY=1 \
     --publish 8082:8080 \
     --publish 50002:50000 \
     --volume jenkins-data:/var/jenkins_home \
     --volume jenkins-docker-certs:/certs/client:ro \
     myjenkins-blueocean:2.452.2-1 
   ```


   |    | (*Optional* ) Specifies the Docker container name for this instance of the Docker image.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         |
   | -- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
   |    | Always restart the container if it stops. If it is manually stopped, it is restarted only when Docker daemon restarts or the container itself is manually restarted.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
   |    | (*Optional* ) Runs the current container in the background, known as "detached" mode, and outputs the container ID. If you do not specify this option, then the running Docker log for this container is displayed in the terminal window.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
   |    | Connects this container to the`jenkins` network previously defined. The Docker daemon is now available to this Jenkins container through the hostname `docker`.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
   |    | Specifies the environment variables used by`docker`, `docker-compose`, and other Docker tools to connect to the Docker daemon from the previous step.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            |
   |    | Maps, or publishes, port 8080 of the current container to port 8080 on the host machine. The first number represents the port on the host, while the last represents the container’s port. For example, to access Jenkins on your host machine through port 49000, enter`-p 49000:8080` for this option.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        |
   |    | (*Optional* ) Maps port 50000 of the current container to port 50000 on the host machine. This is only necessary if you have set up one or more inbound Jenkins agents on other machines, which in turn interact with your `jenkins-blueocean` container, known as the Jenkins "controller". Inbound Jenkins agents communicate with the Jenkins controller through TCP port 50000 by default. You can change this port number on your Jenkins controller through the [Security](https://www.jenkins.io/doc/book/managing/security/) page. For example, if you update the **TCP port for inbound Jenkins agents** of your Jenkins controller to 51000, you need to re-run Jenkins via the `docker run …` command. Specify the "publish" option as follows: the first value is the port number on the machine hosting the Jenkins controller, and the last value matches the changed value on the Jenkins controller, for example,`--publish 52000:51000`. Inbound Jenkins agents communicate with the Jenkins controller on that port (52000 in this example). Note that [WebSocket agents](https://www.jenkins.io/blog/2020/02/02/web-socket/) do not need this configuration. |
   |    | Maps the`/var/jenkins_home` directory in the container to the Docker [volume](https://docs.docker.com/engine/admin/volumes/volumes/) with the name `jenkins-data`. Instead of mapping the `/var/jenkins_home` directory to a Docker volume, you can also map this directory to one on your machine’s local file system. For example, specify the option `--volume $HOME/jenkins:/var/jenkins_home` to map the container’s `/var/jenkins_home` directory to the `jenkins` subdirectory within the `$HOME` directory on your local machine — typically `/Users/<your-username>/jenkins` or `/home/<your-username>/jenkins`. NOTE: If you change the source volume or directory for this, the volume from the `docker:dind` container above needs to be updated to match this.                                                                                                                                                                                                                                                                                                                                                                                                 |
   |    | Maps the`/certs/client` directory to the previously created `jenkins-docker-certs` volume. The client TLS certificates required to connect to the Docker daemon are now available in the path specified by the `DOCKER_CERT_PATH` environment variable.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
   |    | The name of the Docker image, which you built in the previous step.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
   |    | If you have problems copying and pasting the command snippet, use the annotation-free version below:                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
   | -- | ------------------------------------------------------------------------------------------------------                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           |


   ```
   docker run --name jenkins-blueocean --restart=on-failure --detach \
     --network jenkins --env DOCKER_HOST=tcp://docker:2376 \
     --env DOCKER_CERT_PATH=/certs/client --env DOCKER_TLS_VERIFY=1 \
     --publish 8080:8080 --publish 50000:50000 \
     --volume jenkins-data:/var/jenkins_home \
     --volume jenkins-docker-certs:/certs/client:ro \
     myjenkins-blueocean:2.452.2-1
   ```
6. Proceed to the [Post-installation setup wizard](https://www.jenkins.io/doc/book/installing/docker/#setup-wizard).


## Post-installation setup wizard

After downloading, installing and running Jenkins using one of the procedures above (except for installation with Jenkins Operator), the post-installation setup wizard begins.

This setup wizard takes you through a few quick "one-off" steps to unlock Jenkins, customize it with plugins and create the first administrator user through which you can continue accessing Jenkins.

### Unlocking Jenkins

When you first access a new Jenkins instance, you are asked to unlock it using an automatically-generated password.

1. Browse to `http://localhost:8080` (or whichever port you configured for Jenkins when installing it) and wait until the **Unlock Jenkins** page appears.
   ![Unlock Jenkins page](https://www.jenkins.io/doc/book/resources/tutorials/setup-jenkins-01-unlock-jenkins-page.jpg)
2. From the Jenkins console log output, copy the automatically-generated alphanumeric password (between the 2 sets of asterisks).
   ![Copying initial admin password](https://www.jenkins.io/doc/book/resources/tutorials/setup-jenkins-02-copying-initial-admin-password.png)
   **Note:**

   * The command: `sudo cat /var/lib/jenkins/secrets/initialAdminPassword` will print the password at console.
   * If you are running Jenkins in Docker using the official `jenkins/jenkins` image you can use `sudo docker exec ${CONTAINER_ID or CONTAINER_NAME} cat /var/jenkins_home/secrets/initialAdminPassword` to print the password in the console without having to exec into the container.
3. On the **Unlock Jenkins** page, paste this password into the **Administrator password** field and click **Continue**.
   **Note:**

   * The Jenkins console log indicates the location (in the Jenkins home directory) where this password can also be obtained. This password must be entered in the setup wizard on new Jenkins installations before you can access Jenkins’s main UI. This password also serves as the default administrator account’s password (with username "admin") if you happen to skip the subsequent user-creation step in the setup wizard.

### Customizing Jenkins with plugins

After [unlocking Jenkins](https://www.jenkins.io/doc/book/installing/docker/#unlocking-jenkins), the **Customize Jenkins** page appears. Here you can install any number of useful plugins as part of your initial setup.

Click one of the two options shown:

* **Install suggested plugins** - to install the recommended set of plugins, which are based on most common use cases.
* **Select plugins to install** - to choose which set of plugins to initially install. When you first access the plugin selection page, the suggested plugins are selected by default.


|  | If you are not sure what plugins you need, choose **Install suggested plugins**. You can install (or remove) additional Jenkins plugins at a later point in time via the [**Manage Jenkins**](https://www.jenkins.io/doc/book/managing) > [**Plugins**](https://www.jenkins.io/doc/book/managing/plugins/) page in Jenkins. |
| - | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |

The setup wizard shows the progression of Jenkins being configured and your chosen set of Jenkins plugins being installed. This process may take a few minutes.

### Creating the first administrator user

Finally, after [customizing Jenkins with plugins](https://www.jenkins.io/doc/book/installing/docker/#customizing-jenkins-with-plugins), Jenkins asks you to create your first administrator user.

1. When the **Create First Admin User** page appears, specify the details for your administrator user in the respective fields and click **Save and Finish**.
2. When the **Jenkins is ready** page appears, click **Start using Jenkins**.
   **Notes:**
   * This page may indicate **Jenkins is almost ready!** instead and if so, click **Restart**.
   * If the page does not automatically refresh after a minute, use your web browser to refresh the page manually.
3. If required, log in to Jenkins with the credentials of the user you just created and you are ready to start using Jenkins!
