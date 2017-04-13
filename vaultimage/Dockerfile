FROM vault:0.7.0
MAINTAINER Randy Fay <rfay@newmediadenver.com>

# Provide for envsubst from gettext
RUN apk add --no-cache gettext curl

# These additional packages won't be necessary after dev stage
RUN apk add --no-cache curl bash

# Replace docker-entrypoint with one that doesn't rewrite args for dev version!
COPY docker-entrypoint.sh /usr/local/bin/docker-entrypoint.sh

ENTRYPOINT ["docker-entrypoint.sh"]

EXPOSE 8201

CMD ["vault", "server", "-config", "/vault/config"]
