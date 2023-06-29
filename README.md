# Podman Local Registry

## Configuration

As root, install some packages

    su -
    dnf -y module install container-tools nano
    yum install -y httpd-tools podman buildah skopeo
    chmod 777 /var/lib/containers/sigstore

Installation of SSL is not done here, not in the registry container, but instead it is done in the Nginx reverse proxy container. Since we will use the HTTP protocol with Nginx for SSL termination, we need to configure the registry to approve insecure traffic.

    cat << 'EOF' > /etc/containers/registries.conf
    [[registry]]
    location = "localhost:5000"
    insecure = true
    EOF

As root, create a new user and become the user

    useradd registry
    passwd registry
    loginctl enable-linger $(id -u registry)
    su - registry

prepare its home directory

    cd ~
    mkdir containers.mikaelhelin.com
    cd containers.mikaelhelin.com
    mkdir -p DATA/opt/registry/{data,auth,certs}

create a user with password, you need to enter the same password twice manually

    htpasswd -Bc DATA/opt/registry/auth/htpasswd mikael

## Installation

Put the file run_pod-registry into the folder /home/registry/containers.mikaelhelin.com and chmod it

    chmod +x run_pod-registry

then become root and install the systemd file

    su -
    cat << 'EOF' > /etc/systemd/system/pod-registry.service
    [Unit]
    Description=Run registry service at boot
    After=network-online.target podman.socket

    [Service]
    User=registry
    Restart=always
    ExecStart=/usr/bin/bash /home/registry/containers.mikaelhelin.com/run_registry.sh
    ExecStop=/usr/bin/rm /home/registry/containers.mikaelhelin.com/.lock

    [Install]
    WantedBy=multi-user.target
    EOF
    systemctl daemon-reload
    systemctl enable pod-registry.service
    reboot

## Testing

We need a convention for image names. It is developer_name/project_name__sub_project_name:tag where tag is for version.

As a first test, here we get the official Debian image and put it into our registry with help of skopeo.

    podman login http://localhost:5000
    skopeo copy docker://docker.io/library/debian:bullseye docker://localhost:5000/debian/debian:bullseye
    skopeo copy docker://docker.io/library/debian:latest docker://localhost:5000/debian/debian:latest

check the contents of the registry

    curl -u mikael http://localhost:5000/v2/_catalog
    curl -u mikael http://localhost:5000/v2/debian/debian/tags/list

