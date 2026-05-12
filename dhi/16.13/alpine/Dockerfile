FROM dhi.io/postgres:16.13-alpine3.22-dev AS builder

FROM dhi.io/postgres:16.13-alpine3.22
COPY --from=builder /usr/bin/find  /usr/bin/find
COPY --from=builder /usr/bin/grep  /usr/bin/grep
