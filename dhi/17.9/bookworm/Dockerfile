FROM dhi.io/postgres:17.9-debian13-dev AS builder

FROM dhi.io/postgres:17.9-debian13
COPY --from=builder /usr/bin/find  /usr/bin/find
COPY --from=builder /usr/bin/grep  /usr/bin/grep
