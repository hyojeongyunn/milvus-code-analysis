# Milvus Debug Guide

This is a kind guide for developers new in milvus.  
If you follow instructions below **exactly**, you can 1. run binary milvus and 2. do unittest 3. debug milvus with log.

## Build Milvus

Hardware requirements: 
- disk spalce: at least 50GB  
- ram: at least 8GB  

Code requirement:  
- To follow below instructions, you must start with the branch named `baseline` from this public [repository](https://github.com/hyojeongyunn/milvus.git). 
    - I fixed the v2.4.0 to build & execute on docker container with ubuntu:20.04, code installed on `home` directory of the container. 
- You can find more information on this [commit](https://github.com/hyojeongyunn/milvus/commit/704a2b63b6e3d3599d52d1c9c2c4e1568ce7af14) to make it fit in to your setting.

Let's start.

1. Install code  
    ```bash
    $ git clone https://github.com/hyojeongyunn/milvus.git
    $ cd milvus
    $ git checkout -t origin/baseline
    ```

2. Set container  
    ```bash
    $ docker run -itd --name milvus -p -v {your milvus code destination}:/home --cap-add=SYS_PTRACE --privileged --security-opt seccomp=unconfined ubuntu:20.04
    ```

3. Start container  
    ```bash
    $ docker attach milvus
    $ apt-get update && apt-get install clang-format clang-tidy git g++ gcc vim sudo cmake ninja-build software-properties-common iproute2 locales gdb wget curl zip unzip tar libopenblas-dev python3.8 python3-pip xsltproc tmux -y
    $ curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
    $ . "$HOME/.cargo/env"
    ```

4. Update `cmake` version  
    Current cmake version for ubuntu:20.04 does not fit with milvus.  Milvus requires at least `3.26.4` version for cmake.  
    ```bash
    $ cd {your destination for cmake}
    $ wget https://github.com/Kitware/CMake/releases/download/v3.29.0-rc4/cmake-3.29.0-rc4.tar.gz
    $ sudo apt purge cmake
    $ tar -xvzf cmake-3.29.0-rc4.tar.gz
    $ sudo apt install qt5-default build-essential libssl-dev -y
    $ cd cmake-3.29.0-rc4
    ```
    
    ```bash
    $ ./bootstrap
    ```
    It will take about 6 minutes.

    ```bash
    $ make
    ```
    It will take about 17 minutes.

    ```bash
    $ sudo make install
    $ vim ~/.bashrc
    ```
    Append `PATH=$PATH:/usr/local/bin/` at the end of the file.  
    cf. a (insert), esc :wq (write and quit) 

    ```bash
    $ source ~/.bashrc
    $ cmake --version
    ```
    Check whether the version is >= `3.26.4`.

5. Install `go`
    ```bash
    $ cd {your destination for go}
    $ wget https://go.dev/dl/go1.22.1.linux-amd64.tar.gz
    $ tar -C /usr/local -xzf go1.22.1.linux-amd64.tar.gz
    $ vim ~/.bashrc
    ```
    Append `PATH=$PATH:/usr/local/go/bin` at the end of the file.  
    cf. a (insert), esc :wq (write and quit) 

    ```bash
    $ source ~/.bashrc
    $ go version
    ```
    Check whether `go` is installed.

6. Install dependencies for milvus
    ```bash
    $ cd /home
    $ ./scripts/install_deps.sh
    $ export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:$PWD/lib
    $ make install
    ```
    It will take <= 30 minutes.
    You must find that `bin`, `lib`, `configs` are created.

## Run Binary Milvus
This only fits for **standalone** milvus.  

1. Etcd
    ```bash
    $ cd {your etcd destination}
    $ wget https://github.com/etcd-io/etcd/releases/download/v3.5.0/etcd-v3.5.0-linux-amd64.tar.gz
    $ tar zxvf etcd-v3.5.0-linux-amd64.tar.gz
    $ cd etcd-v3.5.0-linux-amd64
    $ ./etcd -advertise-client-urls=http://127.0.0.1:2379 -listen-client-urls http://0.0.0.0:2379 --data-dir /etcd
    ```

2. MinIO
    ```bash
    $ cd {your minio destination}
    $ wget https://dl.min.io/server/minio/release/linux-amd64/minio
    $ chmod +x minio
    $ ./minio server /minio
    ```

3. Milvus
    ```bash
    $ cd /home
    $ export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:$PWD/lib
    $ ./bin/milvus run standalone
    ```

## Do unittest
Because we have to specify `LD_LIBRARY_PATH`, you can not directly click `Run Test` button on the code.
You always have to use `launch.json` file to do unittest inside your container.  
You can use `laucn.json` in this repository.  

Also, you can attach debugger (e.g., `dlv`) on unittest.  
For more information about vscode debugging, refer to ["Debugging | Visual Studio Code"](https://code.visualstudio.com/docs/editor/debugging).

## Debug Running Milvus
This is few informations I can provide to you.

- You may not attach debugger such as `dlv` or `gdb` to running milvus.
    - It's because Milvus is a multi-thread program. However, one can try to attach `gdb` and follow ["How to restrict gdb debugging to one thread at a time"](https://stackoverflow.com/questions/6721940/how-to-restrict-gdb-debugging-to-one-thread-at-a-time)
    - See this [discussion](https://github.com/milvus-io/milvus/discussions/32758) for more information.
- However, you can check and `ctrl+f` the log.  
This is the way I collected the log from Milvus.

    At terminal 1,
    ```bash
    $ script milvus_log
    $ export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:$PWD/lib
    $ ./bin/milvus run standalone
    ```

    At terminal 2,
    ```bash
    $ cat /run/milvus/standalone.pid
    $ kill -9 {pid that you retrieved}
    ```

    Go back to terminal 1,
    ```bash
    $ ctrl+d
    ```
    Use script file for debugging.


## Appendix
- Command for checking server storage
    - `du -sh *` : 1-depth usage  
    - `df . -h` : human-friendly current disk usage 

- Kill running milvus
    ```bash
    $ cat /run/milvus/standalone.pid
    $ kill -9 {pid that you retrieved}
    ```

