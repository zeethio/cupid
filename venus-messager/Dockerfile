FROM ubuntu:focal
RUN DEBIAN_FRONTEND=noninteractive apt-get update -y -q
RUN DEBIAN_FRONTEND=noninteractive apt install mesa-opencl-icd ocl-icd-opencl-dev gcc git bzr jq pkg-config curl clang build-essential hwloc libhwloc-dev wget dnsutils iputils-ping iproute2 -y  -q && \
  apt upgrade -y -q && \

  wget -c https://golang.org/dl/go1.16.2.linux-amd64.tar.gz -O - | tar -xz -C /usr/local && \
  echo "export PATH=$PATH:/usr/local/go/bin" >> root/.bashrc 

SHELL ["/bin/bash", "-c"]
ENV PATH=$PATH:/usr/local/go/bin


WORKDIR '/app'
RUN  git clone --branch v1.2.0 https://github.com/filecoin-project/venus-messager.git && \
     cd venus-messager && \
     make

EXPOSE 39812
ENTRYPOINT ["/app/venus-messager/venus-messager"]
