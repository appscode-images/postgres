# =============================================================================
# PostgreSQL 18.6 bookworm (Postgres Enterprise by AppsCode) — Custom Image
#
# Base already ships: pgvector, PostGIS, pg_repack, pg_cron, pgaudit,
#                     credcheck, appscode_license  — none of them are touched.
#
# Added here: set_user 4.2.0, pg_stat_monitor 2.4.0, passwordcheck_cracklib 3.2.0,
#             manage_user_permissions 1.4
# =============================================================================

ARG BASE_IMAGE=ghcr.io/appscode-images/postgres-enterprise:18.6-bookworm-ext

# -----------------------------------------------------------------------------
# Stage 1: Builder
# -----------------------------------------------------------------------------
FROM ${BASE_IMAGE} AS builder

USER root

ENV SET_USER_VERSION=REL4_2_0
ENV PG_STAT_MONITOR_VERSION=2.4.0
ENV PASSWORDCHECK_CRACKLIB_VERSION=3.2.0
ENV MANAGE_USER_PERMISSIONS_COMMIT=8b3f8ed79e742c4fa3b19584e8c38a62e354743e

# passwordcheck_cracklib hardcodes an RHEL dictionary path in its Makefile.
# On Debian the cracklib dictionary lives here, so PG_CPPFLAGS is overridden below.
ENV CRACKLIB_DICTPATH=/var/cache/cracklib/cracklib_dict

ENV PATH="/usr/local/bin:${PATH}"

RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        build-essential \
        ca-certificates \
        cracklib-runtime \
        curl \
        git \
        libcrack2-dev \
        libkrb5-dev \
        libssl-dev \
        pkg-config \
    && rm -rf /var/lib/apt/lists/*

# Staged install root: only the newly built files land here, which is what keeps
# the final COPY from disturbing anything already present in the base image.
# with_llvm=no skips bitcode emission so no clang version match is required.
RUN mkdir -p /staging

# ---------------------------------------------------------------------------
# 1. set_user
# ---------------------------------------------------------------------------
RUN git clone --branch ${SET_USER_VERSION} --depth 1 \
        https://github.com/pgaudit/set_user.git /tmp/set_user

RUN cd /tmp/set_user \
    && make USE_PGXS=1 with_llvm=no

RUN cd /tmp/set_user \
    && make USE_PGXS=1 with_llvm=no install DESTDIR=/staging

# ---------------------------------------------------------------------------
# 2. pg_stat_monitor
#
# Must be listed after pg_stat_statements in shared_preload_libraries.
# ---------------------------------------------------------------------------
RUN git clone --branch ${PG_STAT_MONITOR_VERSION} --depth 1 \
        https://github.com/percona/pg_stat_monitor.git /tmp/pg_stat_monitor

RUN cd /tmp/pg_stat_monitor \
    && make USE_PGXS=1 with_llvm=no

RUN cd /tmp/pg_stat_monitor \
    && make USE_PGXS=1 with_llvm=no install DESTDIR=/staging

# ---------------------------------------------------------------------------
# 3. passwordcheck_cracklib
#
# MODULE_big, not a CREATE EXTENSION extension: it installs a .so only, with no
# .control file, and is activated through shared_preload_libraries. It will
# never appear in pg_available_extensions — verify it by asserting that a weak
# password is rejected on CREATE ROLE / ALTER ROLE.
# ---------------------------------------------------------------------------
RUN git clone --branch ${PASSWORDCHECK_CRACKLIB_VERSION} --depth 1 \
        https://github.com/devrimgunduz/passwordcheck_cracklib.git \
        /tmp/passwordcheck_cracklib

RUN cd /tmp/passwordcheck_cracklib \
    && make USE_PGXS=1 with_llvm=no \
        PG_CPPFLAGS="-DUSE_CRACKLIB '-DCRACKLIB_DICTPATH=\"${CRACKLIB_DICTPATH}\"'"

RUN cd /tmp/passwordcheck_cracklib \
    && make USE_PGXS=1 with_llvm=no \
        PG_CPPFLAGS="-DUSE_CRACKLIB '-DCRACKLIB_DICTPATH=\"${CRACKLIB_DICTPATH}\"'" \
        install DESTDIR=/staging

# ---------------------------------------------------------------------------
# 4. manage_user_permissions
#
# Pure SQL/plpgsql: DATA + .control only, no C and no shared library, so it
# builds identically on every base. The upstream Makefile hardcodes
# PG_CONFIG=/usr/pgsql-16/bin/pg_config, a PGDG RHEL path that does not exist
# in these images, so PG_CONFIG is overridden on the command line where it
# takes precedence over the makefile assignment.
#
# The repo publishes no tags, so the commit is pinned instead.
# ---------------------------------------------------------------------------
RUN git clone https://github.com/kubedb/manage_user_permissions.git \
        /tmp/manage_user_permissions

RUN cd /tmp/manage_user_permissions \
    && git checkout ${MANAGE_USER_PERMISSIONS_COMMIT}

RUN cd /tmp/manage_user_permissions \
    && make USE_PGXS=1 PG_CONFIG=pg_config install DESTDIR=/staging

# =============================================================================
# Stage 2: Final image
# =============================================================================
FROM ${BASE_IMAGE}

USER root

# Runtime libraries: libcrack for the module, plus the dictionary it opens at
# CRACKLIB_DICTPATH.
RUN apt-get update \
    && apt-get install -y --no-install-recommends \
        cracklib-runtime \
        libcrack2 \
    && rm -rf /var/lib/apt/lists/*

# Only the three new extensions are copied in; every pre-existing extension in
# the base image is left exactly as it was.
COPY --from=builder /staging/ /

USER postgres
