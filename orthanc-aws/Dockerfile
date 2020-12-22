##############################################################################
### First stage: build the Orthanc aws plugin ###############################
##############################################################################
# From https://book.orthanc-server.com/plugins/object-storage.html#id1
# For testing use: docker build --target build-orthanc-aws -t build-orthanc-aws -f Dockerfile .
FROM jodogne/orthanc-python:1.8.2 AS build-orthanc-aws

# git is needed for installing aws-sdk-cpp's dependencies (aws-c-common, aws-checksums, aws-c-event-stream)
RUN apt-get -y clean && apt-get -y update
RUN DEBIAN_FRONTEND=noninteractive apt-get -y install \
    wget unzip cmake git build-essential libcurl4-openssl-dev libssl-dev uuid-dev zlib1g-dev \
    mercurial libjsoncpp-dev libboost-filesystem-dev libboost-thread-dev libboost-system-dev libboost-regex-dev libboost-date-time-dev libboost-atomic-dev libcrypto++-dev && \
    apt-get clean && rm -rf /var/lib/apt/lists/*

# Install AWS C++ SDK:
# More background on https://docs.aws.amazon.com/sdk-for-cpp/v1/developer-guide/setup.html
RUN mkdir ~/aws
RUN cd ~/aws && wget https://github.com/aws/aws-sdk-cpp/archive/1.8.111.zip && unzip 1.8.111.zip && mv aws-sdk-cpp-1.8.111 aws-sdk-cpp
RUN mkdir -p ~/aws/builds/aws-sdk-cpp && cd ~/aws/builds/aws-sdk-cpp && cmake -DBUILD_ONLY="s3;transfer" ~/aws/aws-sdk-cpp && make -j 4 && make install

# Install Osimis AWS plugin. Don't use vcpkg. Use libcrypto from package manager (?)
RUN cd ~ && hg clone https://hg.orthanc-server.com/orthanc-object-storage
RUN mkdir -p ~/build/aws
# Note only building a static library appears to work at the moment, see https://groups.google.com/g/orthanc-users/c/AZ804k8ba1A/m/kajtEdKUBAAJ
# RUN cd ~/build/aws && cmake -DUSE_VCPKG_PACKAGES=OFF -DUSE_SYSTEM_GOOGLE_TEST=OFF -DALLOW_DOWNLOADS=ON ../../orthanc-object-storage/Aws && make -j 4
RUN cd ~/build/aws && cmake -DSTATIC_BUILD=ON -DUSE_VCPKG_PACKAGES=OFF -DUSE_SYSTEM_GOOGLE_TEST=OFF -DALLOW_DOWNLOADS=ON ../../orthanc-object-storage/Aws && make -j 4


##############################################################################
### Second stage: Add AWS plugin to the orthanc-python image ################
##############################################################################
FROM jodogne/orthanc-python:1.8.2 AS orthanc-aws

MAINTAINER Sebastien Jodogne <s.jodogne@gmail.com>
LABEL Description="Orthanc, free DICOM server, with plugins, aws s3 and Python" Vendor="The Orthanc project"

# Copy plugin shared library artifact from previous stage and install missing runtime requirements
COPY --from=build-orthanc-aws /root/build/aws/libOrthancAwsS3Storage.so \
     /usr/local/share/orthanc/plugins/

# Install dependencies needed for plugin (note includes some AWS CPP SDK dependencies):
RUN apt-get -y clean && apt-get -y update
RUN DEBIAN_FRONTEND=noninteractive apt-get -y install \
    libcrypto++6 libcurl4 && \
    apt-get clean && rm -rf /var/lib/apt/lists/*

VOLUME [ "/var/lib/orthanc/db" ]
EXPOSE 4242
EXPOSE 8042

ENTRYPOINT [ "Orthanc" ]
CMD [ "/etc/orthanc/" ]

# https://groups.google.com/d/msg/orthanc-users/qWqxpvCPv8g/Z8huoA5FDAAJ
ENV MALLOC_ARENA_MAX 5
