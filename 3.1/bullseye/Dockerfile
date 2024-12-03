#
# NOTE: THIS DOCKERFILE IS GENERATED VIA "apply-templates.sh"
#
# PLEASE DO NOT EDIT IT DIRECTLY.
#
FROM buildpack-deps:bullseye

RUN set -eux; \
	apt-get update; \
	apt-get install -y --no-install-recommends \
		libjemalloc2 \
	; \
	rm -rf /var/lib/apt/lists/*

# skip installing gem documentation
RUN set -eux; \
	mkdir -p /usr/local/etc; \
	{ \
		echo 'install: --no-document'; \
		echo 'update: --no-document'; \
	} >> /usr/local/etc/gemrc

ENV LANG C.UTF-8

# https://www.ruby-lang.org/en/news/2024/05/29/ruby-3-1-6-released/
ENV RUBY_VERSION 3.1.6
ENV RUBY_DOWNLOAD_URL https://cache.ruby-lang.org/pub/ruby/3.1/ruby-3.1.6.tar.xz
ENV RUBY_DOWNLOAD_SHA256 597bd1849f252d8a6863cb5d38014ac54152b508c36dca156f6356a9e63c6102

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
		patchelf \
	; \
	rm -rf /var/lib/apt/lists/*; \
	\
	wget -O ruby.tar.xz "$RUBY_DOWNLOAD_URL"; \
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
	# add ruby 3.1 jemalloc patch
	wget -O 'ruby_31_jemalloc.patch' 'https://raw.githubusercontent.com/fullstaq-ruby/server-edition/main/resources/ruby_31_jemalloc.patch'; \
	patch -p1 -ti ruby_31_jemalloc.patch; \
	rm ruby_31_jemalloc.patch; \
	./configure \
		--build="$gnuArch" \
		--disable-install-doc \
		--enable-shared \
	; \
	make -j "$(nproc)"; \
	make install; \
	patchelf --add-needed libjemalloc.so.2 /usr/local/bin/ruby; \
	strip --strip-all /usr/local/bin/ruby; \
	\
	apt-mark auto '.*' > /dev/null; \
	apt-mark manual $savedAptMark > /dev/null; \
	find /usr/local -type f -executable -not \( -name '*tkinter*' \) -exec ldd '{}' ';' \
		| awk '/=>/ { so = $(NF-1); if (index(so, "/usr/local/") == 1) { next }; gsub("^/(usr/)?", "", so); printf "*%s\n", so }' \
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
# verify we have no "ruby" packages installed
	if dpkg -l | grep -i ruby; then exit 1; fi; \
	[ "$(command -v ruby)" = '/usr/local/bin/ruby' ]; \
# rough smoke test
	ruby --version; \
	gem --version; \
	bundle --version

# Sanity check for jemalloc
RUN ruby -e 'abort "jemalloc not enabled" if `MALLOC_CONF=stats_print:true ruby -e "exit" 2>&1` == ""'

ENV MALLOC_CONF dirty_decay_ms:1000,narenas:2,background_thread:true
# don't create ".bundle" in all our apps
ENV GEM_HOME /usr/local/bundle
ENV BUNDLE_SILENCE_ROOT_WARNING=1 \
	BUNDLE_APP_CONFIG="$GEM_HOME"
ENV PATH $GEM_HOME/bin:$PATH
RUN set -eux; \
	mkdir "$GEM_HOME"; \
# adjust permissions of GEM_HOME for running "gem install" as an arbitrary user
	chmod 1777 "$GEM_HOME"

CMD [ "irb" ]
