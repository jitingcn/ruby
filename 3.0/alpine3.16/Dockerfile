#
# NOTE: THIS DOCKERFILE IS GENERATED VIA "apply-templates.sh"
#
# PLEASE DO NOT EDIT IT DIRECTLY.
#
FROM alpine:3.16

RUN set -eux; \
	apk add --no-cache \
		bzip2 \
		ca-certificates \
		gmp-dev \
		libffi-dev \
		procps \
		yaml-dev \
		zlib-dev \
	;

# skip installing gem documentation
RUN set -eux; \
	mkdir -p /usr/local/etc; \
	{ \
		echo 'install: --no-document'; \
		echo 'update: --no-document'; \
	} >> /usr/local/etc/gemrc

ENV LANG C.UTF-8
ENV RUBY_MAJOR 3.0
ENV RUBY_VERSION 3.0.6
ENV RUBY_DOWNLOAD_SHA256 b5cbee93e62d85cfb2a408c49fa30a74231ae8409c2b3858e5f5ea254d7ddbd1

ENV jemalloc_pkgver 3.6.0
ENV jemalloc_sha256 e16c2159dd3c81ca2dc3b5c9ef0d43e1f2f45b04548f42db12e7c12d7bdf84fe

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
	mkdir /jemalloc-stable && cd /jemalloc-stable && \
		wget "https://github.com/jemalloc/jemalloc/releases/download/$jemalloc_pkgver/jemalloc-$jemalloc_pkgver.tar.bz2" && \
		echo "$jemalloc_sha256 *jemalloc-$jemalloc_pkgver.tar.bz2" | sha256sum -c - && \
		tar -xjf "jemalloc-$jemalloc_pkgver.tar.bz2" && cd "jemalloc-$jemalloc_pkgver"; \
		wget https://raw.githubusercontent.com/fullstaq-labs/fullstaq-ruby-server-edition/main/resources/jemalloc_cxx_fix.patch; \
		patch -t -p1 < jemalloc_cxx_fix.patch && rm jemalloc_cxx_fix.patch; \
		./configure --prefix=/usr --sysconfdir=/etc --localstatedir=/var && \
		make -j "$(nproc)" && make install && \
		strip --strip-debug "/usr/lib/libjemalloc.so" && rm -rf "/usr/lib/pkgconfig"; \
		cd / && rm -rf /jemalloc-stable; \
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
	# fix builds on arm32v6/7 and s390x: https://github.com/docker-library/ruby/issues/308
	# and don't break the other arches: https://github.com/docker-library/ruby/issues/365
	apkArch="$(apk --print-arch)"; \
	case "$apkArch" in \
		s390x | armhf | armv7) \
			apk add --no-cache libucontext-dev; \
			export LIBS='-lucontext'; \
			;; \
	esac; \
	gnuArch="$(dpkg-architecture --query DEB_BUILD_GNU_TYPE)"; \
	./configure \
		--build="$gnuArch" \
		--disable-install-doc \
		--enable-shared \
		--with-jemalloc \
	; \
	make -j "$(nproc)"; \
	make install; \
	strip --strip-all /usr/local/bin/ruby; \
	\
	runDeps="$( \
		scanelf --needed --nobanner --format '%n#p' --recursive /usr/local \
			| tr ',' '\n' \
			| sort -u \
			| awk 'system("[ -e /usr/local/lib/" $1 " ]") == 0 { next } { print "so:" $1 }' \
			| sed s/"so:libjemalloc.so.1"// \
	)"; \
	apk add --no-network --virtual .ruby-rundeps $runDeps; \
	apk del --no-network .ruby-builddeps; \
	\
	cd /; \
	rm -r /usr/src/ruby; \
	rm /usr/lib/libjemalloc.a; \
	rm /usr/lib/libjemalloc_pic.a; \
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

# Sanity check for jemalloc
RUN ruby -r rbconfig -e "abort 'jemalloc not enabled' unless RbConfig::CONFIG['LIBS'].include?('jemalloc') || RbConfig::CONFIG['MAINLIBS'].include?('jemalloc')"; \
	ruby -e 'abort "jemalloc not enabled" if `MALLOC_CONF=stats_print:true ruby -e "exit" 2>&1` == ""'
RUN MALLOC_CONF="stats_print:true" \
	LD_PRELOAD="/usr/lib/libjemalloc.so" \
	ruby -e "exit" 2>&1 | grep "3.6.0-0"

ENV LD_PRELOAD=/usr/lib/libjemalloc.so
# don't create ".bundle" in all our apps
ENV GEM_HOME /usr/local/bundle
ENV BUNDLE_SILENCE_ROOT_WARNING=1 \
	BUNDLE_APP_CONFIG="$GEM_HOME"
ENV PATH $GEM_HOME/bin:$PATH
# adjust permissions of a few directories for running "gem install" as an arbitrary user
RUN mkdir -p "$GEM_HOME" && chmod 1777 "$GEM_HOME"

CMD [ "irb" ]
