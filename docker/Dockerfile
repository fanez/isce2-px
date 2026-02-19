FROM ubuntu:24.04 as builder

# Set an encoding to make things work smoothly.
ENV LANG en_US.UTF-8
ENV TZ US/Pacific
ARG DEBIAN_FRONTEND=noninteractive

RUN set -ex \
 && apt-get update \
 && apt-get install -y --no-install-recommends \
    build-essential \
    cmake \
    cython3 \
    gfortran \
    git \
    libfftw3-dev \
    libgdal-dev \
    libhdf4-alt-dev \
    libhdf5-dev \
    libopencv-dev \
    ninja-build \
    python3 \
    python3-dev \
    python3-gdal \
    python3-h5py \
    python3-numpy \
    python3-scipy \
 && ln -sf /usr/bin/cython3 /usr/local/bin/cython \
 && rm -rf /var/lib/apt/lists/*

# copy repo
COPY . /opt/isce2/src/isce2

# build ISCE
RUN set -ex \
 && cd /opt/isce2/src/isce2 \
 && cmake -S . -B build \
        -DCYTHON_EXECUTABLE=/usr/bin/cython3 \
        -DPYTHON_MODULE_DIR="$(python3 -c 'import site; print(site.getsitepackages()[-1])')" \
        -DCMAKE_INSTALL_PREFIX=install \
 && cmake --build build -j"$(nproc)" --target install \
 && cd build \
 && cpack -G DEB \
 && cp isce*.deb /tmp/

FROM ubuntu:24.04

# Set an encoding to make things work smoothly.
ENV LANG en_US.UTF-8
ENV TZ US/Pacific
ENV PATH /opt/venv/bin:$PATH
ARG DEBIAN_FRONTEND=noninteractive
ARG CDSE_USERNAME
ARG CDSE_PASSWORD

RUN set -ex \
 && apt-get update \
 && apt-get install -y --no-install-recommends \
    ca-certificates \
    curl \
    gdal-bin \
    libfftw3-double3 \
    python3 \
    python3-gdal \
    python3-h5py \
    python3-matplotlib \
    python3-numpy \
    python3-requests \
    python3-scipy \
    python3-shapely \
    python3-venv \
 && rm -rf /var/lib/apt/lists/*

# install ISCE from DEB
COPY --from=builder /tmp/isce*.deb /tmp/isce2.deb

RUN set -ex \
 && apt-get update \
 && apt-get install -y /tmp/isce2.deb \
 && rm -f /tmp/isce2.deb \
 && rm -rf /var/lib/apt/lists/*

# Install MintPy in a dedicated virtual environment.
RUN set -ex \
 && python3 -m venv /opt/venv \
 && /opt/venv/bin/pip install --no-cache-dir --upgrade pip setuptools wheel \
 && /opt/venv/bin/pip install --no-cache-dir --only-binary=:all: mintpy

# Provide an isce alias for packages expecting `import isce`.
RUN set -ex \
 && python3 - <<'PY'
import os
import site

site_pkgs = site.getsitepackages()[-1]
src = os.path.join(site_pkgs, 'isce2')
dst = os.path.join(site_pkgs, 'isce')
if os.path.isdir(src) and not os.path.exists(dst):
    os.symlink(src, dst)
PY

# Quick runtime sanity checks.
RUN set -ex \
 && python3 -c "import isce" \
 && /opt/venv/bin/python -c "import mintpy"

# Bake credentials into .netrc
RUN set -ex \
 && if [ -n "$CDSE_USERNAME" ] && [ -n "$CDSE_PASSWORD" ]; then \
      printf "machine dataspace.copernicus.eu\nlogin %s\npassword %s\n" "$CDSE_USERNAME" "$CDSE_PASSWORD" >> /root/.netrc; \
      chmod 600 /root/.netrc; \
    fi
