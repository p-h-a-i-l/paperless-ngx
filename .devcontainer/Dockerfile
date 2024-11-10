# syntax=docker/dockerfile:1

FROM --platform=$BUILDPLATFORM docker.io/node:20-bookworm-slim as main-app

ARG DEBIAN_FRONTEND=noninteractive

# Buildx provided, must be defined to use though
ARG TARGETARCH

# Can be workflow provided, defaults set for manual building
ARG JBIG2ENC_VERSION=0.29
ARG QPDF_VERSION=11.9.0
ARG GS_VERSION=10.03.1


# ensure local python is preferred over distribution python
ENV PATH /usr/local/bin:$PATH

# cannot remove LANG even though https://bugs.python.org/issue19846 is fixed
# last attempted removal of LANG broke many users:
# https://github.com/docker-library/python/pull/570
ENV LANG C.UTF-8

# runtime dependencies
RUN set -eux; \
	apt-get update; \
	apt-get install -y --no-install-recommends \
		libbluetooth-dev \
		tk-dev \
		uuid-dev \
	; \
	rm -rf /var/lib/apt/lists/*

ENV GPG_KEY 7169605F62C751356D054A26A821E680E5FA6305
ENV PYTHON_VERSION 3.12.7
ENV PYTHON_SHA256 24887b92e2afd4a2ac602419ad4b596372f67ac9b077190f459aba390faf5550

RUN set -eux; \
	\
	wget -O python.tar.xz "https://www.python.org/ftp/python/${PYTHON_VERSION%%[a-z]*}/Python-$PYTHON_VERSION.tar.xz"; \
	echo "$PYTHON_SHA256 *python.tar.xz" | sha256sum -c -; \
	wget -O python.tar.xz.asc "https://www.python.org/ftp/python/${PYTHON_VERSION%%[a-z]*}/Python-$PYTHON_VERSION.tar.xz.asc"; \
	GNUPGHOME="$(mktemp -d)"; export GNUPGHOME; \
	gpg --batch --keyserver hkps://keys.openpgp.org --recv-keys "$GPG_KEY"; \
	gpg --batch --verify python.tar.xz.asc python.tar.xz; \
	gpgconf --kill all; \
	rm -rf "$GNUPGHOME" python.tar.xz.asc; \
	mkdir -p /usr/src/python; \
	tar --extract --directory /usr/src/python --strip-components=1 --file python.tar.xz; \
	rm python.tar.xz; \
	\
	cd /usr/src/python; \
	gnuArch="$(dpkg-architecture --query DEB_BUILD_GNU_TYPE)"; \
	./configure \
		--build="$gnuArch" \
		--enable-loadable-sqlite-extensions \
		--enable-optimizations \
		--enable-option-checking=fatal \
		--enable-shared \
		--with-lto \
		--with-ensurepip \
	; \
	nproc="$(nproc)"; \
	EXTRA_CFLAGS="$(dpkg-buildflags --get CFLAGS)"; \
	LDFLAGS="$(dpkg-buildflags --get LDFLAGS)"; \
	make -j "$nproc" \
		"EXTRA_CFLAGS=${EXTRA_CFLAGS:-}" \
		"LDFLAGS=${LDFLAGS:-}" \
	; \
# https://github.com/docker-library/python/issues/784
# prevent accidental usage of a system installed libpython of the same version
	rm python; \
	make -j "$nproc" \
		"EXTRA_CFLAGS=${EXTRA_CFLAGS:-}" \
		"LDFLAGS=${LDFLAGS:--Wl},-rpath='\$\$ORIGIN/../lib'" \
		python \
	; \
	make install; \
	\
# enable GDB to load debugging data: https://github.com/docker-library/python/pull/701
	bin="$(readlink -ve /usr/local/bin/python3)"; \
	dir="$(dirname "$bin")"; \
	mkdir -p "/usr/share/gdb/auto-load/$dir"; \
	cp -vL Tools/gdb/libpython.py "/usr/share/gdb/auto-load/$bin-gdb.py"; \
	\
	cd /; \
	rm -rf /usr/src/python; \
	\
	find /usr/local -depth \
		\( \
			\( -type d -a \( -name test -o -name tests -o -name idle_test \) \) \
			-o \( -type f -a \( -name '*.pyc' -o -name '*.pyo' -o -name 'libpython*.a' \) \) \
		\) -exec rm -rf '{}' + \
	; \
	\
	ldconfig; \
	\
	export PYTHONDONTWRITEBYTECODE=1; \
	python3 --version; \
	pip3 --version

# make some useful symlinks that are expected to exist ("/usr/local/bin/python" and friends)
RUN set -eux; \
	for src in idle3 pip3 pydoc3 python3 python3-config; do \
		dst="$(echo "$src" | tr -d 3)"; \
		[ -s "/usr/local/bin/$src" ]; \
		[ ! -e "/usr/local/bin/$dst" ]; \
		ln -svT "$src" "/usr/local/bin/$dst"; \
	done













# Set Python environment variables
ENV PYTHONDONTWRITEBYTECODE=1 \
    PYTHONUNBUFFERED=1 \
    # Ignore warning from Whitenoise
    PYTHONWARNINGS="ignore:::django.http.response:517" \
    PNGX_CONTAINERIZED=1

#
# Begin installation and configuration
# Order the steps below from least often changed to most
#

# Packages need for running
ARG RUNTIME_PACKAGES="\
  # General utils
  curl \
  # Docker specific
  gosu \
  # Timezones support
  tzdata \
  # fonts for text file thumbnail generation
  fonts-liberation \
  gettext \
  ghostscript \
  gnupg \
  icc-profiles-free \
  imagemagick \
  # PostgreSQL
  postgresql-client \
  # MySQL / MariaDB
  mariadb-client \
  # OCRmyPDF dependencies
  tesseract-ocr \
  tesseract-ocr-eng \
  tesseract-ocr-deu \
  tesseract-ocr-fra \
  tesseract-ocr-ita \
  tesseract-ocr-spa \
  unpaper \
  pngquant \
  jbig2dec \
  # lxml
  libxml2 \
  libxslt1.1 \
  # itself
  qpdf \
  # Mime type detection
  file \
  libmagic1 \
  media-types \
  zlib1g \
  # Barcode splitter
  libzbar0 \
  poppler-utils \
  htop \
  sudo"

# Install basic runtime packages.
# These change very infrequently
RUN set -eux \
  echo "Installing system packages" \
    && apt-get update \
    && apt-get install --yes --quiet --no-install-recommends ${RUNTIME_PACKAGES}

ARG PYTHON_PACKAGES="\
  pipenv \
  ca-certificates"

RUN set -eux \
  echo "Installing python packages" \
    && apt-get update \
    && apt-get install --yes --quiet ${PYTHON_PACKAGES}

RUN set -eux \
  && echo "Installing pre-built updates" \
    && echo "Installing qpdf ${QPDF_VERSION}" \
      && curl --fail --silent --show-error --location \
        --output libqpdf29_${QPDF_VERSION}-1_${TARGETARCH}.deb \
        https://github.com/paperless-ngx/builder/releases/download/qpdf-${QPDF_VERSION}/libqpdf29_${QPDF_VERSION}-1_${TARGETARCH}.deb \
      && curl --fail --silent --show-error --location \
        --output qpdf_${QPDF_VERSION}-1_${TARGETARCH}.deb \
        https://github.com/paperless-ngx/builder/releases/download/qpdf-${QPDF_VERSION}/qpdf_${QPDF_VERSION}-1_${TARGETARCH}.deb \
      && dpkg --install ./libqpdf29_${QPDF_VERSION}-1_${TARGETARCH}.deb \
      && dpkg --install ./qpdf_${QPDF_VERSION}-1_${TARGETARCH}.deb \
    && echo "Installing Ghostscript ${GS_VERSION}" \
      && curl --fail --silent --show-error --location \
          --output libgs10_${GS_VERSION}.dfsg-1_${TARGETARCH}.deb \
          https://github.com/paperless-ngx/builder/releases/download/ghostscript-${GS_VERSION}/libgs10_${GS_VERSION}.dfsg-1_${TARGETARCH}.deb \
      && curl --fail --silent --show-error --location \
          --output ghostscript_${GS_VERSION}.dfsg-1_${TARGETARCH}.deb \
          https://github.com/paperless-ngx/builder/releases/download/ghostscript-${GS_VERSION}/ghostscript_${GS_VERSION}.dfsg-1_${TARGETARCH}.deb \
      && curl --fail --silent --show-error --location \
          --output libgs10-common_${GS_VERSION}.dfsg-1_all.deb \
          https://github.com/paperless-ngx/builder/releases/download/ghostscript-${GS_VERSION}/libgs10-common_${GS_VERSION}.dfsg-1_all.deb \
        && dpkg --install ./libgs10-common_${GS_VERSION}.dfsg-1_all.deb \
        && dpkg --install ./libgs10_${GS_VERSION}.dfsg-1_${TARGETARCH}.deb \
        && dpkg --install ./ghostscript_${GS_VERSION}.dfsg-1_${TARGETARCH}.deb \
    && echo "Installing jbig2enc" \
      && curl --fail --silent --show-error --location \
        --output jbig2enc_${JBIG2ENC_VERSION}-1_${TARGETARCH}.deb \
        https://github.com/paperless-ngx/builder/releases/download/jbig2enc-${JBIG2ENC_VERSION}/jbig2enc_${JBIG2ENC_VERSION}-1_${TARGETARCH}.deb \
      && dpkg --install ./jbig2enc_${JBIG2ENC_VERSION}-1_${TARGETARCH}.deb

# setup docker-specific things
# These change sometimes, but rarely
WORKDIR /usr/src/paperless/src/docker/

COPY [ \
  "docker/imagemagick-policy.xml", \
  "./" \
]

RUN set -eux \
  && echo "Configuring ImageMagick" \
    && mv imagemagick-policy.xml /etc/ImageMagick-6/policy.xml

# Packages needed only for building a few quick Python
# dependencies
ARG BUILD_PACKAGES="\
  build-essential \
  git \
  # https://www.psycopg.org/docs/install.html#prerequisites
  libpq-dev \
  # https://github.com/PyMySQL/mysqlclient#linux
  default-libmysqlclient-dev \
  pkg-config \
  pre-commit"

# hadolint ignore=DL3042
RUN --mount=type=cache,target=/root/.cache/pip/,id=pip-cache \
  set -eux \
  && echo "Installing build system packages" \
    && apt-get update \
    && apt-get install --yes --quiet ${BUILD_PACKAGES}

RUN set -eux \
  && npm update npm -g

# add users, setup scripts
# Mount the compiled frontend to expected location
RUN set -eux \
  && echo "Setting up user/group" \
    && groupmod --new-name paperless node \
    && usermod --login paperless --home /usr/src/paperless node \
    && usermod -s /bin/bash paperless \
    && echo "paperless ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers \
  && echo "Creating volume directories" \
    && mkdir --parents --verbose /usr/src/paperless/paperless-ngx/data \
    && mkdir --parents --verbose /usr/src/paperless/paperless-ngx/media \
    && mkdir --parents --verbose /usr/src/paperless/paperless-ngx/consume \
    && mkdir --parents --verbose /usr/src/paperless/paperless-ngx/export \
    && mkdir --parents --verbose /usr/src/paperless/paperless-ngx/.venv \
  && echo "Adjusting all permissions" \
    && chown --from root:root --changes --recursive paperless:paperless /usr/src/paperless
#  && echo "Collecting static files" \
#    && gosu paperless python3 manage.py collectstatic --clear --no-input --link \
#    && gosu paperless python3 manage.py compilemessages

VOLUME ["/usr/src/paperless/paperless-ngx/data", \
        "/usr/src/paperless/paperless-ngx/media", \
        "/usr/src/paperless/paperless-ngx/consume", \
        "/usr/src/paperless/paperless-ngx/export", \
        "/usr/src/paperless/paperless-ngx/.venv"]
