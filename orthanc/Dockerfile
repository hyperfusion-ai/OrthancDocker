FROM debian:stable-slim

MAINTAINER Sebastien Jodogne <s.jodogne@gmail.com>
LABEL Description="Orthanc, free DICOM server" Vendor="The Orthanc project"

WORKDIR /root/

ADD ./setup-locales.sh ./setup-locales.sh
RUN bash ./setup-locales.sh

ADD ./download-orthanc.sh ./download-orthanc.sh
RUN bash ./download-orthanc.sh

ADD ./create-config.sh ./create-config.sh
RUN bash ./create-config.sh

RUN rm ./setup-locales.sh ./download-orthanc.sh ./create-config.sh

VOLUME [ "/var/lib/orthanc/db" ]
EXPOSE 4242
EXPOSE 8042

# Check if useradd is installed, if not install it
RUN which useradd || apt-get update && apt-get install -y passwd

# Check if user with UID 1000 exists, if not create one
RUN id -u 1000 &>/dev/null || useradd -u 1000 -m orthanc

# Give the user with UID 1000 permissions for the application files
RUN chown -R 1000:1000 /root /etc/orthanc /var/lib/orthanc

# Switch to the user with UID 1000
USER 1000

ENTRYPOINT [ "Orthanc" ]
CMD [ "/etc/orthanc/" ]

# https://groups.google.com/d/msg/orthanc-users/qWqxpvCPv8g/Z8huoA5FDAAJ
ENV MALLOC_ARENA_MAX 5
