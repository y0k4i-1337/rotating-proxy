FROM alpine

# Install general dependencies 
RUN set -xe; \
    apk add --no-cache \
    build-base \
    openssl \
    bzip2 \
    ca-certificates \
    gmp-dev \
    libffi-dev \
    procps \
    yaml-dev \
    zlib-dev \
    tor \
    netcat-openbsd 

# Install ruby
ENV LANG C.UTF-8
ENV RUBY_MAJOR 2.7
ENV RUBY_VERSION 2.7.4
ENV RUBY_DOWNLOAD_SHA256 2a80824e0ad6100826b69b9890bf55cfc4cf2b61a1e1330fccbcb30c46cef8d7

# some of ruby's build scripts are written in ruby
#   we purge system ruby later to make sure our final image uses what we just built
RUN set -eux; \
	\
# readline-dev vs libedit-dev: https://bugs.ruby-lang.org/issues/11869 and https://github.com/docker-library/ruby/issues/75
	apk add --no-cache --virtual .ruby-builddeps \
		autoconf \
		bison \
		bzip2 \
		bzip2-dev \
		ca-certificates \
		coreutils \
		dpkg-dev dpkg \
		g++ \
		gcc \
		gdbm-dev \
		glib-dev \
		libc-dev \
		libffi-dev \
		libxml2-dev \
		libxslt-dev \
		linux-headers \
		make \
		ncurses-dev \
		openssl \
		openssl-dev \
		patch \
		procps \
		readline-dev \
		ruby \
		tar \
		xz \
		yaml-dev \
		zlib-dev \
	; \
	\
	wget -O ruby.tar.xz "https://cache.ruby-lang.org/pub/ruby/${RUBY_MAJOR%-rc}/ruby-$RUBY_VERSION.tar.xz"; \
	echo "$RUBY_DOWNLOAD_SHA256 *ruby.tar.xz" | sha256sum --check --strict; \
	\
	mkdir -p /usr/src/ruby; \
	tar -xJf ruby.tar.xz -C /usr/src/ruby --strip-components=1; \
	rm ruby.tar.xz; \
	\
	cd /usr/src/ruby; \
	\
# https://github.com/docker-library/ruby/issues/196
# https://bugs.ruby-lang.org/issues/14387#note-13 (patch source)
# https://bugs.ruby-lang.org/issues/14387#note-16 ("Therefore ncopa's patch looks good for me in general." -- only breaks glibc which doesn't matter here)
	wget -O 'thread-stack-fix.patch' 'https://bugs.ruby-lang.org/attachments/download/7081/0001-thread_pthread.c-make-get_main_stack-portable-on-lin.patch'; \
	echo '3ab628a51d92fdf0d2b5835e93564857aea73e0c1de00313864a94a6255cb645 *thread-stack-fix.patch' | sha256sum --check --strict; \
	patch -p1 -i thread-stack-fix.patch; \
	rm thread-stack-fix.patch; \
	\
# the configure script does not detect isnan/isinf as macros
	export ac_cv_func_isnan=yes ac_cv_func_isinf=yes; \
	\
# hack in "ENABLE_PATH_CHECK" disabling to suppress:
#   warning: Insecure world writable dir
	{ \
		echo '#define ENABLE_PATH_CHECK 0'; \
		echo; \
		cat file.c; \
	} > file.c.new; \
	mv file.c.new file.c; \
	\
	autoconf; \
	gnuArch="$(dpkg-architecture --query DEB_BUILD_GNU_TYPE)"; \
	./configure \
		--build="$gnuArch" \
		--disable-install-doc \
		--enable-shared \
	; \
	make -j "$(nproc)"; \
	make install; \
	\
	runDeps="$( \
		scanelf --needed --nobanner --format '%n#p' --recursive /usr/local \
			| tr ',' '\n' \
			| sort -u \
			| awk 'system("[ -e /usr/local/lib/" $1 " ]") == 0 { next } { print "so:" $1 }' \
	)"; \
	apk add --no-network --virtual .ruby-rundeps $runDeps; \
	apk del --no-network .ruby-builddeps; \
	\
	cd /; \
	rm -r /usr/src/ruby; \
# verify we have no "ruby" packages installed
	if \
		apk --no-network list --installed \
			| grep -v '^[.]ruby-rundeps' \
			| grep -i ruby \
	; then \
		exit 1; \
	fi; \
	[ "$(command -v ruby)" = '/usr/local/bin/ruby' ]; \
# rough smoke test
	ruby --version; \
	gem --version; \
	bundle --version

# don't create ".bundle" in all our apps
ENV GEM_HOME /usr/local/bundle
ENV BUNDLE_SILENCE_ROOT_WARNING=1 \
	BUNDLE_APP_CONFIG="$GEM_HOME"
ENV PATH $GEM_HOME/bin:$PATH
# adjust permissions of a few directories for running "gem install" as an arbitrary user
RUN mkdir -p "$GEM_HOME" && chmod 777 "$GEM_HOME"


# Install polipo
ENV POLIPO_VER=v1.1.1
RUN addgroup -S polipo \
    && adduser -S -D -H -s /bin/false -g "Polipo Proxy Service" \
           -G polipo polipo \
    && set -x \
    && apk add --no-cache --virtual .build-deps alpine-sdk \
    && git clone --branch $POLIPO_VER --depth 1 --single-branch \
           https://github.com/binlab/polipo.git /tmp/polipo \
    && cd /tmp/polipo \
    && make \
    && install polipo /usr/local/bin/ \
    && mkdir -p /var/cache/polipo /usr/share/polipo/www \
    && rm -rf /tmp/polipo \
    && apk del .build-deps


# Install haproxy
# roughly, https://git.alpinelinux.org/aports/tree/main/haproxy/haproxy.pre-install?h=3.12-stable
RUN set -eux; \
	addgroup --gid 99 --system haproxy; \
	adduser \
		--disabled-password \
		--home /var/lib/haproxy \
		--ingroup haproxy \
		--no-create-home \
		--system \
		--uid 99 \
		haproxy

ENV HAPROXY_VERSION 1.4.27
ENV HAPROXY_URL https://www.haproxy.org/download/1.4/src/haproxy-1.4.27.tar.gz
ENV HAPROXY_MD5 459b82968791472427326ea009834c66

# see https://sources.debian.net/src/haproxy/jessie/debian/rules/ for some helpful navigation of the possible "make" arguments
RUN set -eux; \
	\
	apk add --no-cache --virtual .build-deps \
		gcc \
		libc-dev \
		linux-headers \
		lua5.3-dev \
		make \
		openssl \
		openssl-dev \
		pcre2-dev \
		readline-dev \
		tar \
		zlib-dev \
	; \
	\
	wget -O haproxy.tar.gz "$HAPROXY_URL"; \
	echo "$HAPROXY_MD5 *haproxy.tar.gz" | md5sum -c; \
	mkdir -p /usr/src/haproxy; \
	tar -xzf haproxy.tar.gz -C /usr/src/haproxy --strip-components=1; \
	rm haproxy.tar.gz; \
	\
	makeOpts=' \
		CFLAGS+="-Wno-address-of-packed-member" \
		TARGET=linux2628 \
		USE_GETADDRINFO=1 \
		USE_LUA=1 LUA_INC=/usr/include/lua5.3 LUA_LIB=/usr/lib/lua5.3 \
		USE_OPENSSL=1 \
		USE_PCRE2=1 USE_PCRE2_JIT=1 \
		USE_ZLIB=1 \
		\
		EXTRA_OBJS=" \
		" \
	'; \
	\
	nproc="$(getconf _NPROCESSORS_ONLN)"; \
	eval "make -C /usr/src/haproxy -j '$nproc' all $makeOpts"; \
	eval "make -C /usr/src/haproxy install-bin $makeOpts"; \
	\
	mkdir -p /usr/local/etc/haproxy; \
	cp -R /usr/src/haproxy/examples/errorfiles /usr/local/etc/haproxy/errors; \
	rm -rf /usr/src/haproxy; \
	\
	runDeps="$( \
		scanelf --needed --nobanner --format '%n#p' --recursive /usr/local \
			| tr ',' '\n' \
			| sort -u \
			| awk 'system("[ -e /usr/local/lib/" $1 " ]") == 0 { next } { print "so:" $1 }' \
	)"; \
	apk add --no-network --virtual .haproxy-rundeps $runDeps; \
	apk del --no-network .build-deps; \
	\
# smoke test
	haproxy -v


ADD start.rb /usr/local/bin/start.rb
ADD newnym.sh /usr/local/bin/newnym.sh
ADD uncachable /etc/polipo/uncachable

RUN gem install excon \
	&& chmod +x /usr/local/bin/start.rb \
	&& chmod +x /usr/local/bin/newnym.sh


CMD /usr/local/bin/start.rb
