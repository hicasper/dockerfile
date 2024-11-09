FROM redhat/ubi8-minimal

ENV PS_VERSION=5.7.44-48.1
ENV OS_VER=el8
ENV FULL_PERCONA_VERSION="$PS_VERSION.$OS_VER"

# Do not report during Docker image creation.
# Note that doing so, would create telemetry config file
# which would prevent reporting when new container is started.
# If we want to track Docker image creation as well,
# remove telemetry config file after installing packages!
ARG PERCONA_TELEMETRY_DISABLE=1

COPY ps-entry.sh /docker-entrypoint.sh
RUN set -ex; \
    # shadow-utils are needed for user/group manipulation on UBI-based images
    # microdnf -y update; \
    microdnf -y install shadow-utils; \
    groupadd -g 1001 mysql; \
    useradd -u 1001 -r -g 1001 -s /sbin/nologin \
        -c "Default Application User" mysql; \
    export GNUPGHOME="$(mktemp -d)"; \
    gpg --batch --keyserver keyserver.ubuntu.com --recv-keys 4D1BB29D63D98E422B2113B19334A25F8507EFA5 99DB70FAE1D7CE227FB6488205B555B38483C65D; \
    gpg --batch --export --armor 4D1BB29D63D98E422B2113B19334A25F8507EFA5 > ${GNUPGHOME}/PERCONA-PACKAGING-KEY; \
    gpg --batch --export --armor 99DB70FAE1D7CE227FB6488205B555B38483C65D > ${GNUPGHOME}/RPM-GPG-KEY-centosofficial; \
    rpmkeys --import ${GNUPGHOME}/PERCONA-PACKAGING-KEY ${GNUPGHOME}/RPM-GPG-KEY-centosofficial; \
    microdnf install -y findutils; \
    curl -sSL -o /tmp/percona-release.rpm https://repo.percona.com/yum/percona-release-latest.noarch.rpm; \
    rpmkeys --checksig /tmp/percona-release.rpm; \
    rpm -i /tmp/percona-release.rpm; \
    percona-release enable-only ps-57; \
    rm -rf "$GNUPGHOME" /tmp/percona-release.rpm; \
    rpm --import /etc/pki/rpm-gpg/PERCONA-PACKAGING-KEY; \
    # microdnf -y module disable mysql; \
    # curl -Lf -o /tmp/numactl-libs.rpm http://vault.centos.org/centos/8/BaseOS/x86_64/os/Packages/numactl-libs-2.0.12-13.el8.x86_64.rpm; \
    # rpmkeys --checksig /tmp/numactl-libs.rpm; \
    # rpm -i /tmp/numactl-libs.rpm; \
    # rm -rf /tmp/numactl-libs.rpm; \
    rpm -e --nodeps tzdata; \
    microdnf -y install \
        tzdata \
        jemalloc \
        which \
        cracklib-dicts \
        policycoreutils; \
    # repoquery -a --location \
    #     selinux-policy \
    #         | xargs curl -Lf -o /tmp/selinux-policy.rpm; \
    # rpm -iv /tmp/selinux-policy.rpm --nodeps; \
    # rm -rf /tmp/selinux-policy.rpm; \
    microdnf -y install \
        Percona-Server-server-57-${FULL_PERCONA_VERSION}; \
    #    Percona-Server-devel-57-${FULL_PERCONA_VERSION} \
    #    Percona-Server-tokudb-57-${FULL_PERCONA_VERSION} \
    #    Percona-Server-rocksdb-57-${FULL_PERCONA_VERSION}; \
    microdnf clean all; \
    rm -rf /var/cache/dnf /var/cache/yum /var/lib/mysql; \
    # purge and re-create /var/lib/mysql with appropriate ownership
    /usr/bin/install -m 0775 -o mysql -g root -d /var/lib/mysql /var/run/mysqld /docker-entrypoint-initdb.d; \
    # comment out a few problematic configuration values
    find /etc/percona-server.cnf /etc/percona-server.conf.d /etc/my.cnf.d -name '*.cnf' -print0 \
      | xargs -0 grep -lZE '^(bind-address|log|user)' \
      | xargs -rt -0 sed -Ei 's/^(bind-address|log|user)/#&/'; \
    # don't reverse lookup hostnames, they are usually another container
    printf '[mysqld]\nskip-host-cache\nskip-name-resolve\n' > /etc/my.cnf.d/docker.cnf; \
    # TokuDB modifications
    /usr/bin/install -m 0664 -o mysql -g root /dev/null /etc/sysconfig/mysql; \
    echo "LD_PRELOAD=/usr/lib64/libjemalloc.so.1" >> /etc/sysconfig/mysql; \
    echo "THP_SETTING=never" >> /etc/sysconfig/mysql; \
    # keep backward compatibility with debian images
    ln -s /etc/my.cnf.d /etc/mysql; \
    # allow to change config files
    chown -R mysql:root /etc/percona-server.cnf /etc/percona-server.conf.d /etc/my.cnf.d; \
    chmod -R ug+rwX /etc/percona-server.cnf /etc/percona-server.conf.d /etc/my.cnf.d; \
    chmod +x /docker-entrypoint.sh

VOLUME ["/var/lib/mysql", "/var/log/mysql"]
ENTRYPOINT ["/docker-entrypoint.sh"]

USER mysql
EXPOSE 3306
CMD ["mysqld"]
