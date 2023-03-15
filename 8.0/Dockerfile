# Base image
FROM debian:bullseye-slim as base

RUN apt-get update && apt-get install -y --no-install-recommends \
	gnupg \
	dirmngr \
	pwgen \
	openssl \
	perl \
	xz-utils \
	&& rm -rf /var/lib/apt/lists/*

FROM base as builder

RUN set -ex && \
	apt-get update && apt-get install -y \
	gcc \
	bzip2 \
	cmake \
	build-essential \
	libssl-dev \
	libncurses5-dev \
	libbison-dev \
	libevent-dev \
	manpages-dev \
	libldap2-dev \
	libedit-dev \
	libaio-dev \
	libghc-network-bsd-dev \
	libcrypt-dev \
	gzip \
	vis \
	pkg-config \
	wget \
	&& rm -rf /var/lib/apt/lists/*

RUN set -eux; \
	savedAptMark="$(apt-mark showmanual)"; \
	apt-get update; \
	apt-get install -y --no-install-recommends ca-certificates wget tzdata; \
	rm -rf /var/lib/apt/lists/*; \
	dpkgArch="$(dpkg --print-architecture | awk -F- '{ print $NF }')";
RUN mkdir /docker-entrypoint-initdb.d

RUN apt-get update && apt-get install -y --no-install-recommends \
	build-essential \
	&& rm -rf /var/lib/apt/lists/*

# Set environment variables
ENV MYSQL_VERSION 8.0.28

# Install dependencies
RUN apt-get update \
	&& apt-get install -y build-essential cmake curl \
	&& apt-get install -y libncurses5-dev libssl-dev libaio-dev libmecab-dev \
	&& curl -LO https://dev.mysql.com/get/Downloads/MySQL-${MYSQL_VERSION}/mysql-${MYSQL_VERSION}.tar.gz

# Build and install MySQL
RUN set -ex && \
	tar xzf mysql-${MYSQL_VERSION}.tar.gz && \
	cd mysql-${MYSQL_VERSION} && \
	cmake . \
	-DCMAKE_BUILD_TYPE=Release \
	-DBUILD_CONFIG=mysql_release \
	-DFORCE_INSOURCE_BUILD=1 \
	-DCMAKE_INSTALL_PREFIX="/usr/share" \
	-DMYSQL_DATADIR=/var/lib/mysql \
	-DINSTALL_MYSQLSHAREDIR=mysql \
	-DINSTALL_MYSQLKEYRINGDIR=/var/lib/mysql-keyring \
	-DSYSCONFDIR=/etc \
	-DENABLED_LOCAL_INFILE=1 \
	-DDEFAULT_CHARSET=utf8mb4 \
	-DDEFAULT_COLLATION=utf8mb4_general_ci \
	-DDOWNLOAD_BOOST=1 \
	-DWITH_BOOST=/root/mysql-${MYSQL_VERSION}/boost && \
	make -j`nproc` && \
	make install && \
	cd .. && \
	rm -rf mysql-${MYSQL_VERSION}

FROM base as app

# add our user and group first to make sure their IDs get assigned consistently, regardless of whatever dependencies get added
RUN groupadd -r mysql && useradd -r -g mysql mysql

COPY --from=builder /usr/share/bin /usr/share/bin
COPY --from=builder /usr/share/lib /usr/share/lib
COPY --from=builder /usr/share/mysql /usr/share/mysql
COPY --from=builder /usr/share/support-files /usr/share/support-files

COPY config /etc/mysql

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

EXPOSE 3306 33060

ENV PATH="/usr/share/bin/:${PATH}"
CMD ["mysqld"]