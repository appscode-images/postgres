FROM dhi.io/postgres:18-debian13-dev AS builder
RUN apt-get update && apt-get install -y --no-install-recommends findutils grep

FROM dhi.io/postgres:18-debian13
COPY --from=builder /usr/bin/find  /usr/bin/find
COPY --from=builder /usr/bin/grep  /usr/bin/grep
COPY --from=builder /usr/bin/xargs /usr/bin/xargs