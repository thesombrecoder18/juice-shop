# Image Jenkins personnalisée pour le TP DevSecOps
# Basée sur Jenkins LTS + client Docker (pour lancer Semgrep / ZAP / Trivy en "sibling containers")
FROM jenkins/jenkins:lts-jdk17

USER root

# Installation du client Docker (Docker-out-of-Docker via le socket de l'hôte)
RUN apt-get update && \
    apt-get install -y ca-certificates curl gnupg lsb-release && \
    install -m 0755 -d /etc/apt/keyrings && \
    curl -fsSL https://download.docker.com/linux/debian/gpg | gpg --dearmor -o /etc/apt/keyrings/docker.gpg && \
    chmod a+r /etc/apt/keyrings/docker.gpg && \
    echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/debian $(lsb_release -cs) stable" \
        > /etc/apt/sources.list.d/docker.list && \
    apt-get update && \
    apt-get install -y docker-ce-cli && \
    rm -rf /var/lib/apt/lists/*

# Le GID du groupe docker de l'hôte est 953 (cf. `getent group docker`).
# On crée un groupe docker avec ce GID et on y ajoute l'utilisateur jenkins
# pour qu'il puisse écrire dans /var/run/docker.sock.
RUN groupadd -g 953 docker || groupmod -g 953 docker ; usermod -aG docker jenkins

USER jenkins

# Pré-installation des plugins indispensables (pipeline, git, github, email)
RUN jenkins-plugin-cli --plugins \
    workflow-aggregator \
    git \
    github \
    github-branch-source \
    email-ext \
    matrix-auth \
    htmlpublisher \
    timestamper \
    ws-cleanup
