FROM dhi.io/postgres:18.3-debian13-dev AS builder

FROM dhi.io/postgres:18.3-debian13
COPY --from=builder /usr/bin/find  /usr/bin/find
COPY --from=builder /usr/bin/grep  /usr/bin/grep
