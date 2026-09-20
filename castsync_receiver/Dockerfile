ARG BUILD_FROM
FROM ${BUILD_FROM} AS airplay-builder
# shairport-sync built from source, not the Alpine package, because we need
# --with-stdout (so snapserver can read it directly) and --with-tinysvcmdns
# (self-contained mDNS responder, so the container doesn't need an
# avahi-daemon/D-Bus stack just to be discoverable as an AirPlay speaker).
RUN apk add --no-cache \
        build-base autoconf automake libtool git pkgconfig linux-headers \
        alsa-lib-dev libconfig-dev openssl-dev popt-dev soxr-dev
WORKDIR /src
RUN git clone --depth 1 https://github.com/mikebrady/tinysvcmdns.git \
    && cd tinysvcmdns \
    && make \
    && cp libtinysvcmdns.a /usr/lib/ \
    && cp *.h /usr/include/
RUN git clone --depth 1 --branch 4.3.4 https://github.com/mikebrady/shairport-sync.git \
    && cd shairport-sync \
    && autoreconf -fi \
    && ./configure --sysconfdir=/etc \
        --with-stdout \
        --with-tinysvcmdns \
        --with-ssl=openssl \
        --with-soxr \
        --without-alsa \
        --without-pa \
    && make -j"$(nproc)" \
    && make install

# ---------------------------------------------------------------------------
FROM ${BUILD_FROM} AS spotify-builder
# librespot built from source (musl target) rather than using the upstream
# release tarballs, which are glibc binaries and will not run on this Alpine
# base without gcompat. Slower image build, no ABI guessing.
RUN apk add --no-cache build-base rust cargo pkgconfig alsa-lib-dev openssl-dev git
WORKDIR /src
RUN git clone --depth 1 --branch v0.6.0 https://github.com/librespot-org/librespot.git \
    && cd librespot \
    && cargo build --release \
    && cp target/release/librespot /usr/local/bin/librespot

# ---------------------------------------------------------------------------
FROM ${BUILD_FROM} AS snapserver-builder
# No Alpine package for snapcast, so build the server binary only (no client,
# no avahi) from source.
RUN apk add --no-cache \
        build-base cmake git pkgconfig \
        alsa-lib-dev boost-dev flac-dev libvorbis-dev opus-dev soxr-dev \
        openssl-dev
WORKDIR /src
RUN git clone --depth 1 --branch v0.29.0 https://github.com/badaix/snapcast.git \
    && cd snapcast \
    && cmake -B build \
        -DCMAKE_BUILD_TYPE=Release \
        -DBUILD_CLIENT=OFF \
        -DBUILD_SERVER=ON \
        -DBUILD_WITH_AVAHI=OFF \
        -DBUILD_TESTS=OFF \
    && cmake --build build -j"$(nproc)" \
    && install -Dm755 build/server/snapserver /usr/local/bin/snapserver

# ---------------------------------------------------------------------------
FROM ${BUILD_FROM}
# Runtime-only shared libs matching the -dev packages used above, plus
# bashio/tempio/s6 which the HA base image already provides.
RUN apk add --no-cache \
        alsa-lib libconfig openssl-libs-static popt soxr \
        boost-system boost-filesystem boost-program_options \
        flac libvorbis opus \
        avahi-tools bash

COPY --from=airplay-builder /usr/local/bin/shairport-sync /usr/local/bin/shairport-sync
COPY --from=spotify-builder /usr/local/bin/librespot /usr/local/bin/librespot
COPY --from=snapserver-builder /usr/local/bin/snapserver /usr/local/bin/snapserver
# Upstream's maintained sample config, used as a base at runtime so every
# section we don't touch (buffer sizes, resampling, etc.) stays whatever the
# snapcast project currently recommends instead of a copy we'd silently drift
# out of date with.
COPY --from=snapserver-builder /src/snapcast/server/etc/snapserver.conf /etc/snapserver.conf.sample

COPY rootfs /

RUN chmod a+x /etc/s6-overlay/s6-rc.d/*/run 2>/dev/null || true

LABEL \
    io.hass.type="addon" \
    io.hass.arch="aarch64|amd64|armv7"
