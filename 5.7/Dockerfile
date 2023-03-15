#
# NOTE: THIS DOCKERFILE IS GENERATED VIA "apply-templates.sh"
#
# PLEASE DO NOT EDIT IT DIRECTLY.
#

FROM debian:bullseye-slim as base

ENV GOSU_VERSION 1.13

RUN set -eux; \
	apt-get update && \
	apt-get install -y --no-install-recommends \
	ca-certificates \
	wget \
	tzdata \
	gnupg \
	dirmngr \
	pwgen \
	openssl \
	libncurses5 \
	bison \
	manpages \
	libldap-2.4-2 \
	libedit2 \
	libaio1 \
	perl \
	curl \
	xz-utils; \
	rm -rf /var/lib/apt/lists/*; \
	savedAptMark="$(apt-mark showmanual)"; \
	dpkgArch="$(dpkg --print-architecture | awk -F- '{ print $NF }')"; \
	wget -O /usr/local/bin/gosu "https://github.com/tianon/gosu/releases/download/$GOSU_VERSION/gosu-$dpkgArch"; \
	wget -O /usr/local/bin/gosu.asc "https://github.com/tianon/gosu/releases/download/$GOSU_VERSION/gosu-$dpkgArch.asc"; \
	export GNUPGHOME="$(mktemp -d)"; \
	gpg --batch --keyserver hkps://keys.openpgp.org --recv-keys B42F6819007F00F88E364FD4036A9C25BF357DD4; \
	gpg --batch --verify /usr/local/bin/gosu.asc /usr/local/bin/gosu; \
	gpgconf --kill all; \
	rm -rf "$GNUPGHOME" /usr/local/bin/gosu.asc; \
	apt-mark auto '.*' > /dev/null; \
	[ -z "$savedAptMark" ] || apt-mark manual $savedAptMark > /dev/null; \
	apt-get purge -y --auto-remove -o APT::AutoRemove::RecommendsImportant=false; \
	chmod +x /usr/local/bin/gosu; \
	gosu --version; \
	gosu nobody true

FROM base as builder

RUN set -ex && \
	apt-get update && apt-get install -y \
	gcc \
	cmake \
	build-essential \
	libssl-dev \
	libncurses5-dev \
	libbison-dev \
	manpages-dev \
	libldap2-dev \
	libedit-dev \
	libaio-dev \
	libmecab-dev \
	libghc-network-bsd-dev \
	libcrypt-dev \
	gzip \
	vis \
	pkg-config \
	wget \
	&& rm -rf /var/lib/apt/lists/*

ENV MYSQL_VERSION 5.7.31

RUN set -ex && \
	curl -LO https://dev.mysql.com/get/Downloads/MySQL-${MYSQL_VERSION}/mysql-${MYSQL_VERSION}.tar.gz && \
	tar -xzf mysql-${MYSQL_VERSION}.tar.gz && \
	cd mysql-${MYSQL_VERSION} && \
	# wget -P /root/mysql-5.7.31/boost/boost_1_59_0/ https://dl.gobuildrun.com/src/boost_1_59_0.tar.gz && \
	cmake . \
	-DBUILD_CONFIG=mysql_release \
	-DCMAKE_BUILD_TYPE=Release \
	-DCMAKE_INSTALL_PREFIX="/usr/share" \
	-DMYSQL_DATADIR=/var/lib/mysql \
	-DINSTALL_MYSQLSHAREDIR=mysql \
	-DINSTALL_MYSQLKEYRINGDIR=/var/lib/mysql-keyring \
	-DSYSCONFDIR=/etc \
	-DENABLED_LOCAL_INFILE=1 \
	-DENABLE_DTRACE=0 \
	-DDEFAULT_CHARSET=utf8mb4 \
	-DDEFAULT_COLLATION=utf8mb4_general_ci \
	-DWITH_EMBEDDED_SERVER=1 \
	-DDOWNLOAD_BOOST=1 \
	-DWITH_BOOST=/root/mysql-${MYSQL_VERSION}/boost/boost && \
	make -j`nproc` && \
	make install && \
	cd .. && \
	rm -rf mysql-boost-${MYSQL_VERSION}.tar.gz mysql-${MYSQL_VERSION} /root/mysql-${MYSQL_VERSION}/boost/boost

# the "/var/lib/mysql" stuff here is because the mysql-server postinst doesn't have an explicit way to disable the mysql_install_db codepath besides having a database already "configured" (ie, stuff in /var/lib/mysql/mysql)
# also, we set debconf keys to make APT a little quieter

FROM base as app

RUN mkdir -p /docker-entrypoint-initdb.d

# add our user and group first to make sure their IDs get assigned consistently, regardless of whatever dependencies get added
RUN groupadd -r mysql && useradd -r -g mysql mysql

COPY --from=builder /usr/share/bin /usr/share/bin
COPY --from=builder /usr/share/lib /usr/share/lib
COPY --from=builder /usr/share/mysql /usr/share/mysql
COPY --from=builder /usr/share/support-files /usr/share/support-files

COPY mysql /etc/mysql

RUN \
	rm -rf /var/lib/mysql && mkdir -p /var/lib/mysql /var/run/mysqld \
	&& chown -R mysql:mysql /var/lib/mysql /var/run/mysqld \
	# ensure that /var/run/mysqld (used for socket and lock files) is writable regardless of the UID our mysqld instance ends up having at runtime
	&& chmod 777 /var/run/mysqld \
	# comment out a few problematic configuration values
	&& find /etc/mysql/ -name '*.cnf' -print0 \
	| xargs -0 grep -lZE '^(bind-address|log)' \
	| xargs -rt -0 sed -Ei 's/^(bind-address|log)/#&/' \
	# don't reverse lookup hostnames, they are usually another container
	&& echo '[mysqld]\nskip-host-cache\nskip-name-resolve' > /etc/mysql/conf.d/docker.cnf

VOLUME /var/lib/mysql

COPY docker-entrypoint.sh /usr/local/bin/
RUN ln -s /usr/local/bin/docker-entrypoint.sh /entrypoint.sh # backwards compat
RUN chmod a+x /usr/local/bin/docker-entrypoint.sh
ENTRYPOINT ["/usr/local/bin/docker-entrypoint.sh"]

EXPOSE 3306 3306

ENV PATH="/usr/share/bin/:${PATH}"
CMD ["mysqld"]