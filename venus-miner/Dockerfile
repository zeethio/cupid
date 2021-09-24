FROM ubuntu:focal
RUN DEBIAN_FRONTEND=noninteractive apt-get update -y -q
RUN DEBIAN_FRONTEND=noninteractive apt install mesa-opencl-icd ocl-icd-opencl-dev gcc git bzr jq pkg-config curl clang build-essential hwloc libhwloc-dev wget -y  -q && \
  apt upgrade -y -q && \

  wget -c https://golang.org/dl/go1.16.2.linux-amd64.tar.gz -O - | tar -xz -C /usr/local && \
  echo "export PATH=$PATH:/usr/local/go/bin" >> root/.bashrc 

SHELL ["/bin/bash", "-c"]
ENV PATH=$PATH:/usr/local/go/bin


WORKDIR '/app'
RUN  git clone https://github.com/zeethio/venus-miner.git && \
     cd venus-miner && \
     git checkout incubation && \
     make

ENTRYPOINT ["/app/venus-miner/venus-miner"]
