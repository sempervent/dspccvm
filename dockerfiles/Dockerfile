FROM debian:stretch-slim
MAINTAINER 'Joshua N. Grant <jngrant9@gmail.com>'

# ensure local python is preferred over distribution python
ENV PATH /usr/local/bin:$PATH

# set the locale
RUN apt-get update && apt-get install -y locales && rm -rf /var/lib/apt/lists/* \
    && localedef -i en_US -c -f UTF-8 -A /usr/share/locale/locale.alias en_US.UTF-8
ENV LANG en_US.utf8
ENV LC_ALL en_US.UTF-8

# install R
RUN echo "deb http://http.debian.net/debian stretch main" > /etc/apt/sources.list.d/debian-unstable.list \
   && echo 'APT::Default-Release "testing";' > /etc/apt/apt.conf.d/default

ENV R_BASE_VERSION 3.5.2
RUN apt-get update \
   && apt-get install -t unstable -y --no-install-recommends \
      littler \
      r-cran-littler \
      r-base=${R_BASE_VERSION}-* \
      r-base-dev=${R_BASE_VERSION}-* \
      r-recommended=${R_BASE_VERSION}-* \
   && ln -s /usr/lib/R/site-library/littler/examples/install.r /usr/local/bin/install.r \
   && ln -s /usr/lib/R/site-library/littler/examples/install2.r /usr/local/bin/install2.r \
   && ln -s /usr/lib/R/site-library/littler/examples/installGithub.r /usr/local/bin/installGithub.r \
   && ln -s /usr/lib/R/site-library/littler/examples/testInstalled.r \
   && install.r docopt \
   && rm -rf /tmp/downloaded_packages/ /tmp/*.rds \
   && rm -rf /var/lib/apt/lists/*


# python3
ENV PYTHON_VERSION 3.6.8
ENV GPG_KEY 0D96DF4D4110E5C43FBFB17F2D347EA6AA65421D
RUN set -ex \
   \
   && wget -O python.tar.xz "https://www.python.org/ftp/python/$(PYTHON_VERSION%%[a-z]*)/Python-$PYTHON_VERSION.tar.xz" \
   && wget -O python.tar.xz.asc "https://www.python.org/ftp/python/$(PYTHON_VERSION%%[a-z]*)/Python-$PYTHON_VERSION.tar.xz.asc" \
   && export GNUPGHOME="$(mktemp -d)" \
   && gpg --batch --keyserver ha.pool.sks-keyservers.net --recv-keys "$GPG_KEY" \
   && gpg --batch --verify python.tar.xz.asc python.tar.xz \
   && { command -v gpgconf > /dev/null && gpgconf --kill all || :; } \
   && rm -rf "$GNUPGHOME" python.tar.xz.asc \
   && mkdir -p /usr/src/python\
   && tar -xJC /usr/src/python --strip-components=1 python.tar.xz \
   && rm python.tar.xz \
   && cd /usr/src/python \
   && gnuArch="$(dpkg-architecture --query DEB_BUILD_GNU_TYPE)" \
   && ./configure \
      --build="$gnuArch" \
      --enable-loadable-sqlite-extensions \
      --enable-shared \
      --with-system-expat \
      --with-system-ffi \
      --without-ensuregrip \
   && make -j "$(nproc)" \
   && make install \
   && ldconfig \
   \
   && find /usr/ocal -depth \
      \( \
         \( -type d -a \( -name test -o -name tests \) \) \
         -o \
         \( -type f -a \( -name '*.pyc' -o -name '*.pyo' \) \) \
      \) -exec rm -rf '{}' + \
   && rm -rf /usr/src/python \
   \
   && python3 --version

# make some useful symlinks that are expected to exist
RUN cd /usr/local/bin \
   && ln -s idle3 idle \
   && ln -s pydoc3 pydoc \
   && ln -s python3 python \
   && ln -s python3-config python-config

# pip setup
ENV PYTHON_PIP_VERSION 18.1
RUN set -ex; \
   \
   wget -O get-pip.py 'https://bootstrap.pypa.io/get-pip.py'; \
   \
   python get-pip.py \
      --disable-pip-version-check \
      --no-cache-dir \
      "pip==$PYTHON_PIP_VERSION" \
   ; \
   pip --version; \
   \
   find /usr/local -depth \
      \( \
         \( -type d -a \( -name test -o -name tests \) \) \
         -o \
         \( -type f -a \( -name '*.pyc' -o -name '*.pyo' \) \) \
      \) -exec rm -rf '{}' +; \
   rm -f get-pip.py

CMD["python3"]

# Airflow
ARG AIRFLOW_VERSION=1.10.1
ARG AIRFLOW_HOME=/usr/local/airflow
ARG AIRFLOW_DEPS=""
ENV AIRFLOW_GPL_UNIDECODE yes


RUN -ex \
   && buildDeps=' \
      freetds-dev \
      libkrb5-dev \
      libsasl2-dev \
      libssl-dev \
      libffi-dev \
      libpq-dev \
      git \
   '\
   && apt-get update -yqq \
   && apt-get upgrade -yqq \
   && apt-get install -yqq --no-install-recommends \
      &buildDeps \
      freetds-bin \
      build-essential \
      default-libmysqlclient-dev \
      apt-utils \
      curl \
      rsync \
      netcat \
   && useradd -ms /bin/bash -d ${AIRFLOW_HOME} airflow \
   && pip install -U pip setuptools wheel \
   && pip install pytz \
   && pip install pyOpenSSL \
   && pip install ndg-httpsclient \
   && pip install pyasn1 \
   && pip install apache-airflow[crypto,celery,ssh,mysql,jdbc,postgresi${AIRFLOW_DEPS:+}${AIRFLOW_DEPS}]==${AIRFLOW_VERSION} \
   && pip install 'redis>=2.10.5,<3' \
   && apt-get purge --auto-remove -yqq $buildDeps \
   && apt-get autoremove -yqq --purge \
   && apt-get clean \
   && rm -rf \
      /var/lib/apt/lists/* \
      /tmp/* \
      /var/tmp/* \
      /usr/share/man \
      /usr/share/doc \
      /usr/share/doc-base
COPY script/entrypoint.sh /entrypoint.sh
COPY config/airflow.cfg ${AIRFLOW_HOME}/airflow.cfg

RUN chown -R airflow: ${AIRFLOW_HOME}

EXPOSE 8080 5555 8793

USER airflow
WORKDIR ${AIRFLOW_HOME}
ENTRYPOINT ["/entrypoint.sh"]
CMD {"webserver"}

