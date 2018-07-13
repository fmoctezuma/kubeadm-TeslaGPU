FROM nvidia/cuda-ppc64le:8.0-devel-ubuntu16.04

RUN apt-get update && apt-get install -y --no-install-recommends \
        cuda-samples-$CUDA_PKG_VERSION && \
    rm -rf /var/lib/apt/lists/*

WORKDIR /usr/local/cuda/samples/0_Simple/vectorAdd
RUN make

CMD ./vectorAdd
