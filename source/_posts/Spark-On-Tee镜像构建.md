---
title: Spark-On-Tee镜像构建
date: 2025-04-16 16:14:59
tags:
    - Spark
    - TEE
    - Occlum
category: 隐私计算
---

# 1. 介绍

## 1.1 方案介绍

Spark作为一个流行的通用大数据计算框架，能让开发者快速高效的编写ETL、ML任务代码，但在机密计算的场景下，与运行在REE环境中的Spark任务相比较，主要有以下几点差异:
- 运行环境: 需要运行在支持TEE的硬件设备上且使用对应的TEE技术，如使用Intel SGX技术，应用需要基于SGX的应用架构模式开发，才能保证应用的内容不可窥探。
- 数据加密: 各方数据都为加密状态，防止传输态和存储态的数据安全，而在TEE环境中，Spark程序需要解密进行数据处理。
-  远程认证: 需要运行的Spark应用确实是运行在TEE应用，且是我们信任的程序，只有信任的程序能获得密钥，解密数据后使用

使用SGX SDK去重构Spark显然不现实，我们借助Occlum实现的LibOS，能在TEE Enclave中运行一个虚拟的OS，Occlum在这个虚拟的OS内实现各种系统调用，在安全区和非安全全之间进行交互。在这个虚拟的OS内运行用户进程（Spark本质是一个Java虚拟机程序），则无需关心“底层”的TEE环境，只需要关心自己的业务实现。

而远程认证和KMS我们选择使用蚂蚁的AECS，Occlum本身已经实现了一套[init_ra](https://occlum.readthedocs.io/en/latest/init_ra.html)流程,可以使用AECS进行远程认证和密钥获取，这个流程在虚拟OS执行用户进程前已经完成，用户无需关心是如何进行认证和获取密钥的，只需要在指定的文件中获取密钥使用即可。

构建方案参考Intel [BIGDL-PPML]()的构建流程，区别是使用了AECS进行init_ra。
<!-- more -->


## 1.2 目录介绍
![](spark_build_catelog.png)
- production/Dockerfile : 构建生产环境能使用镜像的Dockerfile
- production/run_spark_on_occlum_glibc.sh 包含Occlum应用的制作脚本，Occlum使用AECS进行init_ra也是修改此脚本
-  build_production_image.sh: 构建生产环境镜像脚本
-  customer/Dockerfile: 构建真正运行在客户环境镜像的Dockerfile
- build_customer_image.sh: 构建客户环境镜像脚本

> production和customer的区别，production的镜像，实际已经满足运行需求，可以用于客户环境中，但因存在一些无关的数据和历史的Docker Layer，整体的镜像大小比较大，有15G以上，不利于传输使用。因此，再进行一步customer打包，把production镜像中有用的文件（实际就是occlum build完的加密文件，即虚拟OS的文件）拷贝到新的镜像中，这样制作而成的镜像，大小只有5G。


# 2. 准备
## 2.1 Docker
需要安装Docker用于镜像生成。

## 2.2 Intel基础镜像
```
docker pull intelanalytics/bigdl-ppml-trusted-big-data-ml-scala-occlum-production:2.4.0
```

## 2.3 image_key
image_key是Occlum用于加密“镜像”的密钥，这里的“镜像”可以理解为虚拟的OS中所需要使用的数据和程序，而非最终打包生成的Docker镜像。image_key是程序开发者生成的，最终也会提交到AECS中，基于Occlum构建的Spark TEE应用启动时，会进行认证，认证通过可以拿到image_key,解密“镜像”文件，再运行用户进程。


```
#生成 image_key
docker run -it intelanalytics/bigdl-ppml-trusted-big-data-ml-scala-occlum-production:2.3.0 bash
cd /opt/occlum_spark
occlum gen-image-key image_key
 
#查看 image_key
cat ./image_key
 
#退出容器
exit
```

示例如下
```
48-f1-87-6c-da-13-be-ae-2f-14-f6-78-9e5-15-1d-e1
```

将image_key的内容填写到`production/image_key`中。


# 3. 构建
![](Spark镜像构建流程.excalidraw.png)
## 3.1 production
主要完成
- Occlum init -> 生成image文件夹
- 将相关文件（如Spark）拷贝到image文件夹中
- Occlum build -> 生成build文件夹

### 3.1.1 build_production_image.sh
```
export SGX_MEM_SIZE=32GB
export SGX_THREAD=512
export SGX_HEAP=512MB
export SGX_KERNEL_HEAP=1GB
export ENABLE_SGX_DEBUG=true
export ATTESTATION=true
export USING_TMP_HOSTFS=false
export TAG=2.4.0-SNAPSHOT-ml-test-032803

docker build \
    --build-arg SGX_MEM_SIZE=$SGX_MEM_SIZE \
    --build-arg SGX_THREAD=$SGX_THREAD \
    --build-arg SGX_HEAP=$SGX_HEAP \
    --build-arg SGX_KERNEL_HEAP=$SGX_KERNEL_HEAP \
    --build-arg ENABLE_SGX_DEBUG=$ENABLE_SGX_DEBUG \
    --build-arg ATTESTATION=$ATTESTATION \
    --build-arg USING_TMP_HOSTFS=$USING_TMP_HOSTFS \
    -t demo/bigdl-ppml-trusted-big-data-ml-scala-occlum-production-build:${TAG} -f ./production/Dockerfile ./production
```
设置SGX相关参数即镜像命名

### 3.1.2 production/Dockerfile
```
FROM intelanalytics/bigdl-ppml-trusted-big-data-ml-scala-occlum-production:2.4.0-SNAPSHOT
maintainer "hezhichao93@163.com"

ARG SGX_MEM_SIZE
ARG SGX_THREAD
ARG SGX_HEAP
ARG SGX_KERNEL_HEAP
ARG ENABLE_SGX_DEBUG
ARG ATTESTATION
ARG USING_TMP_HOSTFS

ENV SGX_MEM_SIZE=${SGX_MEM_SIZE}
ENV SGX_THREAD=${SGX_THREAD}
ENV SGX_HEAP=${SGX_HEAP}
ENV SGX_KERNEL_HEAP=${SGX_KERNEL_HEAP}
ENV ENABLE_SGX_DEBUG=${ENABLE_SGX_DEBUG}
ENV ATTESTATION=${ATTESTATION}
ENV USING_TMP_HOSTFS=${USING_TMP_HOSTFS}

COPY run_spark_on_occlum_glibc.sh /opt/
COPY image_key /opt/occlum_conf/image_key
COPY init_ra_conf.json /opt/occlum_conf/init_ra_conf.json
RUN chmod +x /opt/run_spark_on_occlum_glibc.sh



## s3 support，intelanalytics/bigdl-ppml-trusted-big-data-ml-scala-occlum-production:2.4.0-SNAPSHOT里的spark版本为3.1.3
COPY hadoop-aws-3.2.0.jar /opt/spark/jars/
COPY aws-java-sdk-bundle-1.11.375.jar /opt/spark/jars/
COPY verification-listener.jar /opt/spark/jars/


## occlum init
RUN bash /opt/run_spark_on_occlum_glibc.sh init
RUN rm -rf /opt/occlum_conf/image_key



## 挂载entrypoint.sh
COPY entrypoint.sh /opt/entrypoint.sh
RUN chmod +x /opt/entrypoint.sh

ENTRYPOINT [ "/opt/entrypoint.sh" ]
```

如果运行的PySpark需要额外的类库支持，可以在此文件中pip安装依赖，如

```
## 安装依赖包
RUN /opt/python-occlum/bin/pip3 install -i https://mirrors.tuna.tsinghua.edu.cn/pypi/web/simple pandas numpy matplotlib sqlalchemy lightgbm scikit-learn seaborn scipy tqdm &&  apt-get -y  install libomp-dev

```

### 3.1.3 run_spark_on_occlum_glibc.sh
```
!/bin/bash
set -x

BLUE='\033[1;34m'
NC='\033[0m'
occlum_glibc=/opt/occlum/glibc/lib
# occlum-node IP
HOST_IP=`cat /etc/hosts | grep $HOSTNAME | awk '{print $1}'`

check_sgx_dev() {
    if [ -c "/dev/sgx/enclave" ]; then
        echo "/dev/sgx/enclave is ready"
    elif [ -c "/dev/sgx_enclave" ]; then
        echo "/dev/sgx/enclave not ready, try to link to /dev/sgx_enclave"
        mkdir -p /dev/sgx
        ln -s /dev/sgx_enclave /dev/sgx/enclave
    else
        echo "both /dev/sgx/enclave /dev/sgx_enclave are not ready, please check the kernel and driver"
    fi

    if [ -c "/dev/sgx/provision" ]; then
        echo "/dev/sgx/provision is ready"
    elif [ -c "/dev/sgx_provision" ]; then
        echo "/dev/sgx/provision not ready, try to link to /dev/sgx_provision"
        mkdir -p /dev/sgx
        ln -s /dev/sgx_provision /dev/sgx/provision
    else
        echo "both /dev/sgx/provision /dev/sgx_provision are not ready, please check the kernel and driver"
    fi

    ls -al /dev/sgx
}

init_instance() {
    # check and fix sgx device
    check_sgx_dev
    # Init Occlum instance
    cd /opt
    # check if occlum_spark exists
    [[ -d occlum_spark ]] || mkdir occlum_spark
    cd occlum_spark
    occlum init --init-ra aecs
    cp -f ../occlum_conf/init_ra_conf.json .
    new_json="$(jq '.resource_limits.user_space_size = "SGX_MEM_SIZE" |
        .resource_limits.max_num_of_threads = "SGX_THREAD" |
        .process.default_heap_size = "SGX_HEAP" |
        .metadata.debuggable = "ENABLE_SGX_DEBUG" |
        .resource_limits.kernel_space_heap_size="SGX_KERNEL_HEAP" |
        .resource_limits.kernel_space_heap_max_size="SGX_KERNEL_HEAP" |
        .entry_points = [ "/usr/lib/jvm/java-8-openjdk-amd64/bin", "/bin" ] |
        .env.untrusted = [ "MALLOC_ARENA_MAX", "ATTESTATION_DEBUG", "DMLC_TRACKER_URI", "SPARK_DRIVER_URL", "SPARK_TESTING" , "_SPARK_AUTH_SECRET" ] |
        .env.default = [ "OCCLUM=yes","PYTHONHOME=/opt/python-occlum","LD_LIBRARY_PATH=/usr/lib/jvm/java-8-openjdk-amd64/lib/server:/usr/lib/jvm/java-8-openjdk-amd64/lib:/usr/lib/jvm/java-8-openjdk-amd64/../lib:/lib","SPARK_CONF_DIR=/opt/spark/conf","SPARK_ENV_LOADED=1","PYTHONHASHSEED=0","SPARK_HOME=/opt/spark","SPARK_SCALA_VERSION=2.12","SPARK_JARS_DIR=/opt/spark/jars","LAUNCH_CLASSPATH=/bin/jars/*",""]' Occlum.json)" && \
    echo "${new_json}" > Occlum.json
    echo "SGX_MEM_SIZE ${SGX_MEM_SIZE}"

    # add mount conf and mkdir source mount files
    bash add_conf.sh

    #copy python lib and attestation lib
    copy_bom -f /opt/python-glibc.yaml --root image --include-dir /opt/occlum/etc/template

    ## 测试机器学习类库
	# enable tmp hostfs
    # --conf spark.executorEnv.USING_TMP_HOSTFS=true \
    if [[ $USING_TMP_HOSTFS == "true" ]]; then
        echo "use tmp hostfs"
        mkdir ./shuffle
        edit_json="$(cat Occlum.json | jq '.mount+=[{"target": "/tmp","type": "hostfs","source": "./shuffle"}]')" && \
        echo "${edit_json}" > Occlum.json
    fi

    edit_json="$(cat Occlum.json | jq '.mount+=[{"target": "/etc","type": "hostfs","source": "/etc"}]')" && \
    echo "${edit_json}" > Occlum.json


    if [[ -z "$SGX_MEM_SIZE" ]]; then
        sed -i "s/SGX_MEM_SIZE/20GB/g" Occlum.json
    else
        sed -i "s/SGX_MEM_SIZE/${SGX_MEM_SIZE}/g" Occlum.json
    fi

    if [[ -z "$SGX_THREAD" ]]; then
        sed -i "s/\"SGX_THREAD\"/512/g" Occlum.json
    else
        sed -i "s/\"SGX_THREAD\"/${SGX_THREAD}/g" Occlum.json
    fi

    if [[ -z "$SGX_HEAP" ]]; then
        sed -i "s/SGX_HEAP/512MB/g" Occlum.json
    else
        sed -i "s/SGX_HEAP/${SGX_HEAP}/g" Occlum.json
    fi

    if [[ -z "$SGX_KERNEL_HEAP" ]]; then
        sed -i "s/SGX_KERNEL_HEAP/1GB/g" Occlum.json
    else
        sed -i "s/SGX_KERNEL_HEAP/${SGX_KERNEL_HEAP}/g" Occlum.json
    fi

    # check attestation setting
    if [ -z "$ATTESTATION" ]; then
        echo "[INFO] Attestation is disabled!"
        ATTESTATION="false"
    fi

    if [[ $ATTESTATION == "true" ]]; then
           cd /root/demos/remote_attestation/dcap/
           #build .c file
           bash ./get_quote_on_ppml.sh
           cd /opt/occlum_spark
           # dir need to exit when writing quote
           mkdir -p /opt/occlum_spark/image/etc/occlum_attestation/
           mkdir -p /etc/occlum_attestation/
           #copy bom to generate quote
           copy_bom -f /root/demos/remote_attestation/dcap/dcap-ppml.yaml --root image --include-dir /opt/occlum/etc/template
    fi

    #check glic ENV MALLOC_ARENA_MAX for docker
    if [[ -z "$MALLOC_ARENA_MAX" ]]; then
        echo "No MALLOC_ARENA_MAX specified, set to 1."
        export MALLOC_ARENA_MAX=1
    fi

    # ENABLE_SGX_DEBUG
    export ENABLE_SGX_DEBUG=true


    sed -i "s/\"ENABLE_SGX_DEBUG\"/$ENABLE_SGX_DEBUG/g" Occlum.json
    sed -i "s/#USE_SECURE_CERT=FALSE/USE_SECURE_CERT=FALSE/g" /etc/sgx_default_qcnl.conf
}

build_spark() {
    # Copy K8s secret
    mkdir -p image/var/run/secrets/
    cp -r /var/run/secrets/* image/var/run/secrets/

    #copy libs for attest quote in occlum
    cp -f /opt/occlum_spark/image/lib/libgomp.so.1 /opt/occlum_spark/image/opt/occlum/glibc/lib --remove-destination
    cp -f /opt/occlum_spark/image/lib/libc.so /opt/occlum_spark/image/opt/occlum/glibc/lib --remove-destination
    rm image/lib/*
    cp -f /usr/lib/x86_64-linux-gnu/*sgx* /opt/occlum_spark/image/opt/occlum/glibc/lib --remove-destination
    cp -f /usr/lib/x86_64-linux-gnu/*dcap* /opt/occlum_spark/image/opt/occlum/glibc/lib --remove-destination
    cp -f /usr/lib/x86_64-linux-gnu/libcrypt.so.1 /opt/occlum_spark/image/opt/occlum/glibc/lib --remove-destination

    # copy spark and bigdl and others dependencies
    copy_bom -f /opt/spark.yaml --root image --include-dir /opt/occlum/etc/template

    # Build
    if [ -f ../occlum_conf/image_key ]; then
        occlum build --image-key ../occlum_conf/image_key
    else
        occlum build
    fi
}

attestation_init() {
    #occlum build done
    # make source mount file exit to avoid occlum mout fail
    cd /opt/occlum_spark
    bash /opt/mount.sh

    # check occlum log level for docker
    export OCCLUM_LOG_LEVEL=off
    if [[ -z "$SGX_LOG_LEVEL" ]]; then
        echo "No SGX_LOG_LEVEL specified, set to off."
    else
        echo "Set SGX_LOG_LEVEL to $SGX_LOG_LEVEL"
        if [[ $SGX_LOG_LEVEL == "debug" ]] || [[ $SGX_LOG_LEVEL == "trace" ]]; then
            export OCCLUM_LOG_LEVEL=$SGX_LOG_LEVEL
        fi
    fi

    #attestation
    if [[ $ATTESTATION == "true" ]]; then
        if [[ $PCCS_URL == "" ]]; then
            echo "[ERROR] Attestation set to /root/demos/remote_attestation/dcaprue but NO PCCS"
            exit 1
        else
                echo 'PCCS_URL='${PCCS_URL}'/sgx/certification/v3/' > /etc/sgx_default_qcnl.conf
                echo 'USE_SECURE_CERT=FALSE' >> /etc/sgx_default_qcnl.conf
        fi
    fi
}

run_pyspark_pi() {
    attestation_init
    cd /opt/occlum_spark
    echo -e "${BLUE}occlum run pyspark Pi${NC}"
    occlum run /usr/lib/jvm/java-8-openjdk-amd64/bin/java \
                -XX:-UseCompressedOops \
                -XX:ActiveProcessorCount=4 \
                -Divy.home="/tmp/.ivy" \
                -Dos.name="Linux" \
                -Djdk.lang.Process.launchMechanism=vfork \
                -cp "$SPARK_HOME/conf/:$SPARK_HOME/jars/*" \
                -Xmx512m org.apache.spark.deploy.SparkSubmit \
                /py-examples/pi.py
}

run_pyspark_sql_example() {
    attestation_init
    cd /opt/occlum_spark
    echo -e "${BLUE}occlum run pyspark SQL example${NC}"
    occlum run /usr/lib/jvm/java-8-openjdk-amd64/bin/java \
                -XX:-UseCompressedOops \
                -XX:ActiveProcessorCount=4 \
                -Divy.home="/tmp/.ivy" \
                -Dos.name="Linux" \
                -Djdk.lang.Process.launchMechanism=vfork \
                -cp "$SPARK_HOME/conf/:$SPARK_HOME/jars/*" \
                -Xmx3g org.apache.spark.deploy.SparkSubmit \
                /py-examples/sql_example.py
}

run_pyspark_sklearn_example() {
    attestation_init
    cd /opt/occlum_spark
    echo -e "${BLUE}occlum run pyspark sklearn example${NC}"
    occlum run /usr/lib/jvm/java-8-openjdk-amd64/bin/java \
                -XX:-UseCompressedOops \
                -XX:ActiveProcessorCount=4 \
                -Divy.home="/tmp/.ivy" \
                -Dos.name="Linux" \
                -Djdk.lang.Process.launchMechanism=vfork \
                -cp "$SPARK_HOME/conf/:$SPARK_HOME/jars/*" \
                -Xmx3g org.apache.spark.deploy.SparkSubmit \
                /py-examples/sklearn_example.py
}

run_pyspark_tpch_example() {
    attestation_init
    cd /opt/occlum_spark
    echo -e "${BLUE}occlum run pyspark tpch example${NC}"
    occlum run /usr/lib/jvm/java-8-openjdk-amd64/bin/java \
                -XX:-UseCompressedOops \
                -XX:ActiveProcessorCount=4 \
                -Divy.home="/tmp/.ivy" \
                -Dos.name="Linux" \
                -Djdk.lang.Process.launchMechanism=vfork \
                -cp "$SPARK_HOME/conf/:$SPARK_HOME/jars/*" \
                -Xmx5g org.apache.spark.deploy.SparkSubmit \
                --conf spark.sql.shuffle.partitions=8 \
                --py-files /py-examples/tpch/tpch.zip \
                /py-examples/tpch/main.py \
                /host/data/ /host/data/output/ true
}

run_spark_pi() {
    attestation_init
    echo -e "${BLUE}occlum run spark Pi${NC}"
    occlum run /usr/lib/jvm/java-8-openjdk-amd64/bin/java \
                -XX:-UseCompressedOops \
                -XX:ActiveProcessorCount=4 \
                -Divy.home="/tmp/.ivy" \
                -Dos.name="Linux" \
                -cp "$SPARK_HOME/conf/:$SPARK_HOME/jars/*" \
                -Xmx512m org.apache.spark.deploy.SparkSubmit \
                --jars $SPARK_HOME/examples/jars/spark-examples_2.12-${SPARK_VERSION}.jar,$SPARK_HOME/examples/jars/scopt_2.12-3.7.1.jar \
                --class org.apache.spark.examples.SparkPi spark-internal
}

run_spark_unittest() {
    attestation_init
    echo -e "${BLUE}occlum run spark unit test ${NC}"
    run_spark_unittest_only
}

run_spark_unittest_only() {
    export SPARK_TESTING=1
    cd /opt/occlum_spark
    mkdir -p data/olog
    echo -e "${BLUE}occlum run spark unit test only ${NC}"
    occlum start
    for suite in `cat /opt/sqlSuites`
    do occlum exec /usr/lib/jvm/java-8-openjdk-amd64/bin/java -Xmx24g \
                -Divy.home="/tmp/.ivy" \
                -Dos.name="Linux" \
		-Djdk.lang.Process.launchMechanism=posix_spawn \
	        -Dspark.testing=true \
	        -Dspark.test.home=/opt/spark-source \
	        -Dspark.python.use.daemon=false \
	        -Dspark.python.worker.reuse=false \
	        -Dspark.driver.host=127.0.0.1 \
	        -cp "$SPARK_HOME/conf/:$SPARK_HOME/jars/*:$SPARK_HOME/test-jars/*:$SPARK_HOME/test-classes/"  \
	        org.scalatest.tools.Runner \
	        -s ${suite} \
	        -fF /host/data/olog/${suite}.txt
    done
	        #-Dspark.sql.warehouse.dir=hdfs://localhost:9000/111-spark-warehouse \
    occlum stop
}

run_spark_lenet_mnist(){
    attestation_init
    echo -e "${BLUE}occlum run BigDL lenet mnist{NC}"
    echo -e "${BLUE}logfile=$log${NC}"
    occlum run /usr/lib/jvm/java-8-openjdk-amd64/bin/java \
                -XX:-UseCompressedOops \
                -XX:ActiveProcessorCount=4 \
                -Divy.home="/tmp/.ivy" \
                -Dos.name="Linux" \
                -cp "$SPARK_HOME/conf/:$SPARK_HOME/jars/*:/bin/jars/*" \
                -Xmx10g org.apache.spark.deploy.SparkSubmit \
                --master 'local[4]' \
                --conf spark.driver.port=10027 \
                --conf spark.scheduler.maxRegisteredResourcesWaitingTime=5000000 \
                --conf spark.worker.timeout=600 \
                --conf spark.starvation.timeout=250000 \
                --conf spark.rpc.askTimeout=600 \
                --conf spark.blockManager.port=10025 \
                --conf spark.driver.host=127.0.0.1 \
                --conf spark.driver.blockManager.port=10026 \
                --conf spark.io.compression.codec=lz4 \
                --class com.intel.analytics.bigdl.dllib.models.lenet.Train \
                --driver-memory 10G \
                /bin/jars/bigdl-dllib-spark_${SPARK_VERSION}-${BIGDL_VERSION}.jar \
                -f /host/data \
                $* | tee spark.local.sgx.log
}

run_spark_resnet_cifar(){
    attestation_init
    echo -e "${BLUE}occlum run BigDL Resnet Cifar10${NC}"
    occlum run /usr/lib/jvm/java-8-openjdk-amd64/bin/java \
                -XX:-UseCompressedOops \
                -XX:ActiveProcessorCount=4 \
                -Divy.home="/tmp/.ivy" \
                -Dos.name="Linux" \
                -cp "$SPARK_HOME/conf/:$SPARK_HOME/jars/*:/bin/jars/*" \
                -Xmx10g org.apache.spark.deploy.SparkSubmit \
                --master 'local[4]' \
                --conf spark.driver.port=10027 \
                --conf spark.scheduler.maxRegisteredResourcesWaitingTime=5000000 \
                --conf spark.worker.timeout=600 \
                --conf spark.starvation.timeout=250000 \
                --conf spark.rpc.askTimeout=600 \
                --conf spark.blockManager.port=10025 \
                --conf spark.driver.host=127.0.0.1 \
                --conf spark.driver.blockManager.port=10026 \
                --conf spark.io.compression.codec=lz4 \
                --class com.intel.analytics.bigdl.dllib.models.resnet.TrainCIFAR10 \
                --driver-memory 10G \
                /bin/jars/bigdl-dllib-spark_${SPARK_VERSION}-${BIGDL_VERSION}.jar \
                -f /host/data \
                $* | tee spark.local.sgx.log
}

run_spark_tpch(){
    attestation_init
    echo -e "${BLUE}occlum run BigDL spark tpch${NC}"
    occlum run /usr/lib/jvm/java-8-openjdk-amd64/bin/java \
                -XX:-UseCompressedOops \
                -XX:ActiveProcessorCount=4 \
                -Divy.home="/tmp/.ivy" \
                -Dos.name="Linux" \
                -cp "$SPARK_HOME/conf/:$SPARK_HOME/jars/*:/bin/jars/*" \
                -Xmx5g -Xms5g \
                org.apache.spark.deploy.SparkSubmit \
                --conf spark.sql.shuffle.partitions=8 \
                --master 'local[4]' \
                --class com.intel.analytics.bigdl.ppml.examples.tpch.TpchQuery \
                --verbose \
                /bin/jars/spark-tpc-h-queries_2.12-1.0.jar \
                /host/data /host/data/output plain_text plain_text
}

run_spark_xgboost() {
    attestation_init
    echo -e "${BLUE}occlum run BigDL Spark XGBoost${NC}"
    occlum run /usr/lib/jvm/java-8-openjdk-amd64/bin/java \
                -XX:-UseCompressedOops \
                -XX:ActiveProcessorCount=4 \
                -Divy.home="/tmp/.ivy" \
                -Dos.name="Linux" \
                -cp "$SPARK_HOME/conf/:$SPARK_HOME/jars/*:/bin/jars/*" \
                -Xmx10g -Xms10g org.apache.spark.deploy.SparkSubmit \
                --master local[4] \
                --conf spark.task.cpus=2 \
                --class com.intel.analytics.bigdl.dllib.example.nnframes.xgboost.xgbClassifierTrainingExampleOnCriteoClickLogsDataset \
                --num-executors 2 \
                --executor-cores 2 \
                --executor-memory 9G \
                --driver-memory 10G \
                /bin/jars/bigdl-dllib-spark_${SPARK_VERSION}-${BIGDL_VERSION}.jar \
                -i /host/data -s /host/data/model -t 2 -r 100 -d 2 -w 1
}

run_spark_gbt() {
    attestation_init
    echo -e "${BLUE}occlum run BigDL Spark GBT${NC}"
    occlum run /usr/lib/jvm/java-8-openjdk-amd64/bin/java \
                -XX:-UseCompressedOops \
                -XX:ActiveProcessorCount=4 \
                -Divy.home="/tmp/.ivy" \
                -Dos.name="Linux" \
                -cp "$SPARK_HOME/conf/:$SPARK_HOME/jars/*:/bin/jars/*" \
                -Xmx10g -Xms10g org.apache.spark.deploy.SparkSubmit \
                --master local[4] \
                --conf spark.task.cpus=2 \
                --class com.intel.analytics.bigdl.dllib.example.nnframes.gbt.gbtClassifierTrainingExampleOnCriteoClickLogsDataset \
                --num-executors 2 \
                --executor-cores 2 \
                --executor-memory 9G \
                --driver-memory 10G \
                /bin/jars/bigdl-dllib-spark_${SPARK_VERSION}-${BIGDL_VERSION}.jar \
                -i /host/data -s /host/data/model -I 100 -d 5
}

run_spark_gbt_e2e() {
    attestation_init
    cd /opt/occlum_spark
    echo -e "${BLUE}occlum run BigDL Spark GBT e2e${NC}"
    EHSM_URL=${ATTESTATION_URL}
    EHSM_KMS_IP=${EHSM_URL%:*}
    EHSM_KMS_PORT=${EHSM_URL#*:}
    occlum run /usr/lib/jvm/java-8-openjdk-amd64/bin/java \
                -XX:-UseCompressedOops \
                -XX:ActiveProcessorCount=4 \
                -Divy.home="/tmp/.ivy" \
                -Dos.name="Linux" \
                -cp "$SPARK_HOME/conf/:$SPARK_HOME/jars/*:/bin/jars/*" \
                -Xmx5g -Xms5g org.apache.spark.deploy.SparkSubmit \
                --class com.intel.analytics.bigdl.ppml.examples.GbtClassifierTrainingExampleOnCriteoClickLogsDataset \
                /bin/jars/bigdl-dllib-spark_${SPARK_VERSION}-${BIGDL_VERSION}.jar \
                --primaryKeyPath /host/data/key/ehsm_encrypted_primary_key \
                --kmsType EHSMKeyManagementService \
                --trainingDataPath /host/data/encryptEhsm/ \
                --modelSavePath /host/data/model/ \
                --inputEncryptMode AES/CBC/PKCS5Padding \
                --kmsServerIP $EHSM_KMS_IP \
                --kmsServerPort $EHSM_KMS_PORT \
                --ehsmAPPID $APP_ID \
                --ehsmAPIKEY $API_KEY \
                --maxDepth 5 \
                --maxIter 100
}

run_spark_sql_e2e() {
    attestation_init
    cd /opt/occlum_spark
    echo -e "${BLUE}occlum run BigDL Spark SQL e2e${NC}"
    EHSM_URL=${ATTESTATION_URL}
    EHSM_KMS_IP=${EHSM_URL%:*}
    EHSM_KMS_PORT=${EHSM_URL#*:}
    occlum run /usr/lib/jvm/java-8-openjdk-amd64/bin/java \
                -XX:-UseCompressedOops \
                -XX:ActiveProcessorCount=4 \
                -Divy.home="/tmp/.ivy" \
                -Dos.name="Linux" \
                -cp "$SPARK_HOME/conf/:$SPARK_HOME/jars/*:/bin/jars/*" \
                -Xmx5g -Xms5g org.apache.spark.deploy.SparkSubmit \
                --class com.intel.analytics.bigdl.ppml.examples.SimpleQuerySparkExample \
                /bin/jars/bigdl-dllib-spark_${SPARK_VERSION}-${BIGDL_VERSION}.jar \
                --primaryKeyPath /host/data/key/ehsm_encrypted_primary_key \
                --kmsType EHSMKeyManagementService \
                --inputPath /host/data/encryptEhsm/ \
                --outputPath /host/data/model/ \
                --inputEncryptModeValue AES/CBC/PKCS5Padding \
                --outputEncryptModeValue AES/CBC/PKCS5Padding \
                --kmsServerIP $EHSM_KMS_IP \
                --kmsServerPort $EHSM_KMS_PORT \
                --ehsmAPPID $APP_ID \
                --ehsmAPIKEY $API_KEY
}

run_multi_spark_sql_e2e() {
    attestation_init
    cd /opt/occlum_spark
    echo -e "${BLUE}occlum run BigDL MultiParty Spark SQL e2e${NC}"
    EHSM_URL=${ATTESTATION_URL}
    EHSM_KMS_IP=${EHSM_URL%:*}
    EHSM_KMS_PORT=${EHSM_URL#*:}
    occlum run /usr/lib/jvm/java-8-openjdk-amd64/bin/java \
                -XX:-UseCompressedOops \
                -XX:ActiveProcessorCount=4 \
                -Divy.home="/tmp/.ivy" \
                -Dos.name="Linux" \
                -cp "$SPARK_HOME/conf/:$SPARK_HOME/jars/*:/bin/jars/*" \
                -Xmx5g -Xms5g org.apache.spark.deploy.SparkSubmit \
                --conf spark.hadoop.io.compression.codecs="com.intel.analytics.bigdl.ppml.crypto.CryptoCodec" \
                --conf spark.bigdl.primaryKey.BobPK.kms.type=EHSMKeyManagementService \
                --conf spark.bigdl.primaryKey.BobPK.kms.ip=$EHSM_KMS_IP \
                --conf spark.bigdl.primaryKey.BobPK.kms.port=$EHSM_KMS_PORT \
                --conf spark.bigdl.primaryKey.BobPK.kms.appId=$APP_ID \
                --conf spark.bigdl.primaryKey.BobPK.kms.apiKey=$API_KEY \
                --conf spark.bigdl.primaryKey.BobPK.material=/host/data/key/ehsm_encrypted_primary_key \
                --conf spark.bigdl.primaryKey.AmyPK.kms.type=SimpleKeyManagementService \
                --conf spark.bigdl.primaryKey.AmyPK.kms.appId=123456654321 \
                --conf spark.bigdl.primaryKey.AmyPK.kms.apiKey=123456654321 \
                --conf spark.bigdl.primaryKey.AmyPK.material=/host/data/key/simple_encrypted_primary_key \
                --class com.intel.analytics.bigdl.ppml.examples.MultiPartySparkQueryExample \
                /bin/jars/bigdl-dllib-spark_${SPARK_VERSION}-${BIGDL_VERSION}.jar \
                /host/data/encryptSimple /host/data/encryptEhsm /host/data/ /host/data/
}

verify() { #verify ehsm quote
  cd /opt
  bash verify-attestation-service.sh
}

register() { #register and get policy_Id
  cd /opt
  bash RegisterMrEnclave.sh
}



id=$([ -f "$pid" ] && echo $(wc -l < "$pid") || echo "0")

arg=$1
case "$arg" in
    init)
        init_instance
        build_spark
        ;;
    initDriver)
        attestation_init
        ;;
    initExecutor)
        # to do
        # now executor have to register again
        attestation_init
        ;;
    pypi)
        run_pyspark_pi
        cd ../
        ;;
    pysql)
        run_pyspark_sql_example
        cd ../
        ;;
    pysklearn)
        run_pyspark_sklearn_example
        cd ../
        ;;
    pytpch)
        run_pyspark_tpch_example
        cd ../
        ;;
    pi)
        run_spark_pi
        cd ../
        ;;
    lenet)
        run_spark_lenet_mnist
        cd ../
        ;;
    ut)
        run_spark_unittest
        cd ../
        ;;
    ut_Only)
        run_spark_unittest_only
        cd ../
        ;;
    resnet)
        run_spark_resnet_cifar
        cd ../
        ;;
    tpch)
        run_spark_tpch
        cd ../
        ;;
    xgboost)
        run_spark_xgboost
        cd ../
        ;;
    gbt)
        run_spark_gbt
        cd ../
        ;;
    gbt_e2e)
        run_spark_gbt_e2e
        cd ../
        ;;
    sql_e2e)
        run_spark_sql_e2e
        cd ../
        ;;
    multi_sql_e2e)
        run_multi_spark_sql_e2e
        cd ../
        ;;
    verify)
        verify
        cd ../
        ;;
    register)
        register
        cd ../
        ;;
esac
```

关注`init_instance`方法中的`occlum init --init-ra aecs`,会使用AECS作为init_ra的服务。另外还能去掉一些官方示例中的演示代码
## 3.2 customer构建
主要完成
- Occlum build后的Spark应用
- 拷贝Spark相关文件
- 拷贝远程认证支持相关文件

### 3.2.1 build_customer_image.sh
```

export image=demo/bigdl-ppml-trusted-big-data-ml-scala-occlum-production-build
export TAG=2.4.0-SNAPSHOT-ml-test-032803
export image_customer=${image}-customer
docker build \
  --no-cache \
  --build-arg HTTP_PROXY=${HTTP_PROXY} \
  --build-arg HTTPS_PROXY=${HTTPS_PROXY} \
  --build-arg FINAL_NAME=${image}:${TAG} \
  --build-arg SPARK_JAR_REPO_URL=${SPARK_JAR_REPO_URL} \
  -t ${image_customer}:${TAG} -f ./customer/Dockerfile ./customer

```


### 3.2.2 customer/Dockerfile
```
ARG FINAL_NAME
FROM ${FINAL_NAME} as final
FROM krallin/ubuntu-tini AS tini
COPY --from=final /opt/occlum_spark /opt/occlum_spark
# remove image dir to reduce image size
RUN rm -rf /opt/occlum_spark/image
FROM ubuntu:20.04

MAINTAINER The BigDL Authors https://github.com/intel-analytics/BigDL
ARG BIGDL_VERSION=2.4.0-SNAPSHOT
ARG SPARK_VERSION=3.1.3
ARG HADOOP_VERSION=3.2.0
ARG SPARK_SCALA_VERSION=2.12
ENV SPARK_SCALA_VERSION=${SPARK_SCALA_VERSION}
ENV HADOOP_VERSION=${HADOOP_VERSION}
ENV SPARK_VERSION=${SPARK_VERSION}
ENV BIGDL_VERSION=${BIGDL_VERSION}
ENV SPARK_HOME=/opt/spark
ENV BIGDL_HOME=/opt/bigdl-${BIGDL_VERSION}
ENV JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64
ENV SGX_MEM_SIZE=20GB

ARG HTTP_PROXY
ARG HTTPS_PROXY


#copy occlum runable instance
COPY --from=tini /opt/occlum_spark /opt/occlum_spark

# Configure sgx and occlum deb repo
RUN export HTTP_PROXY=${HTTP_PROXY} HTTPS_PROXY=${HTTPS_PROXY} &&  apt-get update && DEBIAN_FRONTEND=noninteractive apt-get install -y --no-install-recommends vim ca-certificates gnupg2 jq make gdb wget libfuse-dev libtool tzdata && \
                echo 'deb [arch=amd64] https://download.01.org/intel-sgx/sgx_repo/ubuntu focal main' | tee /etc/apt/sources.list.d/intel-sgx.list && \
                wget -qO - https://download.01.org/intel-sgx/sgx_repo/ubuntu/intel-sgx-deb.key | apt-key add -

RUN export HTTP_PROXY=${HTTP_PROXY} HTTPS_PROXY=${HTTPS_PROXY} &&  echo 'deb [arch=amd64] https://occlum.io/occlum-package-repos/debian focal main' | tee /etc/apt/sources.list.d/occlum.list && \
                wget -qO - https://occlum.io/occlum-package-repos/debian/public.key | apt-key add -

#Install sgx dependencies and occlum
RUN export HTTP_PROXY=${HTTP_PROXY} HTTPS_PROXY=${HTTPS_PROXY} && apt-get update && apt-cache policy occlum
RUN export HTTP_PROXY=${HTTP_PROXY} HTTPS_PROXY=${HTTPS_PROXY} && apt-get install -y occlum
RUN export HTTP_PROXY=${HTTP_PROXY} HTTPS_PROXY=${HTTPS_PROXY} && apt-get install -y libsgx-uae-service libsgx-dcap-ql
# glibc
RUN export HTTP_PROXY=${HTTP_PROXY} HTTPS_PROXY=${HTTPS_PROXY} && apt install -y occlum-toolchains-glibc
RUN echo "source /etc/profile" >> $HOME/.bashrc

#COPY bash files
COPY --from=final /opt/*.sh /opt/
COPY --from=final /opt/sqlSuites /opt
COPY --from=final /opt/spark /opt/spark
COPY --from=final /var/run/secrets /var/run/secrets
COPY --from=final /opt/intel /opt/intel
COPY --from=tini /usr/local/bin/tini /sbin/tini
#COPY --from=final /etc /etc

# Add jdk and attestation ppml jars
COPY --from=final /usr/lib/jvm/java-8-openjdk-amd64 /usr/lib/jvm/java-8-openjdk-amd64
COPY --from=final ${BIGDL_HOME}/jars/bigdl* ${BIGDL_HOME}/jars/
COPY --from=final ${BIGDL_HOME}/jars/all-* ${BIGDL_HOME}/jars/

#Add quote attest lib
COPY --from=final /usr/lib/x86_64-linux-gnu/ /usr/lib/x86_64-linux-gnu/
RUN mkdir -p /etc/occlum_attestation/

# useful etc
COPY --from=final  /etc/java-8-openjdk  /etc/java-8-openjdk
COPY --from=final  /etc/hosts /etc/hosts
COPY --from=final  /etc/hostname  /etc/hostname
COPY --from=final  /etc/ssl /etc/ssl
COPY --from=final  /etc/passwd /etc/passwd
COPY --from=final  /etc/group /etc/group
COPY --from=final  /etc/nsswitch.conf /etc/nsswitch.conf


ENV PATH="/opt/occlum/build/bin:/usr/lib/jvm/java-8-openjdk-amd64/bin:/usr/local/occlum/bin:$PATH"
#ENV PATH="/opt/occlum/build/bin:/usr/local/occlum/bin:$PATH"

ENV HTTP_PROXY=
ENV HTTPS_PROXY=

ENTRYPOINT [ "/opt/entrypoint.sh" ]

```

# 4.运行示例
# 4.1 运行前准备
获取构建好的镜像的两个重要度量值

- **MRENCLAVE**(enclave区域中要运行的代码和数据的散列值 ，代表程序)
- **MRSIGNER**（SGX enclave文件签名密钥中公钥部分的散列值，代表开发者）
可以通过以下命令获取
```
# mrenclave
docker run -it --rm datampc/bigdl-ppml-trusted-big-data-ml-scala-occlum-customer:2.4.0-SNAPSHOT /bin/sh -c "cd /opt/occlum_spark/ && occlum print mrenclave"

# mrsigner
docker run -it --rm datampc/bigdl-ppml-trusted-big-data-ml-scala-occlum:2.4.0-SNAPSHOT /bin/sh -c "cd /opt/occlum_spark/ && occlum print mrsigner"
```

在AECS中添加密钥时(image_key和data_key)时，填写对应的mrenclave和mrsigner,限制只有我们开发的应用的可以获取密钥。

## 4.2 submit.sh
```
function param_parse(){

  params=$@
  array=(${params})
  len=${#array[*]}
  i=0
  res=0
  while [ $i -lt $len ]
  do
    if [[ "${array[i]}" == "-"* ]];then
       key=${array[i]:1}
       let i++
       value=${array[i]}
       if [[ "${value}" == "-"*  ]];then
          res=1
       fi
       if [ -z "${value}" ]; then
         res=1
       fi
       eval ${key}='${value}'
    else
      let i++
    fi

  done
  return $res

}
params=$@
param_parse ${params}

user=`whoami`
echo "user is ${user}"

echo "company_s3_info: ${company_s3_info}"

# 启动spark
random_pod_name="tee-spark-driver-$(date +%s)"


export KUBECONFIG=$COMPUT_CLSUTER_CONFIG

${SPARK_HOME}/bin/spark-submit    --class org.example.FtMatchApp \
    --master ${COMPUT_CLSUTER_URL} \
    --deploy-mode cluster \
    --conf spark.kubernetes.namespace=${COMPUT_CLSUTER_NAMESPACE} \
    --conf spark.kubernetes.driver.pod.name=$random_pod_name \
    --conf spark.kubernetes.executor.podNamePrefix=$random_pod_name-executor-pod \
    --conf spark.kubernetes.authenticate.driver.serviceAccountName=${COMPUT_CLSUTER_ACCOUNT} \
    --conf spark.driver.memory=32g \
    --conf spark.executor.memory=${spark_executor_memory} \
    --conf spark.executor.memoryOverhead=2g \
    --conf spark.executor.cores=${spark_executor_cores} \
    --conf spark.driver.cores=8 \
    --conf spark.executor.instances=${spark_num_executors} \
    --conf spark.shuffle.io.maxRetries=10 \
    --conf spark.shuffle.io.retryWait=10s \
    --conf spark.worker.cleanup.interval=60 \
    --conf spark.worker.cleanup.appDataTtl=60 \
    --conf spark.kubernetes.container.image=demo/bigdl-ppml-trusted-big-data-ml-scala-occlum-production-build-customer:2.4.0-SNAPSHOT-ml-test-032803 \
    --conf spark.kubernetes.container.image.pullPolicy=IfNotPresent \
    --conf spark.kubernetes.driver.podTemplateFile=./podTemplate/spark-pod-driver-template.yaml \
    --conf spark.kubernetes.executor.podTemplateFile=./podTemplate/spark-pod-executor-template.yaml \
    --conf spark.executor.memoryOverhead=2g \
    --conf spark.memory.fraction=0.8 \
    --conf spark.memory.storageFraction=0.5 \
    --conf spark.kubernetes.file.upload.path=s3a://hezc/upload \
    --conf spark.hadoop.fs.s3a.endpoint=http://xxx \
    --conf spark.hadoop.fs.s3a.access.key=xxx \
    --conf spark.hadoop.fs.s3a.secret.key=xxx \
    --conf spark.hadoop.fs.s3a.aws.credentials.provider=org.apache.hadoop.fs.s3a.SimpleAWSCredentialsProvider \
    --conf spark.hadoop.fs.s3a.impl=org.apache.hadoop.fs.s3a.S3AFileSystem \
    --conf spark.hadoop.fs.s3a.path.style.access=true \
    --conf spark.hadoop.fs.s3a.connection.ssl.enabled=false \
    --conf spark.executor.extraJavaOptions=-'XX:+UseG1GC -XX:+UseStringDeduplication -XX:InitiatingHeapOccupancyPercent=45 -XX:MaxGCPauseMillis=200' \
    --conf spark.driver.extraJavaOptions='-XX:+UseParallelGC' \
    --conf spark.kubernetes.driverEnv.SGX_DRIVER_JVM_MEM_SIZE="1g" \
    --conf spark.executorEnv.SGX_EXECUTOR_JVM_MEM_SIZE="13g" \
    ./code/model-test.py
    
```

### 4.3 Spark yaml
```
apiVersion: v1
kind: Pod
metadata:
  name: spark-deployment
  namespace: spark
spec:
  containers:
  - name: spark-example
    imagePullPolicy: Never
    volumeMounts:
    - name: device-plugin
      mountPath: /var/lib/kubelet/device-plugins
    - name: sgx-enclave
      mountPath: /dev/sgx/enclave
    - name: sgx-provision
      mountPath: /dev/sgx/provision
    - name: data-exchange
      mountPath: /opt/occlum_spark/data
    - name: app-log
      mountPath: /app/log
#    securityContext:
#      privileged: true
    resources:
      requests:
        sgx.intel.com/epc: 19327352832
        sgx.intel.com/enclave: 1
        sgx.intel.com/provision: 1
      limits:
        sgx.intel.com/epc: 19327352832
        sgx.intel.com/enclave: 1
        sgx.intel.com/provision: 1
    env:
    - name: NETTY_THREAD
      value: "32"
    - name: SGX_MEM_SIZE
      value: "15GB"
    - name: SGX_THREAD
      value: "1024"
#    - name: SGX_HEAP
#      value: "512MB"
    - name: SGX_KERNEL_HEAP
      value: "1GB"
    - name: ATTESTATION
      value: true
    - name: PCCS_URL
      value: "https://172.20.1.13:8081"
    - name: AECS_KMS_URL
      value: "172.20.1.13:19527"
    - name: OCCLUM_INIT_RA_KMS_SERVER
      value: "172.20.1.13:19527"
      #    - name: UA_ENV_PCCS_URL
      #      value: "https://sgx-dcap-server.cn-shanghai.aliyuncs.com/sgx/certification/v3/"
      #    - name: AECS_KMS_URL
      #      value: "172.16.87.16:19527"
      #    - name: OCCLUM_INIT_RA_KMS_SERVER
      #      value: "172.16.87.16:19527"
#    - name: PCCS_URL
#      value: "https://PCCS_IP:PCCS_PORT"
#    - name: ATTESTATION_URL
#      value: "ESHM_IP:EHSM_PORT"
#    - name: APP_ID
#      value: your_app_id
#    - name: API_KEY
#      value: your_api_key
#    - name: CHALLENGE
#      value: cHBtbAo=
#    - name: REPORT_DATA
#      value: ppml
  volumes:
    - name: device-plugin
      hostPath:
        path: /var/lib/kubelet/device-plugins
    - name: sgx-enclave
      hostPath:
        path: /dev/sgx_enclave
    - name: sgx-provision
      hostPath:
        path: /dev/sgx_provision
    - name: data-exchange
      hostPath:
        path: /tmp
    - name: app-log
      hostPath:
        path: /app/log
```
挂载驱动文件、配置PCCS地址环境变量、epc资源等。


# 5.总结
以上使用了[Occlum](https://occlum.io/)和[BigDL-PPML](https://github.com/intel/BigDL/tree/main/ppml)实现了在TEE环境中运行Spark任务，并使用AECS进行远程认证，认证后获取密钥进行数据处理。

需要额外指出的是，Occlum等LibOS在带来便捷性的同时，也带来更多的攻击面，特别是场景中运行Spark这样的大型应用，比如Spark可以运行远程代码（如S3上的JAR），这个代码并不属于Spark应用本身，所以攻击者没有改变Enclave的度量值，也就意味着还是可以通过AECS的认证和获取密钥，这个代码可以打印出所有密钥或打印出敏感数据。此场景中，我们对要运行的代码进行签名，运行前会验签来保障运行的代码也是我们认可的。

所以TEE环境!=安全，还是需要做很多额外的安全加固来保障安全。
