#####该文件是用来记录自己编译brpc的步骤/问题及解决办法####

1. 下载brpc 
    git链接:git clone https://github.com/apache/brpc.git
    参数安装文档:https://github.com/apache/brpc/blob/master/docs/cn/getting_started.md#docker

2. 下载并编译依赖项: gflags, protobuf,leveldb,  (源码编译,这样对机器的系统适配性较好)
    gflags: 本次使用v2.2.2(一定要加-DCMAKE_CXX_FLAGS=-fPIC)
        git clone https://github.com/gflags/gflags.git
        或者下载解码源码:
            $ tar xzf gflags-$version-source.tar.gz
            $ cd gflags-$version

        $ mkdir build && cd build
        $ cmake .. -DBUILD_SHARED_LIBS=1 -DBUILD_STATIC_LIBS=1 -DCMAKE_CXX_FLAGS=-fPIC

        - Press 'c' to configure the build system and 'e' to ignore warnings.
        - Set CMAKE_INSTALL_PREFIX and other CMake variables and options.
        - Continue pressing 'c' until the option 'g' is available.
        - Then press 'g' to generate the configuration files for GNU Make.

        $ make
        $ make test    (optional)
        $ make install (optional)

    protobuf:(3.20.3可以用以下编译方式,太新的版本好像没有make方式了)
        git clone https://github.com/protocolbuffers/protobuf.git
        cd protobuf
        git submodule update --init --recursive
        ./autogen.sh
        ./configure --prefix=/usr -DCMAKE_CXX_FLAGS=-fPIC
        make
        make check
        sudo make install
        sudo ldconfig # refresh shared library cache.(编译在本文件夹的话不需要该操作,只需要拿到 include/lib/bin)

    leveldb:v_1.23
        git clone https://github.com/google/leveldb.git
        cd leveldb
        mkdir -p build && cd build
        cmake  .. -DCMAKE_BUILD_TYPE=Release -DBUILD_SHARED_LIBS=1 -DBUILD_STATIC_LIBS=1 -DCMAKE_CXX_FLAGS=-fPIC
        make 

3. 在brpc目录中新建thirdparty目录, 依次把编译的依赖项输出(include/lib/bin)拷贝到thirdparty目录中
    brpc/thirdparty
        .
        |-- gflags_v2.2.2
        |   |-- include
        |   `-- lib
        |-- leveldb
        |   |-- bin
        |   |-- include
        |   `-- lib
        |-- protobuf_v3.20.3
        |   |-- bin
        |   |-- include
        |   `-- lib

4. 进入brpc目录,开始编译
    ## sh config_brpc.sh --headers=/usr/local/include --libs=/usr/local/lib --cc=clang --cxx=clang++
    1. 打开MakeFile文件,替换其中的"-std=c++0x"为"-std=c++11"
    2. sh config_brpc.sh --headers="./thirdparty/leveldb/include ./thirdparty/gflags_v2.2.2/include ./thirdparty/protobuf_v3.20.3/include \
        /usr/include /usr/local/include" --libs="./thirdparty/leveldb/lib ./thirdparty/gflags_v2.2.2/lib ./thirdparty/protobuf_v3.20.3/lib \
        ./thirdparty/protobuf_v3.20.3/bin"
    3. make &>log.txt
    4. 成功后在output目录中有 include/lib/bin

问题:
    1. 如果前面步骤中有顺序差异,尤其是protobuf存在多版本错乱时, 可能出现"idl_options.pb" 使用的protoc版本过旧/新错误,这时需要正确配置 config_brpc.sh的参数项,
        并把brpc/src下的idl_options.pb.*文件删除(这些是protoc新生成的,不要删除了idl_options.proto),再重新make.
    2. 如果编译过程出现一些undefine refence std::string c++11等的错误, 则需要 确认执行上面 4中的第1步, 修改配置文件的-std=c++11
    3. 最后编译example/echo_c++中的client/server时,如果出现很多undefined refernce to 'brpc::RpczSpan::RpczSpan()'等时,
        可以使用nm libbrpc.so| c++filt |grep brpc::StreamSettings::StreamSettings 命令或者 ldd -r libbrpc.so,验证libbrpc.so中是否存在未定义的符号.
        本人编译过程中发现是存在未定义的符号,排查后 是gflags库编译过程中没有加-fPIC,
        需要重新编译gflags,然后替换依赖的gflags后,再重新编译libbrpc.so, 再次验证ldd -r libbrpc.so发现没有未定义的符号了, example也可以正常编译生成可执行文件client/server

        背景资料:gflag编译gflags.a的时候默认不会添加-fPIC选项，这会导致打包-libbrpc.so时出错，一种解决办法是，通过源代码编译gflags，然后执行cmake的时候，
        添加选项CXXFLAGS=-fPIC，或者export CXXFLAGS=-fPIC即可加上-fPIC，你也可以删掉libgflags.a，这样就会通过动态库的方式链接gflags（如果有的话）
    
    
