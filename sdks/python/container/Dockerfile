###############################################################################
#  Licensed to the Apache Software Foundation (ASF) under one
#  or more contributor license agreements.  See the NOTICE file
#  distributed with this work for additional information
#  regarding copyright ownership.  The ASF licenses this file
#  to you under the Apache License, Version 2.0 (the
#  "License"); you may not use this file except in compliance
#  with the License.  You may obtain a copy of the License at
#
#      http://www.apache.org/licenses/LICENSE-2.0
#
#  Unless required by applicable law or agreed to in writing, software
#  distributed under the License is distributed on an "AS IS" BASIS,
#  WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
#  See the License for the specific language governing permissions and
# limitations under the License.
###############################################################################

ARG py_version
FROM python:"${py_version}"-buster
MAINTAINER "Apache Beam <dev@beam.apache.org>"

ARG pull_licenses

# Install native bindings required for dependencies.
RUN apt-get update && \
    apt-get install -y \
       # Required by python-snappy
       libsnappy-dev \
       # Required by pyyaml (for c bindings)
       libyaml-dev \
       # This is used to speed up the re-installation of the sdk.
       ccache \
       && \
    rm -rf /var/lib/apt/lists/*

####
# Install required packages for Beam Python SDK and common dependencies used by users.
####

# SDK dependencies not listed in base_image_requirements.txt will be installed
# when we install SDK with pip below.
COPY target/base_image_requirements.txt /tmp/base_image_requirements.txt
RUN \
    pip install -r /tmp/base_image_requirements.txt && \
    python -c "import nltk; nltk.download('stopwords')" && \
    rm /root/nltk_data/corpora/stopwords.zip && \
    # Check that the fast implementation of protobuf is used.
    python -c "from google.protobuf.internal import api_implementation; assert api_implementation._default_implementation_type == 'cpp'; print ('Verified fast protobuf used.')" && \
    # Remove pip cache.
    rm -rf /root/.cache/pip && \
    rm -rf /tmp/base_image_requirements.txt

# Install Google Cloud SDK.
ENV CLOUDSDK_CORE_DISABLE_PROMPTS yes
ENV PATH $PATH:/usr/local/gcloud/google-cloud-sdk/bin
RUN mkdir -p /usr/local/gcloud && \
    cd /usr/local/gcloud && \
    curl -s -O https://dl.google.com/dl/cloudsdk/release/google-cloud-sdk.tar.gz && \
    tar -xf google-cloud-sdk.tar.gz && \
    /usr/local/gcloud/google-cloud-sdk/install.sh && \
    rm google-cloud-sdk.tar.gz

# Configure ccache prior to installing Beam SDK.
RUN ln -s /usr/bin/ccache /usr/local/bin/gcc
# These parameters are needed as pip compiles artifacts in random temporary directories.
RUN ccache --set-config=sloppiness=file_macro && ccache --set-config=hash_dir=false

####
# Install Apache Beam SDK
####
COPY target/apache-beam.tar.gz /opt/apache/beam/tars/
RUN pip install -v /opt/apache/beam/tars/apache-beam.tar.gz[gcp]

COPY target/license_scripts /tmp/license_scripts/

# Add golang licenses. Because the go-license directory may be empty if
# pull_licenses is false, and COPY fails if there are no files,
# copy an extra LICENSE file then remove it.
COPY target/LICENSE target/go-licenses/* /opt/apache/beam/third_party_licenses/golang/
RUN rm /opt/apache/beam/third_party_licenses/golang/LICENSE

COPY target/license_scripts /tmp/license_scripts/
RUN if [ "$pull_licenses" = "true" ] ; then \
    pip install 'pip-licenses<3.0.0;python_version>="3.5"' && \
    pip install 'pip-licenses==1.18.0;python_version<="2.7"' && \
    python /tmp/license_scripts/pull_licenses_py.py && \
    pip uninstall -y pip-licenses && \
    # Remove pip cache.
    rm -rf /root/.cache/pip ; \
   else \
    # Remove above golang license and dir if pull licenses false
    rm -rf /opt/apache/beam/third_party_licenses ; \
   fi

# Remove license scripts
RUN rm -rf /tmp/license_scripts

# Log complete list of what exact packages and versions are installed.
RUN pip freeze --all
# Make sure there are no conflicting dependencies.
RUN pip check


COPY target/LICENSE /opt/apache/beam/
COPY target/LICENSE.python /opt/apache/beam/
COPY target/NOTICE /opt/apache/beam/

ADD target/launcher/linux_amd64/boot /opt/apache/beam/
ENTRYPOINT ["/opt/apache/beam/boot"]
