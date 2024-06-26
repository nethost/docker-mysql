#
# NOTE: THIS DOCKERFILE IS GENERATED VIA "apply-templates.sh"
#
# PLEASE DO NOT EDIT IT DIRECTLY.
#

FROM oraclelinux:7-slim AS builder

RUN set -eux; \
	groupadd --system --gid 999 mysql; \
	useradd --system --uid 999 --gid 999 --home-dir /var/lib/mysql --no-create-home mysql

# add gosu for easy step-down from root
# https://github.com/tianon/gosu/releases
ENV GOSU_VERSION 1.16
RUN set -eux; \
# TODO find a better userspace architecture detection method than querying the kernel
	arch="$(uname -m)"; \
	case "$arch" in \
		aarch64) gosuArch='arm64' ;; \
		x86_64) gosuArch='amd64' ;; \
		*) echo >&2 "error: unsupported architecture: '$arch'"; exit 1 ;; \
	esac; \
	curl -fL -o /usr/local/bin/gosu.asc "https://github.com/tianon/gosu/releases/download/$GOSU_VERSION/gosu-$gosuArch.asc"; \
	curl -fL -o /usr/local/bin/gosu "https://github.com/tianon/gosu/releases/download/$GOSU_VERSION/gosu-$gosuArch"; \
	export GNUPGHOME="$(mktemp -d)"; \
	gpg --batch --keyserver hkps://keys.openpgp.org --recv-keys B42F6819007F00F88E364FD4036A9C25BF357DD4; \
	gpg --batch --verify /usr/local/bin/gosu.asc /usr/local/bin/gosu; \
	rm -rf "$GNUPGHOME" /usr/local/bin/gosu.asc; \
	chmod +x /usr/local/bin/gosu; \
	gosu --version; \
	gosu nobody true

RUN set -eux; \
# https://github.com/docker-library/mysql/pull/871#issuecomment-1167954236
	yum install -y --setopt=skip_missing_names_on_install=False oracle-epel-release-el7; \
	yum install -y --setopt=skip_missing_names_on_install=False \
		bzip2 \
		gzip \
		openssl \
		xz \
		zstd \
	; \
	yum clean all

RUN set -eux; \
# https://dev.mysql.com/doc/refman/8.0/en/checking-gpg-signature.html
# gpg: key 3A79BD29: public key "MySQL Release Engineering <mysql-build@oss.oracle.com>" imported
	key='859BE8D7C586F538430B19C2467B942D3A79BD29'; \
	export GNUPGHOME="$(mktemp -d)"; \
	gpg --batch --keyserver keyserver.ubuntu.com --recv-keys "$key"; \
	gpg --batch --export --armor "$key" > /etc/pki/rpm-gpg/RPM-GPG-KEY-mysql; \
	rm -rf "$GNUPGHOME"

ENV MYSQL_MAJOR 5.7
ENV MYSQL_VERSION 5.7.44-1.el7

RUN set -eu; \
	. /etc/os-release; \
	{ \
		echo '[mysql5.7-server-minimal]'; \
		echo 'name=MySQL 5.7 Server Minimal'; \
		echo 'enabled=1'; \
		echo "baseurl=https://repo.mysql.com/yum/mysql-5.7-community/docker/el/${VERSION_ID%%[.-]*}/\$basearch/"; \
		echo 'gpgcheck=1'; \
		echo 'gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-mysql'; \
# https://github.com/docker-library/mysql/pull/680#issuecomment-825930524
		echo 'module_hotfixes=true'; \
	} | tee /etc/yum.repos.d/mysql-community-minimal.repo

RUN set -eux; \
	yum install -y --setopt=skip_missing_names_on_install=False "mysql-community-server-minimal-$MYSQL_VERSION"; \
	yum clean all; \
# the "socket" value in the Oracle packages is set to "/var/lib/mysql" which isn't a great place for the socket (we want it in "/var/run/mysqld" instead)
# https://github.com/docker-library/mysql/pull/680#issuecomment-636121520
	grep -F 'socket=/var/lib/mysql/mysql.sock' /etc/my.cnf; \
	sed -i 's!^socket=.*!socket=/var/run/mysqld/mysqld.sock!' /etc/my.cnf; \
	grep -F 'socket=/var/run/mysqld/mysqld.sock' /etc/my.cnf; \
	{ echo '[client]'; echo 'socket=/var/run/mysqld/mysqld.sock'; } >> /etc/my.cnf; \
	\
# make sure users dumping files in "/etc/mysql/conf.d" still works
	! grep -F '!includedir' /etc/my.cnf; \
	{ echo; echo '!includedir /etc/mysql/conf.d/'; } >> /etc/my.cnf; \
	mkdir -p /etc/mysql/conf.d; \
# 5.7 Debian-based images also included "/etc/mysql/mysql.conf.d" so let's include it too
	{ echo '!includedir /etc/mysql/mysql.conf.d/'; } >> /etc/my.cnf; \
	mkdir -p /etc/mysql/mysql.conf.d; \
	\
# comment out a few problematic configuration values
	find /etc/my.cnf /etc/mysql/ -name '*.cnf' -print0 \
		| xargs -0 grep -lZE '^(bind-address|log)' \
		| xargs -rt -0 sed -Ei 's/^(bind-address|log)/#&/'; \
	\
# ensure these directories exist and have useful permissions
# the rpm package has different opinions on the mode of `/var/run/mysqld`, so this needs to be after install
	mkdir -p /var/lib/mysql /var/run/mysqld; \
	chown mysql:mysql /var/lib/mysql /var/run/mysqld; \
# ensure that /var/run/mysqld (used for socket and lock files) is writable regardless of the UID our mysqld instance ends up having at runtime
	chmod 1777 /var/lib/mysql /var/run/mysqld; \
	\
	mkdir /docker-entrypoint-initdb.d; \
	\
	mysqld --version; \
	mysql --version

RUN set -eu; \
	. /etc/os-release; \
	{ \
		echo '[mysql-tools-community]'; \
		echo 'name=MySQL Tools Community'; \
		echo "baseurl=https://repo.mysql.com/yum/mysql-tools-community/el/${VERSION_ID%%[.-]*}/\$basearch/"; \
		echo 'enabled=1'; \
		echo 'gpgcheck=1'; \
		echo 'gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-mysql'; \
# https://github.com/docker-library/mysql/pull/680#issuecomment-825930524
		echo 'module_hotfixes=true'; \
	} | tee /etc/yum.repos.d/mysql-community-tools.repo
ENV MYSQL_SHELL_VERSION 8.0.35-1.el7
RUN set -eux; \
	yum install -y --setopt=skip_missing_names_on_install=False "mysql-shell-$MYSQL_SHELL_VERSION"; \
	yum clean all; \
	\
	mysqlsh --version

VOLUME /var/lib/mysql

COPY docker-entrypoint.sh /usr/local/bin/
RUN ln -s usr/local/bin/docker-entrypoint.sh /entrypoint.sh # backwards compat
ENTRYPOINT ["docker-entrypoint.sh"]

EXPOSE 3306 33060
CMD ["mysqld"]
