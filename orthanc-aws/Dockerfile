##############################################################################
### First stage: build the Orthanc aws plugin ###############################
##############################################################################
# From https://book.orthanc-server.com/plugins/object-storage.html#id1
# For testing use: docker build --target build-orthanc-aws -t build-orthanc-aws -f Dockerfile .
FROM jodogne/orthanc-python:1.7.3 AS build-orthanc-aws

# aws-sdk-cpp dependencies (aws-c-common, aws-checksums, aws-c-event-stream)
RUN apt-get -y clean && apt-get -y update
RUN DEBIAN_FRONTEND=noninteractive apt-get -y install \
    cmake unzip git build-essential libssl-dev zlib1g-dev libcurl4-openssl-dev

# Compile and install Osimis AWS plugin. Don't use vcpkg. Use libcrypto from package manager (?)
# Note when using the Orthanc LSB binaries, only building a static library is
# likely to work at the moment, see https://groups.google.com/g/orthanc-users/c/AZ804k8ba1A/m/kajtEdKUBAAJ
RUN DEBIAN_FRONTEND=noninteractive apt-get -y install \
    libboost-filesystem-dev libboost-thread-dev libboost-system-dev libboost-regex-dev libboost-date-time-dev libboost-atomic-dev \
    mercurial uuid-dev libcrypto++-dev

# Clone source:
RUN cd ~ && hg clone https://hg.orthanc-server.com/orthanc-object-storage
# Add caFile to config:
COPY ./orthanc-object-storage_AwsS3StoragePlugin-ca.patch /root/orthanc-object-storage
RUN cd ~/orthanc-object-storage && patch -p1 < orthanc-object-storage_AwsS3StoragePlugin-ca.patch
# Compile plugin
RUN mkdir -p ~/build/aws
RUN cd ~/build/aws && cmake -DSTATIC_BUILD=ON -DCMAKE_BUILD_TYPE=Release -DUSE_VCPKG_PACKAGES=OFF -DSTATIC_AWS_CLIENT=ON -DALLOW_DOWNLOADS=ON -DUSE_SYSTEM_GOOGLE_TEST=OFF -DENABLE_GOOGLE_TEST=OFF -DUSE_SYSTEM_JSONCPP=OFF -DORTHANC_FRAMEWORK_VERSION=1.7.3 ../../orthanc-object-storage/Aws && make -j4


# ##############################################################################
# ### Second stage: Add AWS plugin to the orthanc-python image ################
# ##############################################################################
FROM jodogne/orthanc-python:1.7.3 AS orthanc-aws

MAINTAINER Sebastien Jodogne <s.jodogne@gmail.com>
LABEL Description="Orthanc, free DICOM server, with plugins, aws s3 and Python" Vendor="The Orthanc project"

# Copy plugin shared library artifact from previous stage and install missing runtime requirements
COPY --from=build-orthanc-aws /root/build/aws/libOrthancAwsS3Storage.so \
     /usr/local/share/orthanc/plugins/

# Install dependencies needed for plugin (note includes some AWS CPP SDK dependencies):
RUN apt-get -y clean && apt-get -y update
RUN DEBIAN_FRONTEND=noninteractive apt-get -y install \
    libcurl4-openssl-dev libcrypto++-dev && \
    apt-get clean && rm -rf /var/lib/apt/lists/*

VOLUME [ "/var/lib/orthanc/db" ]
EXPOSE 4242
EXPOSE 8042

ENTRYPOINT [ "Orthanc" ]
CMD [ "/etc/orthanc/" ]

# https://groups.google.com/d/msg/orthanc-users/qWqxpvCPv8g/Z8huoA5FDAAJ
ENV MALLOC_ARENA_MAX 5
