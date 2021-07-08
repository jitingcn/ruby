FROM alpine:3.12

RUN apk add --no-cache \
		gmp-dev

# skip installing gem documentation
RUN set -eux; \
	mkdir -p /usr/local/etc; \
	{ \
		echo 'install: --no-document'; \
		echo 'update: --no-document'; \
	} >> /usr/local/etc/gemrc

ENV LANG C.UTF-8
ENV RUBY_MAJOR 2.7
ENV RUBY_VERSION 2.7.4
ENV RUBY_DOWNLOAD_SHA256 2a80824e0ad6100826b69b9890bf55cfc4cf2b61a1e1330fccbcb30c46cef8d7

ENV	jemalloc_pkgver 5.2.1
ENV jemalloc_sha512 0bbb77564d767cef0c6fe1b97b705d368ddb360d55596945aea8c3ba5889fbce10479d85ad492c91d987caacdbbdccc706aa3688e321460069f00c05814fae02

# some of ruby's build scripts are written in ruby
#   we purge system ruby later to make sure our final image uses what we just built
# readline-dev vs libedit-dev: https://bugs.ruby-lang.org/issues/11869 and https://github.com/docker-library/ruby/issues/75
RUN set -eux; \
	\
	apk add --no-cache --virtual .ruby-builddeps \
		autoconf \
		bison \
		bzip2 \
		bzip2-dev \
		ca-certificates \
		coreutils \
		dpkg-dev dpkg \
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
	wget -O jemalloc.tar.bz2 "https://github.com/jemalloc/jemalloc/releases/download/$jemalloc_pkgver/jemalloc-$jemalloc_pkgver.tar.bz2"; \
	echo "$jemalloc_sha512 *jemalloc.tar.bz2" | sha512sum -c; \
	mkdir /builder; \
	tar -xjf jemalloc.tar.bz2 -C /builder --strip-components=1; \
	rm jemalloc.tar.bz2; \
	cd /builder; \
	./configure --prefix=/usr --sysconfdir=/etc --localstatedir=/var --disable-syscall; \
	make -j "$(nproc)"; \
	make DESTDIR="/tmp/pkg" install_include; \
	make DESTDIR="/tmp/pkg" install_lib; \
	cp /builder/jemalloc.pc /tmp/pkg; \
	cd /tmp; \
	/usr/bin/install -c -d /usr/include/jemalloc; \
	/usr/bin/install -c -m 644 /tmp/pkg/usr/include/jemalloc/jemalloc.h /usr/include/jemalloc; \
	/usr/bin/install -c -d /usr/lib; \
	/usr/bin/install -c -m 755 /tmp/pkg/usr/lib/libjemalloc.so.2 /usr/lib; \
	ln -sf libjemalloc.so.2 /usr/lib/libjemalloc.so; \
	/usr/bin/install -c -d /usr/lib; \
	/usr/bin/install -c -m 755 /tmp/pkg/usr/lib/libjemalloc.a /usr/lib; \
	/usr/bin/install -c -m 755 /tmp/pkg/usr/lib/libjemalloc_pic.a /usr/lib; \
	/usr/bin/install -c -d /usr/lib/pkgconfig; \
	/usr/bin/install -c -m 644 /tmp/pkg/jemalloc.pc /usr/lib/pkgconfig; \
	rm -rf /tmp/pkg; \
	rm -rf /builder; \
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
# the configure script does not detect isnan/isinf as macros
	export ac_cv_func_isnan=yes ac_cv_func_isinf=yes; \
	./configure \
		--build="$gnuArch" \
		--disable-install-doc \
		--enable-shared \
		--with-jemalloc \
	; \
	make -j "$(nproc)"; \
	make install; \
	\
	runDeps="$( \
		scanelf --needed --nobanner --format '%n#p' --recursive /usr/local \
			| tr ',' '\n' \
			| sort -u \
			| awk 'system("[ -e /usr/local/lib/" $1 " ]") == 0 { next } { print "so:" $1 }' \
			| sed s/"so:libjemalloc.so.2"// \
	)"; \
	apk add --no-network --virtual .ruby-rundeps \
		$runDeps \
		bzip2 \
		ca-certificates \
		libffi-dev \
		procps \
		yaml-dev \
		zlib-dev \
	; \
	apk del --no-network .ruby-builddeps; \
	\
	cd /; \
	rm -r /usr/src/ruby; \
	rm /usr/lib/libjemalloc.a; \
	rm /usr/lib/libjemalloc_pic.a; \
# verify we have no "ruby" packages installed
	! apk --no-network list --installed \
		| grep -v '^[.]ruby-rundeps' \
		| grep -i ruby \
	; \
	[ "$(command -v ruby)" = '/usr/local/bin/ruby' ]; \
# rough smoke test
	ruby --version; \
	gem --version; \
	bundle --version

# Sanity check for jemalloc
RUN ruby -r rbconfig -e "abort 'jemalloc not enabled' unless RbConfig::CONFIG['LIBS'].include?('jemalloc') || RbConfig::CONFIG['MAINLIBS'].include?('jemalloc')"; \
	ruby -e 'abort "jemalloc not enabled" if `MALLOC_CONF=stats_print:true ruby -e "exit" 2>&1` == ""'

# don't create ".bundle" in all our apps
ENV GEM_HOME /usr/local/bundle
ENV BUNDLE_SILENCE_ROOT_WARNING=1 \
	BUNDLE_APP_CONFIG="$GEM_HOME"
ENV PATH $GEM_HOME/bin:$PATH
# adjust permissions of a few directories for running "gem install" as an arbitrary user
RUN mkdir -p "$GEM_HOME" && chmod 777 "$GEM_HOME"

CMD [ "irb" ]