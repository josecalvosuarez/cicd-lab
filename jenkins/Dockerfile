FROM jenkins/jenkins:lts-jdk17

USER root

# Install Docker CLI so Jenkins can build and push images via the mounted socket
RUN apt-get update && apt-get install -y \
    docker.io \
    && rm -rf /var/lib/apt/lists/* \
    && usermod -aG docker jenkins

# Pre-install the plugins students will need
COPY plugins.txt /usr/share/jenkins/ref/plugins.txt
RUN jenkins-plugin-cli --plugin-file /usr/share/jenkins/ref/plugins.txt

USER jenkins
