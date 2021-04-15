#
# Scala and sbt Dockerfile
#
# https://github.com/zalbia/scala-sbt
#

# Pull base image
ARG BASE_IMAGE_TAG
FROM ghcr.io/graalvm/graalvm-ce:${BASE_IMAGE_TAG:-java11-21.0.0}

# Env variables
ARG SCALA_VERSION
ENV SCALA_VERSION ${SCALA_VERSION:-2.13.5}
ARG SBT_VERSION
ENV SBT_VERSION ${SBT_VERSION:-1.4.9}
ENV JAVA_OPTS -XX:+UseG1GC 
ARG USER_ID
ENV USER_ID ${USER_ID:-1001}
ARG GROUP_ID 
ENV GROUP_ID ${GROUP_ID:-1001}
ARG NODE_VERSION
ENV NODE_VERSION ${NODE_VERSION:-14.16.0}
ARG ARCH
ENV ARCH ${ARCH:-"x64"}

# Install sbt
RUN \
  curl -fsL "https://github.com/sbt/sbt/releases/download/v$SBT_VERSION/sbt-$SBT_VERSION.tgz" | tar xfz - -C /usr/share && \
  chown -R root:root /usr/share/sbt && \
  chmod -R 755 /usr/share/sbt && \
  ln -s /usr/share/sbt/bin/sbt /usr/local/bin/sbt

# Install Scala
## Piping curl directly in tar
RUN \
  case $SCALA_VERSION in \
    "3"*) URL=https://github.com/lampepfl/dotty/releases/download/$SCALA_VERSION/scala3-$SCALA_VERSION.tar.gz SCALA_DIR=/usr/share/scala3-$SCALA_VERSION ;; \
    *) URL=https://downloads.typesafe.com/scala/$SCALA_VERSION/scala-$SCALA_VERSION.tgz SCALA_DIR=/usr/share/scala-$SCALA_VERSION ;; \
  esac && \
  curl -fsL $URL | tar xfz - -C /usr/share && \
  mv $SCALA_DIR /usr/share/scala && \
  chown -R root:root /usr/share/scala && \
  chmod -R 755 /usr/share/scala && \
  ln -s /usr/share/scala/bin/* /usr/local/bin && \
  echo "println(util.Properties.versionMsg)" > test.scala && scala test.scala && rm test.scala

# Add and use user sbtuser
RUN groupadd --gid $GROUP_ID sbtuser && useradd --gid $GROUP_ID --uid $USER_ID sbtuser --shell /bin/bash
RUN chown -R sbtuser:sbtuser /opt
# RUN mkdir /home/sbtuser && chown -R sbtuser:sbtuser /home/sbtuser
RUN mkdir /logs && chown -R sbtuser:sbtuser /logs
USER sbtuser

# Switch working directory
WORKDIR /home/sbtuser

# Prepare sbt (warm cache)
RUN \
  sbt sbtVersion && \
  mkdir -p project && \
  echo "scalaVersion := \"${SCALA_VERSION}\"" > build.sbt && \
  echo "sbt.version=${SBT_VERSION}" > project/build.properties && \
  echo "case object Temp" > Temp.scala && \
  sbt compile && \
  rm -r project && rm build.sbt && rm Temp.scala && rm -r target

# Link everything into root as well
# This allows users of this container to choose, whether they want to run the container as sbtuser (non-root) or as root
USER root
RUN \
  ln -s /home/sbtuser/.cache /root/.cache && \
  ln -s /home/sbtuser/.ivy2 /root/.ivy2 && \
  ln -s /home/sbtuser/.sbt /root/.sbt


# gpg keys listed at https://github.com/nodejs/node#release-keys
RUN set -ex \
  && for key in \
    4ED778F539E3634C779C87C6D7062848A1AB005C \
    94AE36675C464D64BAFA68DD7434390BDBE9B9C5 \
    74F12602B6F1C4E913FAA37AD3A89613643B6201 \
    71DCFD284A79C3B38668286BC97EC7A07EDE3FC1 \
    8FCCA13FEF1D0C2E91008E09770F7A9A5AE15600 \
    C4F0DFFF4E8C1A8236409D08E73BC641CC11F4C8 \
    C82FA3AE1CBEDC6BE46B9360C43CEC45C17AB93C \
    DD8F2338BAE7501E3DD5AC78C273792F7D83545D \
    A48C2BEE680E841632CD4E44F07496B3EB3C1762 \
    108F52B48DB57BB0CC439B2997B01419BD92F80A \
    B9E2F5981AA6E0CD28160D9FF13993A75599653C \
  ; do \
    gpg --batch --keyserver hkp://p80.pool.sks-keyservers.net:80 --recv-keys "$key" || \
    gpg --batch --keyserver hkp://ipv4.pool.sks-keyservers.net --recv-keys "$key" || \
    gpg --batch --keyserver hkp://pgp.mit.edu:80 --recv-keys "$key" ; \
  done \
  && curl -fsSLO --compressed "https://nodejs.org/dist/v$NODE_VERSION/node-v$NODE_VERSION-linux-$ARCH.tar.gz" \
  && curl -fsSLO --compressed "https://nodejs.org/dist/v$NODE_VERSION/SHASUMS256.txt.asc" \
  && gpg --batch --decrypt --output SHASUMS256.txt SHASUMS256.txt.asc \
  && grep "node-v$NODE_VERSION-linux-$ARCH.tar.gz\$" SHASUMS256.txt | sha256sum -c \
  && tar -xf "node-v$NODE_VERSION-linux-$ARCH.tar.gz" -C /usr/local --strip-components=1 --no-same-owner \
  && rm "node-v$NODE_VERSION-linux-$ARCH.tar.gz" SHASUMS256.txt.asc SHASUMS256.txt \
  && ln -s /usr/local/bin/node /usr/local/bin/nodejs \
  # smoke tests
  && node --version \
  && npm --version

RUN npm i -g pnpm@6.0.2 \
  # smoke test
  && pnpm --version

# Switch working directory back to root
## Users wanting to use this container as non-root should combine the two following arguments
## -u sbtuser
## -w /home/sbtuser
WORKDIR /root


