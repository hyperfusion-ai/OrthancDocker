##############################################################################
### First stage: build the Orthanc aws plugin ###############################
##############################################################################
# From https://book.orthanc-server.com/plugins/object-storage.html#id1
# For testing use: docker build --target build-orthanc-aws -t build-orthanc-aws -f Dockerfile .
FROM jodogne/orthanc-python:1.7.3 AS build-orthanc-aws

# git is needed for installing aws-sdk-cpp's dependencies (aws-c-common, aws-checksums, aws-c-event-stream)
#  libboost-filesystem-dev libboost-thread-dev libboost-system-dev libboost-regex-dev libboost-date-time-dev libboost-atomic-dev libcrypto++-dev 
RUN apt-get -y clean && apt-get -y update
RUN DEBIAN_FRONTEND=noninteractive apt-get -y install \
    wget unzip cmake git build-essential libssl-dev zlib1g-dev libcurl4-openssl-dev \
    mercurial uuid-dev && \
    apt-get clean && rm -rf /var/lib/apt/lists/*


# Compile and install Osimis AWS plugin using vcpkg, don't use system jsoncpp:
# Install AWS C++ SDK:
# More background on https://docs.aws.amazon.com/sdk-for-cpp/v1/developer-guide/setup.html
RUN mkdir ~/aws
RUN cd ~/aws && wget https://github.com/aws/aws-sdk-cpp/archive/1.8.42.zip && unzip 1.8.42.zip && mv aws-sdk-cpp-1.8.42 aws-sdk-cpp
RUN mkdir -p ~/aws/builds/aws-sdk-cpp && cd ~/aws/builds/aws-sdk-cpp && cmake -DBUILD_ONLY="s3" -DENABLE_TESTING=OFF -DBUILD_SHARED_LIBS=OFF ~/aws/aws-sdk-cpp && make -j 16 && make install
RUN apt-get -y clean && apt-get -y update

RUN DEBIAN_FRONTEND=noninteractive apt-get -y install curl zip unzip tar
RUN git clone https://github.com/microsoft/vcpkg && ./vcpkg/bootstrap-vcpkg.sh && ./vcpkg/vcpkg install cryptopp
RUN cd ~ && hg clone https://hg.orthanc-server.com/orthanc-object-storage

RUN mkdir -p ~/build/aws
RUN apt-get -y update 
RUN DEBIAN_FRONTEND=noninteractive apt-get -y install \
      libboost-all-dev
#      libboost-filesystem-dev libboost-thread-dev libboost-system-dev libboost-regex-dev libboost-date-time-dev libboost-atomic-dev
COPY ./orthanc-object-storage_CMakeLists-awssdk-static-vcpkg.patch /root/orthanc-object-storage
COPY ./orthanc-object-storage_AwsS3StoragePlugin-ca.patch /root/orthanc-object-storage
RUN cd ~/orthanc-object-storage && patch -p1 < orthanc-object-storage_CMakeLists-awssdk-static-vcpkg.patch && patch -p1 < orthanc-object-storage_AwsS3StoragePlugin-ca.patch
RUN cd ~/build/aws && cmake -DSTATIC_BUILD=ON -DCMAKE_TOOLCHAIN_FILE=../../vcpkg/scripts/buildsystems/vcpkg.cmake -DUSE_SYSTEM_GOOGLE_TEST=OFF -DALLOW_DOWNLOADS=ON -DUSE_SYSTEM_JSONCPP=OFF -DUSE_LEGACY_JSONCPP=ON -DORTHANC_FRAMEWORK_VERSION=1.7.3 ../../orthanc-object-storage/Aws && make -j16
RUN cp ~/build/aws/libOrthancAwsS3Storage.so /usr/local/share/orthanc/plugins

# # Compile and install Osimis AWS plugin using vcpkg, patching to use jsoncpp from vcpkg as well
# RUN apt-get -y clean && apt-get -y update
# RUN DEBIAN_FRONTEND=noninteractive apt-get -y install curl zip unzip tar
# RUN git clone https://github.com/microsoft/vcpkg && ./vcpkg/bootstrap-vcpkg.sh && ./vcpkg/vcpkg install cryptopp && ./vcpkg/vcpkg install jsoncpp
# COPY ./orthanc-object-storage_Aws_CMakeLists.txt.patch /root/
# RUN cd ~ && hg clone https://hg.orthanc-server.com/orthanc-object-storage && patch -p0 < orthanc-object-storage_Aws_CMakeLists.txt.patch
# RUN mkdir -p ~/build/aws
# RUN apt-get -y update 
# RUN DEBIAN_FRONTEND=noninteractive apt-get -y install \
#      libboost-filesystem-dev libboost-thread-dev libboost-system-dev libboost-regex-dev libboost-date-time-dev libboost-atomic-dev
# RUN cd ~/build/aws && cmake -DCMAKE_TOOLCHAIN_FILE=../../vcpkg/scripts/buildsystems/vcpkg.cmake -DUSE_SYSTEM_GOOGLE_TEST=OFF -DALLOW_DOWNLOADS=ON ../../orthanc-object-storage/Aws && \
# sed -i 's/-ljsoncpp//g' CMakeFiles/UnitTests.dir/link.txt && sed -i 's/-ljsoncpp//g' CMakeFiles/OrthancAwsS3Storage.dir/link.txt && make -j16
# RUN cp ~/build/aws/libOrthancAwsS3Storage.so /usr/local/share/orthanc/plugins

# # Compile and install Osimis AWS plugin. Don't use vcpkg. Use libcrypto from package manager (?)
# RUN apt-get -y clean && apt-get -y update
# RUN DEBIAN_FRONTEND=noninteractive apt-get -y install \
#     libboost-filesystem-dev libboost-thread-dev libboost-system-dev libboost-regex-dev libboost-date-time-dev libboost-atomic-dev libcrypto++-dev
# RUN cd ~ && hg clone https://hg.orthanc-server.com/orthanc-object-storage
# RUN mkdir -p ~/build/aws
# # Note only building a static library appears to work at the moment, see https://groups.google.com/g/orthanc-users/c/AZ804k8ba1A/m/kajtEdKUBAAJ
# RUN cd ~/build/aws && cmake -DSTATIC_BUILD=ON -DCMAKE_BUILD_TYPE=Release -DUSE_VCPKG_PACKAGES=OFF -DALLOW_DOWNLOADS=ON -DUSE_SYSTEM_GOOGLE_TEST=OFF -DENABLE_GOOGLE_TEST=OFF -DUSE_SYSTEM_JSONCPP=OFF -DORTHANC_FRAMEWORK_VERSION=1.7.3 ../../orthanc-object-storage/Aws && make -j4
# RUN cp ~/build/aws/libOrthancAwsS3Storage.so /usr/local/share/orthanc/plugins


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
    libcurl4-openssl-dev && \
    apt-get clean && rm -rf /var/lib/apt/lists/*

VOLUME [ "/var/lib/orthanc/db" ]
EXPOSE 4242
EXPOSE 8042

ENTRYPOINT [ "Orthanc" ]
CMD [ "/etc/orthanc/" ]

# https://groups.google.com/d/msg/orthanc-users/qWqxpvCPv8g/Z8huoA5FDAAJ
ENV MALLOC_ARENA_MAX 5
