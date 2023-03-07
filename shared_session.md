

# Shared Session Build of Onnx Runtime Backend for Triton

This document describes the process for building an version of the
Onnx backend for the triton server based on the code in this repo.

The code in this repo merges a PR that implements session sharing, so that multiple instance of a model share the same onnx session, which includes the loaded model weights.

## The Share Session Pull Request

The PR with shared sessions.
https://github.com/triton-inference-server/onnxruntime_backend/pull/141

This repo merges the above PR into the 22.07 (2.24.0) version of the Onnx Runtime Backend.   We are currently using release 2.24.0 of Triton, which equates to the Triton contain image 22.07.

## Notes on Building the Onnx Backend

1. On an AWS M6I.2xlarge instance
2. Clone the Triton Server repo
      
       git clone  --branch v2.24.0 --depth 1  git@github.com:triton-inference-server/server.git
3. Follow instruction here (build.md) to build the Triton Server.  Need to build the Triton server as this creates a docker container with all the dependencies needed for building the onnx backend.  To build run:

        python ./build.py --enable-all

   Note: check as my onnx backend install seems to have fewer shared libraries bundled with it vs the 22.07 image

4. Clone the ONNX backend repo, add branch from the pull request 141, merge PR changes into release 2.24.0 (docker image 22.07).  This all lives in this fork of the onnx backend repo in branch_w_session_sharing

5. The build in step 3, means we have a docker image called `tritonserver_buildbase`, we’ll use this docker to build the onnx backend, startup the container and map the onnx_backend folder, need to map the docker demon socket for the build to work:

       docker run -it --rm -v/var/run/docker.sock:/var/run/docker.sock  -v/home/ec2-user/nta/onnxruntime_backend/:/workspace/onnxruntime_backend tritonserver_buildbase bash
6. Inside the docker container, build the onnx backend
  
       cd  onnxruntime_backend
       mkdir build
       cd build
       cmake -DCMAKE_INSTALL_PREFIX:PATH=`pwd`/install -DTRITON_BUILD_ONNXRUNTIME_VERSION=1.12.0 -DTRITON_BUILD_CONTAINER_VERSION=22.07 -DTRITON_BACKEND_REPO_TAG=r22.07 -DTRITON_CORE_REPO_TAG=r22.07 -DTRITON_COMMON_REPO_TAG=r22.07 ..
       make install

    This will build the backend and install into the folders in `build/install`

7. Now we can make use of the new backend, by mapping it over the standard backend which is part of the 22.07 Triton container.  Modify the run_inference_server.sh script to run the triton container with the new backend, see new -v mapping below:
        
        docker run --detach \
          ${CPUSET} \
          --name ${CONTAINER_NAME} \
          --volume ${NUMENTA_VOLUME}:${IS_DIR} \
          --volume ${NUMENTA_VOLUME}/models:${MODELS_DIR} \
           -v/home/ec2-user/nta/onnxruntime_backend/build/install/backends/onnxruntime/:/opt/tritonserver/backends/onnxruntime/ \
          --env CAJAL_LICENSE_FILE=${CAJAL_LICENSE_FILE} \
          --env CAJAL_NUM_THREADS=0 \
          --publish ${EXPOSE_PORT} \
          ${DOCKER_IMAGE} \
          /bin/bash ${SCRIPTS_DIR}/triton.sh

8. Some basic testing:
  - Original model config:  memory usage = 2.68 GiB
  - 1 instance per model config: memory usage = 612 MiB
  - 4 instandes per model, shared session config: memory usage = 593 MiB

## ToDo:
- add building this backend to the IS 0.3 build process
- test throughput on an instance where we fully load the cache, eg a true 4 core machine, or a much larger instance.  Issue is on a virtualised machine the CPU cache is not virtualised so if we are on a 4 core instance running on a 32 core machine we maek use of bigger caches than are realistic.
- confirm backend set of library .so files matches with those in the standard container.
  - tried this with --enable-all, still seem to have a lesser set of libraries
- push to get this merged



