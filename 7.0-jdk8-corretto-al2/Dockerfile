FROM amazoncorretto:8-al2-jdk

ENV CATALINA_HOME /usr/local/tomcat
ENV PATH $CATALINA_HOME/bin:$PATH
RUN mkdir -p "$CATALINA_HOME"
WORKDIR $CATALINA_HOME

# let "Tomcat Native" live somewhere isolated
ENV TOMCAT_NATIVE_LIBDIR $CATALINA_HOME/native-jni-lib
ENV LD_LIBRARY_PATH ${LD_LIBRARY_PATH:+$LD_LIBRARY_PATH:}$TOMCAT_NATIVE_LIBDIR

# see https://archive.apache.org/dist/tomcat/tomcat-7/KEYS
ENV GPG_KEYS F22C4FED 86867BA6 E86E29AC 307A10A5 564C17A3 7C037D42 0BECE548 5E763BEC 288584E7 0D811BBE 731FABEE 0D498E23 D63011C7

ENV TOMCAT_MAJOR 7
ENV TOMCAT_VERSION 7.0.109
ENV TOMCAT_SHA512 612e830913bf1401bc9540e2273e465b0ee7ef63750a9969a80f1e9da9edb4888aa621fcc6fa5ba23cff94a40e91eb97e3f969b8064dabd49b2d0ea29e59b57e

RUN set -eux; \
	\
	# http://yum.baseurl.org/wiki/YumDB.html
	if ! command -v yumdb > /dev/null; then \
	yum install -y --setopt=skip_missing_names_on_install=False yum-utils; \
	yumdb set reason dep yum-utils; \
	fi; \
	# a helper function to "yum install" things, but only if they aren't installed (and to set their "reason" to "dep" so "yum autoremove" can purge them for us)
	_yum_install_temporary() { ( set -eu +x; \
	local pkg todo=''; \
	for pkg; do \
	if ! rpm --query "$pkg" > /dev/null 2>&1; then \
	todo="$todo $pkg"; \
	fi; \
	done; \
	if [ -n "$todo" ]; then \
	set -x; \
	yum install -y --setopt=skip_missing_names_on_install=False $todo; \
	yumdb set reason dep $todo; \
	fi; \
	) }; \
	_yum_install_temporary gzip tar; \
	\
	ddist() { \
	local f="$1"; shift; \
	local distFile="$1"; shift; \
	local mvnFile="${1:-}"; \
	local success=; \
	local distUrl=; \
	for distUrl in \
	# https://apache.org/history/mirror-history.html
	"https://dlcdn.apache.org/$distFile" \
	# if the version is outdated, we have to pull from the archive
	"https://archive.apache.org/dist/$distFile" \
	# if all else fails, let's try Maven (https://www.mail-archive.com/users@tomcat.apache.org/msg134940.html; https://mvnrepository.com/artifact/org.apache.tomcat/tomcat; https://repo1.maven.org/maven2/org/apache/tomcat/tomcat/)
	${mvnFile:+"https://repo1.maven.org/maven2/org/apache/tomcat/tomcat/$mvnFile"} \
	; do \
	if curl -fL -o "$f" "$distUrl" && [ -s "$f" ]; then \
	success=1; \
	break; \
	fi; \
	done; \
	[ -n "$success" ]; \
	}; \
	\
	ddist 'tomcat.tar.gz' "tomcat/tomcat-$TOMCAT_MAJOR/v$TOMCAT_VERSION/bin/apache-tomcat-$TOMCAT_VERSION.tar.gz" "$TOMCAT_VERSION/tomcat-$TOMCAT_VERSION.tar.gz"; \
	echo "$TOMCAT_SHA512 *tomcat.tar.gz" | sha512sum --strict --check -; \
	ddist 'tomcat.tar.gz.asc' "tomcat/tomcat-$TOMCAT_MAJOR/v$TOMCAT_VERSION/bin/apache-tomcat-$TOMCAT_VERSION.tar.gz.asc" "$TOMCAT_VERSION/tomcat-$TOMCAT_VERSION.tar.gz.asc"; \
	export GNUPGHOME="$(mktemp -d)"; \
	for key in $GPG_KEYS; do \
	gpg --batch --keyserver keyserver.ubuntu.com --recv-keys "$key"; \
	done; \
	gpg --batch --verify tomcat.tar.gz.asc tomcat.tar.gz; \
	tar -xf tomcat.tar.gz --strip-components=1; \
	rm bin/*.bat; \
	rm tomcat.tar.gz*; \
	rm -rf "$GNUPGHOME"; \
	\
	# https://tomcat.apache.org/tomcat-9.0-doc/security-howto.html#Default_web_applications
	mv webapps webapps.dist; \
	mkdir webapps; \
	# we don't delete them completely because they're frankly a pain to get back for users who do want them, and they're generally tiny (~7MB)
	\
	nativeBuildDir="$(mktemp -d)"; \
	tar -xf bin/tomcat-native.tar.gz -C "$nativeBuildDir" --strip-components=1; \
	_yum_install_temporary \
	apr-devel \
	gcc \
	make \
	openssl11-devel \
	; \
	( \
	export CATALINA_HOME="$PWD"; \
	cd "$nativeBuildDir/native"; \
	aprConfig="$(command -v apr-1-config)"; \
	./configure \
	--libdir="$TOMCAT_NATIVE_LIBDIR" \
	--prefix="$CATALINA_HOME" \
	--with-apr="$aprConfig" \
	--with-java-home="$JAVA_HOME" \
	--with-ssl \
	; \
	nproc="$(nproc)"; \
	make -j "$nproc"; \
	make install; \
	); \
	rm -rf "$nativeBuildDir"; \
	rm bin/tomcat-native.tar.gz; \
	\
	# mark any explicit dependencies as manually installed
	find "$TOMCAT_NATIVE_LIBDIR" -type f -executable -exec ldd '{}' ';' \
	| awk '/=>/ && $(NF-1) != "=>" { print $(NF-1) }' \
	| xargs -rt readlink -e \
	| sort -u \
	| xargs -rt rpm --query --whatprovides \
	| sort -u \
	| tee "$TOMCAT_NATIVE_LIBDIR/.dependencies.txt" \
	| xargs -r yumdb set reason user \
	; \
	\
	# clean up anything added temporarily and not later marked as necessary
	yum autoremove -y; \
	yum clean all; \
	rm -rf /var/cache/yum; \
	\
	# sh removes env vars it doesn't support (ones with periods)
	# https://github.com/docker-library/tomcat/issues/77
	find ./bin/ -name '*.sh' -exec sed -ri 's|^#!/bin/sh$|#!/usr/bin/env bash|' '{}' +; \
	\
	# fix permissions (especially for running as non-root)
	# https://github.com/docker-library/tomcat/issues/35
	chmod -R +rX .; \
	chmod 1777 logs temp work; \
	\
	# smoke test
	catalina.sh version

# verify Tomcat Native is working properly
RUN set -eux; \
	nativeLines="$(catalina.sh configtest 2>&1)"; \
	nativeLines="$(echo "$nativeLines" | grep 'Apache Tomcat Native')"; \
	nativeLines="$(echo "$nativeLines" | sort -u)"; \
	if ! echo "$nativeLines" | grep -E 'INFO: Loaded( APR based)? Apache Tomcat Native library' >&2; then \
	echo >&2 "$nativeLines"; \
	exit 1; \
	fi

EXPOSE 8080
CMD ["catalina.sh", "run"]
