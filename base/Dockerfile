FROM buildpack-deps:xenial

MAINTAINER Kenta Murata <mrkn@mrkn.jp>

USER root

ENV DEBIAN_FRONTEND noninteractive

# Install Tini
RUN wget --quiet https://github.com/krallin/tini/releases/download/v0.10.0/tini \
	&& echo "1361527f39190a7338a0b434bd8c88ff7233ce7b9a4876f3315c22fce7eca1b0 *tini" | sha256sum -c - \
	&& mv tini /usr/local/bin/tini \
	&& chmod +x /usr/local/bin/tini

###############################################################
# Apache Arrow (based on red-datat-tools/red-arrow/Dockerfile)
###############################################################

RUN set -ex \
	&& apt-get update \
	&& apt install -y --no-install-recommends \
		apt-transport-https \
		lsb-release \
	&& { \
		echo "deb https://packages.red-data-tools.org/ubuntu/ $(lsb_release --codename --short) universe"; \
		echo "deb-src https://packages.red-data-tools.org/ubuntu/ $(lsb_release --codename --short) universe"; \
	} >> /etc/apt/sources.list.d/red-data-tools.list \
	&& apt update --allow-insecure-repositories || apt update \
	&& apt install -y --allow-unauthenticated red-data-tools-keyring \
	&& apt update \
	&& apt-get install -y --no-install-recommends \
		libarrow-glib-dev \
		libparquet-glib-dev

###############################################################
# Ruby based on docker-library/ruby
###############################################################

# skip installing gem documentation
RUN mkdir -p /usr/local/etc \
	&& { \
		echo 'install: --no-document'; \
		echo 'update: --no-document'; \
	} >> /usr/local/etc/gemrc

ENV RUBY_MAJOR 2.4
ENV RUBY_VERSION 2.4.2
ENV RUBY_DOWNLOAD_SHA256 748a8980d30141bd1a4124e11745bb105b436fb1890826e0d2b9ea31af27f735
ENV RUBYGEMS_VERSION 2.6.13

# some of ruby's build scripts are written in ruby
#   we purge system ruby later to make sure our final image uses what we just built
RUN set -ex \
	\
	&& buildDeps=' \
		bison \
		libgdbm-dev \
		ruby \
	' \
	&& apt-get update \
	&& apt-get install -y --no-install-recommends $buildDeps \
	&& rm -rf /var/lib/apt/lists/* \
	\
	&& wget -O ruby.tar.xz "https://cache.ruby-lang.org/pub/ruby/${RUBY_MAJOR%-rc}/ruby-$RUBY_VERSION.tar.xz" \
	&& echo "$RUBY_DOWNLOAD_SHA256 *ruby.tar.xz" | sha256sum -c - \
	\
	&& mkdir -p /usr/src/ruby \
	&& tar -xJf ruby.tar.xz -C /usr/src/ruby --strip-components=1 \
	&& rm ruby.tar.xz \
	\
	&& cd /usr/src/ruby \
	\
# hack in "ENABLE_PATH_CHECK" disabling to suppress:
#   warning: Insecure world writable dir
	&& { \
		echo '#define ENABLE_PATH_CHECK 0'; \
		echo; \
		cat file.c; \
	} > file.c.new \
	&& mv file.c.new file.c \
	\
	&& autoconf \
	&& ./configure --disable-install-doc --enable-shared \
	&& make -j"$(nproc)" \
	&& make install \
	\
	&& apt-get purge -y --auto-remove $buildDeps \
	&& cd / \
	&& rm -r /usr/src/ruby \
	\
	&& gem update --system "$RUBYGEMS_VERSION"

ENV BUNDLER_VERSION 1.14.5

RUN gem install bundler --version "$BUNDLER_VERSION"

# install things globally, for great justice
# and don't create ".bundle" in all our apps
ENV GEM_HOME /usr/local/bundle
ENV BUNDLE_PATH="$GEM_HOME" \
	BUNDLE_BIN="$GEM_HOME/bin" \
	BUNDLE_SILENCE_ROOT_WARNING=1 \
	BUNDLE_APP_CONFIG="$GEM_HOME"
ENV PATH $BUNDLE_BIN:$PATH
RUN mkdir -p "$GEM_HOME" "$BUNDLE_BIN" \
	&& chmod 777 "$GEM_HOME" "$BUNDLE_BIN"


###############################################################
# Python based on docker-library/python
###############################################################

# ensure local python is preferred over distribution python
ENV PATH /usr/local/bin:$PATH

# http://bugs.python.org/issue19846
# > At the moment, setting "LANG=C" on a Linux system *fundamentally breaks Python 3*, and that's not OK.
ENV LANG C.UTF-8

# runtime dependencies
RUN apt-get update && apt-get install -y --no-install-recommends \
		tcl \
		tk \
	&& rm -rf /var/lib/apt/lists/*

ENV GPG_KEY 0D96DF4D4110E5C43FBFB17F2D347EA6AA65421D
ENV PYTHON_VERSION 3.6.2

# if this is called "PIP_VERSION", pip explodes with "ValueError: invalid truth value '<VERSION>'"
ENV PYTHON_PIP_VERSION 9.0.1

RUN set -ex \
	&& buildDeps=' \
		tcl-dev \
		tk-dev \
	' \
	&& apt-get update && apt-get install -y $buildDeps --no-install-recommends && rm -rf /var/lib/apt/lists/* \
	\
	&& wget -O python.tar.xz "https://www.python.org/ftp/python/${PYTHON_VERSION%%[a-z]*}/Python-$PYTHON_VERSION.tar.xz" \
	&& wget -O python.tar.xz.asc "https://www.python.org/ftp/python/${PYTHON_VERSION%%[a-z]*}/Python-$PYTHON_VERSION.tar.xz.asc" \
	&& export GNUPGHOME="$(mktemp -d)" \
	&& gpg --keyserver ha.pool.sks-keyservers.net --recv-keys "$GPG_KEY" \
	&& gpg --batch --verify python.tar.xz.asc python.tar.xz \
	&& rm -r "$GNUPGHOME" python.tar.xz.asc \
	&& mkdir -p /usr/src/python \
	&& tar -xJC /usr/src/python --strip-components=1 -f python.tar.xz \
	&& rm python.tar.xz \
	\
	&& cd /usr/src/python \
	&& ./configure \
		--enable-loadable-sqlite-extensions \
		--enable-shared \
	&& make -j$(nproc) \
	&& make install \
	&& ldconfig \
	\
# explicit path to "pip3" to ensure distribution-provided "pip3" cannot interfere
	&& if [ ! -e /usr/local/bin/pip3 ]; then : \
		&& wget -O /tmp/get-pip.py 'https://bootstrap.pypa.io/get-pip.py' \
		&& python3 /tmp/get-pip.py "pip==$PYTHON_PIP_VERSION" \
		&& rm /tmp/get-pip.py \
	; fi \
# we use "--force-reinstall" for the case where the version of pip we're trying to install is the same as the version bundled with Python
# ("Requirement already up-to-date: pip==8.1.2 in /usr/local/lib/python3.6/site-packages")
# https://github.com/docker-library/python/pull/143#issuecomment-241032683
	&& pip3 install --no-cache-dir --upgrade --force-reinstall "pip==$PYTHON_PIP_VERSION" \
# then we use "pip list" to ensure we don't have more than one pip version installed
# https://github.com/docker-library/python/pull/100
	&& [ "$(pip list |tac|tac| awk -F '[ ()]+' '$1 == "pip" { print $2; exit }')" = "$PYTHON_PIP_VERSION" ] \
	\
	&& find /usr/local -depth \
		\( \
			\( -type d -a -name test -o -name tests \) \
			-o \
			\( -type f -a -name '*.pyc' -o -name '*.pyo' \) \
		\) -exec rm -rf '{}' + \
	&& apt-get purge -y --auto-remove $buildDeps \
	&& rm -rf /usr/src/python ~/.cache

# make some useful symlinks that are expected to exist
RUN cd /usr/local/bin \
	&& { [ -e easy_install ] || ln -s easy_install-* easy_install; } \
	&& ln -s idle3 idle \
	&& ln -s pydoc3 pydoc \
	&& ln -s python3 python \
	&& ln -s python3-config python-config


###############################################################
# pycall
###############################################################

RUN set -ex \
	&& echo 'deb http://download.opensuse.org/repositories/home:/zeromq:/git-draft/xUbuntu_16.04/ /' \
		> /etc/apt/sources.list.d/zeromq.list \
	&& curl -fsSL http://download.opensuse.org/repositories/home:zeromq:git-draft/xUbuntu_16.04/Release.key | apt-key add - \
	&& apt-get update \
	&& apt-get install -y --no-install-recommends \
		libopenblas-dev \
		libczmq-dev

RUN pip3 install \
	jupyter \
	ipykernel \
	numpy \
	scipy \
	pandas \
	matplotlib \
	seaborn \
	bokeh \
	plotly \
	holoviews \
	scikit-learn \
	scikit-image \
	sympy \
	gensim \
	nltk \
	cython \
	statsmodels \
	patsy \
	cloudpickle \
	dill \
	numba \
	xray \
	pyarrow \
	tensorflow \
	keras \
	xgboost \
	chainer

RUN gem install  \
	cztop \
	pry \
	pry-doc \
	awesome_print

RUN gem install iruby \
	&& gem install pycall -v '1.0.2' \
	&& gem install numpy \
	&& gem install pandas \
	&& gem install matplotlib \
	&& gem install numo-narray \
	&& gem install numo-linalg \
	&& gem install nmatrix \
	&& gem install nmatrix-lapacke \
	&& gem install red-arrow \
	&& gem install red-arrow-numo-narray \
	&& gem install red-arrow-nmatrix \
	&& gem install daru \
	&& gem install rbplotly

ENTRYPOINT ["tini", "--"]
CMD ["/bin/bash"]
