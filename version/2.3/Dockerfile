FROM debian:stretch-slim
# ------------------------------------------------------------
# Do basic install
# ------------------------------------------------------------
RUN apt-get update \
    && mkdir -p /usr/share/man/man1 \
    && apt-get install -y gcc ssh wget vim curl net-tools bison flex openjdk-8-jdk autoconf make libtool m4 automake bzip2 libxml2 libxml2-dev gfortran g++ iputils-ping pkg-config colordiff nano git sudo lsof jq libtdl* \
    && curl --location --silent --show-error --output /tini \
       https://github.com/krallin/tini/releases/download/v0.19.0/tini \
    && chmod +x /tini \
    && adduser --uid 1000 --home /home/ymluser --shell /bin/bash \
       --disabled-password --gecos '' ymluser \
    && passwd -d ymluser \
    && apt-get install -y openssh-server \
    && mkdir -p /run/sshd /home/ymluser/.ssh /home/ymluser/.ssh-source \
    && echo "StrictHostKeyChecking no" > /home/ymluser/.ssh/config \
    && chown -R ymluser /home/ymluser \
    && sed -i s/#PermitRootLogin.*/PermitRootLogin\ no/ /etc/ssh/sshd_config \
    && sed -i s/#PubkeyAuthentication.*/PubkeyAuthentication\ no/ /etc/ssh/sshd_config \
    && sed -i s/.*UsePAM.*/UsePAM\ no/ /etc/ssh/sshd_config \
    && sed -i s/#PasswordAuthentication.*/PasswordAuthentication\ yes/ /etc/ssh/sshd_config \
    && sed -i s/#PermitEmptyPasswords.*/PermitEmptyPasswords\ yes/ /etc/ssh/sshd_config \
    && sed -i s/#ChallengeResponse.*/ChallengeResponseAuthentication\ no/ /etc/ssh/sshd_config \
    && sed -i s/#PermitUserEnvironment.*/PermitUserEnvironment\ yes/ /etc/ssh/sshd_config \
	&& adduser ymluser sudo

ENV PREFIX=/usr/local \
	OPENMPI_VERSION=3.1.6 \
	EXPAT_VERSION=2.4.1 \
	EXPAT_TAG=R_2_4_1 \
    LD_LIBRARY_PATH=/usr/local/lib \
    DEBCONF_NOWARNINGS=yes

# ------------------------------------------------------------
# Install OpenMPI 3.1
# https://download.open-mpi.org/release/open-mpi/v3.1/openmpi-3.1.6.tar.gz
# ------------------------------------------------------------

# OpenMPI v3.1
RUN repo="https://download.open-mpi.org/release/open-mpi/v3.1" \
    && curl --location --silent --show-error --output openmpi.tar.gz \
      "${repo}/openmpi-${OPENMPI_VERSION}.tar.gz" \
    && tar xzf openmpi.tar.gz -C /tmp/ \
    && cd /tmp/openmpi-${OPENMPI_VERSION} \
	&& env CFLAGS="-O2 -std=gnu99 -fopenmp" \
    && ./configure --prefix=${PREFIX} \
    && make \
    && make install \
    && ldconfig \
    && cd / \
    && rm -rf /tmp/openmpi-${OPENMPI_VERSION} /home/ymluser/openmpi.tar.gz

# ------------------------------------------------------------
# libxepat : needed by libutil-0.1.5
# ------------------------------------------------------------

RUN repo="https://github.com/libexpat/libexpat/tarball/${EXPAT_TAG}" \
	&& curl --location --silent --show-error --output libexpat.tar.gz \
	  "${repo}/expat-${EXPAT_VERSION}.tar.gz" \
	&& tar xzf libexpat.tar.gz -C /tmp/ \
	&& mv /tmp/libexpat* /tmp/libexpat-${EXPAT_VERSION} \
	&& cd /tmp/libexpat-${EXPAT_VERSION}/expat \
	&& libtoolize --force \
	&& aclocal \
	&& autoheader \
	&& automake --force-missing --add-missing \
	&& autoconf \
	&& ./configure --prefix=${PREFIX} \
	&& make \
	&& make install \
	&& ldconfig \
	&& cd /home/ymluser \
	&& rm -rf /tmp/libexpat-${EXPAT_VERSION} /home/ymluser/libexpat.tar.gz


# ------------------------------------------------------------
# Load full YML2.3 bundle from a public repo
# ------------------------------------------------------------

WORKDIR /home/ymluser
RUN wget --no-check-certificate --content-disposition http://www.hpcs.cs.tsukuba.ac.jp/~tsuji/fp2c-140619.tar.gz && \
	tar xzf fp2c-140619.tar.gz && \
    rm fp2c-140619.tar.gz && \
    mv fp2c-140619 YMLEnvironment

# ------------------------------------------------------------
# Load omni-compiler 1.1.3 from a public repo
# ------------------------------------------------------------

WORKDIR /home/ymluser/YMLEnvironment
RUN wget --no-check-certificate --content-disposition https://omni-compiler.org/download/stable/omnicompiler-1.1.3.tar.bz2
RUN bunzip2 omnicompiler-1.1.3.tar.bz2 \
	&& tar xvf omnicompiler-1.1.3.tar \
	&& rm /home/ymluser/YMLEnvironment/omnicompiler-1.1.3.tar
	
# ------------------------------------------------------------
# Install omnicompiler-1.1.3 : requested javac
# ------------------------------------------------------------

WORKDIR /home/ymluser/YMLEnvironment/omnicompiler-1.1.3
RUN cd /home/ymluser/YMLEnvironment/omnicompiler-1.1.3 \
	&& libtoolize --force \
	&& aclocal \
	&& autoheader \
	&& autoconf \
	&& ./configure --prefix=${PREFIX} --with-libxml2=/usr \
#	&& env CFLAGS="-I${PREFIX}/include -Wl,--copy-dt-needed-entries,--allow-multiple-definition" ./configure --prefix=${PREFIX} --with-libxml2-include=/usr/include/libxml2 \
	&& make && make install && make clean && ldconfig
	
# ------------------------------------------------------------
# Patch omnirpc (from YML2.3 bundle) - add fopenmp binding
# ------------------------------------------------------------
WORKDIR /home/ymluser/YMLEnvironment/omnirpc-mpi-2.2.2
RUN echo "patch omnirpc-mpi-2.2.2\n" \
	&& cd /home/ymluser/YMLEnvironment/omnirpc-mpi-2.2.2/src/omrpc-register-yml       \
	&& sed -i s/\-lm/\-lm\ \-fopenmp/g ./omrpc-register-yml.c

# ------------------------------------------------------------
# Install omnirpc-mpi-2.2.2 : mpi 3.1
# ------------------------------------------------------------
WORKDIR /home/ymluser/YMLEnvironment/omnirpc-mpi-2.2.2
RUN cd /home/ymluser/YMLEnvironment/omnirpc-mpi-2.2.2 \
    && export CPPFLAGS="-DOMNI_CPU_X86_64 -DOMNI_OS_LINUX -DNDEBUG -D_REENTRANT" \
	&& env CFLAGS="-Wl,--no-relax" ./configure --prefix=${PREFIX} --enable-gcc --with-cc=mpicc --with-opt=-fPIC \
	&& make && make install && make clean && ldconfig
# ------------------------------------------------------------
# Install zlib : needed by libutil-0.1.5
# ------------------------------------------------------------
WORKDIR /home/ymluser
RUN cd /home/ymluser \
	&& wget --no-check-certificate --content-disposition "http://zlib.net/zlib-1.2.11.tar.gz" \
	&& tar xzf zlib-1.2.11.tar.gz \
	&& cd /home/ymluser/zlib-1.2.11 \
	&& ./configure --prefix=${PREFIX} \
    && make \
    && make install \
	&& rm -rf /home/ymluser/zlib-1.2.11.tar.gz /home/ymluser/zlib-1.2.11 \
    && ldconfig
	
# ------------------------------------------------------------
# Install libutil-0.1.5 : needed by yml 2.3, binding with -lutil
# ------------------------------------------------------------
WORKDIR /home/ymluser/YMLEnvironment/libutil-0.1.5
RUN cd /home/ymluser/YMLEnvironment/libutil-0.1.5 \
	&& env CPPFLAGS=-std=c++03 ./configure --prefix=${PREFIX} --with-expat-incdir=${PREFIX}/include --with-expat-libdir=${PREFIX}/lib \
	&& make && make install && make clean && ldconfig
	
# ------------------------------------------------------------
# Patch yml2.3 --> add fopenmp + clean init/finalize
# ------------------------------------------------------------

WORKDIR /home/ymluser/YMLEnvironment/yml-2.3.0
RUN echo "patch yml2.3\n" \
	&& cd /home/ymluser/YMLEnvironment/yml-2.3.0/src/Plugins/Catalogs/DefaultExecutionCatalog \
	&& sed -i s/\"\-lm\"/\"\-lm\"\ \<\<\ \"\-fopenmp\"/g ./XMPBinaryGenerator.cc \
	&& echo "extern void xmpc_init_all(int, char **);\n" \
	"extern void xmpc_finalize_all(int);\n" \
	"void __XMP_init(int argc, char **argv)\n" \
	"{ \n" \
	"  xmpc_init_all(argc, argv);\n" \
	"}\n" \
	"void __XMP_finalize()\n"\
	"{ \n"\
	"  xmpc_finalize_all(0);\n" \
	"}" >> xmp_std_lib.c

# ------------------------------------------------------------
# Install YML2.3
# ------------------------------------------------------------
WORKDIR /home/ymluser/YMLEnvironment/yml-2.3.0
RUN cd /home/ymluser/YMLEnvironment/yml-2.3.0 \
    && libtoolize --force \
	&& aclocal \
	&& autoheader \
	&& automake --force-missing --add-missing \
	&& autoconf \
	&& export CXXFLAGS="-Wl,--no-as-needed,--copy-dt-needed-entries,--allow-multiple-definition" \
	&& export CFLAGS="-I${PREFIX}/include -DOMPI_SKIP_MPICXX -Wl,--no-undefined,--no-allow-shlib-undefined,--copy-dt-needed-entries,--allow-multiple-definition" \
	&& export CPPFLAGS="-I${PREFIX}/include -std=c++03 -DOMPI_SKIP_MPICXX" \
	&& ./configure --prefix=${PREFIX} --with-expat-incdir=${PREFIX}/include --with-expat-libdir=${PREFIX}/lib --with-mpi-lib="-L/usr/local/lib -lmpi -lutil" \
	&& make && make install && make clean && ldconfig

# ------------------------------------------------------------
# Add some parameters for YML2.3
# ------------------------------------------------------------	
RUN mkdir -p /usr/local/var/yml
RUN	mkdir -p /home/ymluser/.omrpc_registry && chown 1000:1000 /home/ymluser/.omrpc_registry
RUN ln -s /home/ymluser/.omrpc_registry /home/.omrpc_registry
RUN mkdir -p /usr/local/var/yml/DefaultExecutionCatalog/indexes && chown -R 1000:1000 /usr/local/var/yml

RUN echo "ymluser ALL=(ALL) NOPASSWD:ALL\n" >> /etc/sudoers

RUN rm -fr /home/ymluser/.openmpi && mkdir -p /home/ymluser/.openmpi
RUN cd /home/ymluser/.openmpi \
	&& echo "btl=tcp,self\n" \
	"btl_tcp_if_include=eth0\n" \
	"plm_rsh_no_tree_spawn=1\n" >> default-mca-params.conf

RUN chown -R 1000:1000 /home/ymluser/.openmpi
RUN chown -R 1000:1000 /home/ymluser/YMLEnvironment

# ------------------------------------------------------------
# Start mpi python install / user ymluser
# ------------------------------------------------------------
RUN apt-get install -y --no-install-recommends python-dev python-numpy python-pip python-virtualenv python-scipy \
    && apt-get clean && apt-get purge && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*

RUN pip install --upgrade pip

# in order to have python related to ymluser account
USER ymluser
RUN  pip install --user -U setuptools \
    && pip install --user mpi4py
USER root


# ------------------------------------------------------------
# Copy MPI4PY example scripts
# ------------------------------------------------------------

ENV TRIGGER 1

ADD ./mpi4py_benchmarks /home/ymluser/mpi4py_benchmarks
RUN chown -R ymluser:ymluser /home/ymluser/mpi4py_benchmarks

# ------------------------------------------------------------
# The .ssh-source dir contains RSA keys - put in place with docker-compose
# ------------------------------------------------------------


RUN touch /home/ymluser/.ssh-source/authorized_keys
RUN touch /home/ymluser/.ssh-source/id_rsa


# ------------------------------------------------------------
# Do SSHd parameter to enable ymluser to run it
# ------------------------------------------------------------
RUN sed -i s/#UsePrivilegeSeparation.*/UsePrivilegeSeparation\ no/ /etc/ssh/sshd_config
RUN mkdir -p /home/ymluser/ssh
RUN ssh-keygen -q -N "" -t dsa -f /home/ymluser/ssh/ssh_host_dsa_key \
	&& ssh-keygen -q -N "" -t rsa -b 4096 -f /home/ymluser/ssh/ssh_host_rsa_key \
	&& ssh-keygen -q -N "" -t ecdsa -f /home/ymluser/ssh/ssh_host_ecdsa_key \
	&& ssh-keygen -q -N "" -t ed25519 -f /home/ymluser/ssh/ssh_host_ed25519_key

RUN cp /etc/ssh/sshd_config /home/ymluser/ssh/

RUN sed -i s/#HostKey\ \\/etc\\/ssh/HostKey\ \\/home\\/ymluser\\/ssh/ /home/ymluser/ssh/sshd_config
RUN sed -i s/#PidFile\ \\/var\\/run/PidFile\ \\/home\\/ymluser\\/ssh/ /home/ymluser/ssh/sshd_config
RUN sed -i s/#LogLevel.*/LogLevel\ DEBUG3/ /home/ymluser/ssh/sshd_config
RUN sed -i s/PubkeyAuthentication\ no/PubkeyAuthentication\ yes/ /home/ymluser/ssh/sshd_config

RUN chown -R ymluser:ymluser /home/ymluser/ssh

WORKDIR /home/ymluser
ENTRYPOINT ["/tini", "--"]
USER ymluser

CMD cp -R /home/ymluser/.ssh-source/* /home/ymluser/.ssh \
	&& chmod 700 /home/ymluser/.ssh/authorized_keys \
	&& chmod 700 /home/ymluser/.ssh/id_rsa \
	&& exec /usr/sbin/sshd -D -f /home/ymluser/ssh/sshd_config -E /tmp/sshd.log
# do not detach (-D), log to stderr (-e)
# CMD ["/usr/sbin/sshd", "-D", "-e"]