FROM centos:8

hello-world-binary:
    WORKDIR /code
    RUN yum install -y gcc
    COPY hello.c .
    RUN gcc -o hello-world hello.c
    SAVE ARTIFACT hello-world

hello-world-package:
    WORKDIR /package/specs
    RUN yum install -y createrepo rpm-build rpm-sign wget
    COPY hello-world.spec .
    COPY +hello-world-binary/hello-world /root/example/hello-world-program/hello-world
    RUN rpmbuild --target "x86_64" -bb hello-world.spec
    RUN rpm -qpivl --changelog --nomanifest  /root/rpmbuild/RPMS/x86_64/hello-world-1.0.0-1.x86_64.rpm
    SAVE ARTIFACT /root/rpmbuild/RPMS/x86_64/hello-world-1.0.0-1.x86_64.rpm

generate-pgp-key:
    FROM ubuntu:20.04 # pgp keys can be generated anywhere, we will use ubuntu for this.
    WORKDIR /pgp-key
    RUN apt-get update && apt-get install -y gpg
    RUN echo "%echo Generating an example PGP key
Key-Type: RSA
Key-Length: 4096
Name-Real: example
Name-Email: example@example.com
Expire-Date: 0
%no-ask-passphrase
%no-protection
%commit" > example-pgp-key.batch
    RUN gpg --no-tty --batch --gen-key example-pgp-key.batch
    RUN gpg --armor --export example > pgp-key.public
    RUN gpg --armor --export-secret-keys example > pgp-key.private
    SAVE ARTIFACT pgp-key.public
    SAVE ARTIFACT pgp-key.private

create-repo:
    WORKDIR /yum-repo
    RUN yum install -y createrepo rpm-build rpm-sign wget
    COPY +hello-world-package/hello-world-1.0.0-1.x86_64.rpm .

    # import keys
    COPY +generate-pgp-key/pgp-key.private /.
    RUN cat /pgp-key.private | gpg --import
    RUN gpg --with-colons --import-options show-only --import --fingerprint /pgp-key.private | awk -F: '$1 == "fpr" {print $10;}' > /key-id

    RUN echo "%_signature gpg
%_gpg_name $(cat /key-id)" > /root/.rpmmacros

    RUN rpm --addsign *.rpm
    RUN createrepo .
    RUN gpg --detach-sign --armor repodata/repomd.xml

    SAVE ARTIFACT /yum-repo AS LOCAL example-yum-repo

repo-server:
    RUN yum install -y python3
    WORKDIR /www
    COPY +create-repo/yum-repo /www/yum-repo
    COPY +generate-pgp-key/pgp-key.public .
    RUN echo "[example-repo]
name=Example Repo
baseurl=http://127.0.0.1:8000/yum-repo
enabled=1
gpgcheck=1
gpgkey=http://127.0.0.1:8000/pgp-key.public" > example.repo
    CMD ["python3", "-m", "http.server"]


test:
    # TODO update buildkitd/docker-auto-install.sh to handle centos, then move this there.
    RUN yum install -y yum-utils
    RUN yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo && \
        yum install -y docker-ce docker-ce-cli containerd.io
    COPY +generate-pgp-key/pgp-key.public /example.pgp
    COPY docker-compose.yml .
    WITH DOCKER --compose docker-compose.yml --load=repo-server:latest=+repo-server
        RUN \
            yum-config-manager --add-repo http://127.0.0.1:8000/example.repo && \
            yum install -y hello-world && \
            hello-world
    END
