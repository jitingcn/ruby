#
# NOTE: THIS DOCKERFILE IS GENERATED VIA "apply-templates.sh"
#
# PLEASE DO NOT EDIT IT DIRECTLY.
#
FROM buildpack-deps:bookworm

# skip installing gem documentation
RUN set -eux; \
	mkdir -p /usr/local/etc; \
	{ \
		echo 'install: --no-document'; \
		echo 'update: --no-document'; \
	} >> /usr/local/etc/gemrc

ENV LANG C.UTF-8
ENV RUBY_MAJOR 3.3-rc
ENV RUBY_VERSION 3.3.0-preview3
ENV RUBY_DOWNLOAD_SHA256 f79afcf122dc7d04fe26cfa4436b9c488b21766fc54b0d2dfb2ba41cd0cdd355

ENV jemalloc_pkgver 3.6.0
ENV jemalloc_sha256 e16c2159dd3c81ca2dc3b5c9ef0d43e1f2f45b04548f42db12e7c12d7bdf84fe

# some of ruby's build scripts are written in ruby
#   we purge system ruby later to make sure our final image uses what we just built
RUN set -eux; \
	\
	savedAptMark="$(apt-mark showmanual)"; \
	apt-get update; \
	apt-get install -y --no-install-recommends \
		bison \
		dpkg-dev \
		libgdbm-dev \
		ruby \
	; \
	rm -rf /var/lib/apt/lists/*; \
	\
	rustArch=; \
	dpkgArch="$(dpkg --print-architecture)"; \
	case "$dpkgArch" in \
		'amd64') rustArch='x86_64-unknown-linux-gnu'; rustupUrl='https://static.rust-lang.org/rustup/archive/1.25.1/x86_64-unknown-linux-gnu/rustup-init'; rustupSha256='5cc9ffd1026e82e7fb2eec2121ad71f4b0f044e88bca39207b3f6b769aaa799c' ;; \
		'arm64') rustArch='aarch64-unknown-linux-gnu'; rustupUrl='https://static.rust-lang.org/rustup/archive/1.25.1/aarch64-unknown-linux-gnu/rustup-init'; rustupSha256='e189948e396d47254103a49c987e7fb0e5dd8e34b200aa4481ecc4b8e41fb929' ;; \
	esac; \
	\
	if [ -n "$rustArch" ]; then \
		mkdir -p /tmp/rust; \
		\
		wget -O /tmp/rust/rustup-init "$rustupUrl"; \
		echo "$rustupSha256 */tmp/rust/rustup-init" | sha256sum --check --strict; \
		chmod +x /tmp/rust/rustup-init; \
		\
		export RUSTUP_HOME='/tmp/rust/rustup' CARGO_HOME='/tmp/rust/cargo'; \
		export PATH="$CARGO_HOME/bin:$PATH"; \
		/tmp/rust/rustup-init -y --no-modify-path --profile minimal --default-toolchain '1.66.0' --default-host "$rustArch"; \
		\
		rustc --version; \
		cargo --version; \
	fi; \
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
	# add ruby 3.3-rc(preview3) jemalloc patch
	wget -O 'ruby_33_preview3_jemalloc.patch' 'https://gist.githubusercontent.com/jitingcn/16c492b36575112febe1c82d3e45fe2a/raw/3597dd83ec317cb612d0f1d5bbe2c5a9e1c3475b/ruby_33_preview3_jemalloc.patch'; \
	patch -p1 -ti ruby_33_preview3_jemalloc.patch; \
	rm ruby_33_preview3_jemalloc.patch; \
	./configure \
		--build="$gnuArch" \
		--disable-install-doc \
		--enable-shared \
		--with-jemalloc \
		${rustArch:+--enable-yjit} \
	; \
	make -j "$(nproc)"; \
	make install; \
	strip --strip-all /usr/local/bin/ruby; \
	\
	rm -rf /tmp/rust; \
	apt-mark auto '.*' > /dev/null; \
	apt-mark manual $savedAptMark > /dev/null; \
	find /usr/local -type f -executable -not \( -name '*tkinter*' \) -exec ldd '{}' ';' \
		| awk '/=>/ { so = $(NF-1); if (index(so, "/usr/local/") == 1) { next }; gsub("^/(usr/)?", "", so); print so }' \
		| sort -u \
		| xargs -r dpkg-query --search \
		| cut -d: -f1 \
		| sort -u \
		| xargs -r apt-mark manual \
	; \
	apt-get purge -y --auto-remove -o APT::AutoRemove::RecommendsImportant=false; \
	\
	cd /; \
	rm -r /usr/src/ruby; \
	rm /usr/lib/libjemalloc.a; \
	rm /usr/lib/libjemalloc_pic.a; \
# verify we have no "ruby" packages installed
	if dpkg -l | grep -i ruby; then exit 1; fi; \
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
