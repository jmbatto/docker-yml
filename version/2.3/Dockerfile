FROM debian:buster-slim

RUN apt-get update \
    && mkdir -p /usr/share/man/man1 \
    && apt-get install -y gcc ssh wget vim curl net-tools bison flex openjdk-11-jdk autoconf make libtool m4 automake bzip2 libxml2 libxml2-dev gfortran g++ iputils-ping pkg-config libtdl* \
    && curl --location --silent --show-error --output /tini \
       https://github.com/krallin/tini/releases/download/v0.19.0/tini \
    && chmod +x /tini \
    && adduser --uid 1000 --home /home/ymluser --shell /bin/bash \
       --disabled-password --gecos '' ymluser \
    && passwd -d ymluser \
    && apt-get install -y openssh-server \
    && mkdir -p /run/sshd /home/ymluser/.ssh \
    && echo "StrictHostKeyChecking no" > /home/ymluser/.ssh/config \
    && chown -R ymluser /home/ymluser \
    && sed -i s/#PermitRootLogin.*/PermitRootLogin\ no/ /etc/ssh/sshd_config \
    && sed -i s/#PubkeyAuthentication.*/PubkeyAuthentication\ no/ /etc/ssh/sshd_config \
    && sed -i s/.*UsePAM.*/UsePAM\ no/ /etc/ssh/sshd_config \
    && sed -i s/#PasswordAuthentication.*/PasswordAuthentication\ yes/ /etc/ssh/sshd_config \
    && sed -i s/#PermitEmptyPasswords.*/PermitEmptyPasswords\ yes/ /etc/ssh/sshd_config \
    && sed -i s/#ChallengeResponse.*/ChallengeResponseAuthentication\ no/ /etc/ssh/sshd_config \
    && sed -i s/#PermitUserEnvironment.*/PermitUserEnvironment\ yes/ /etc/ssh/sshd_config

ENV PREFIX=/usr/local \
	OPENMPI_VERSION=3.1.6 \
	EXPAT_VERSION=2.4.1 \
	EXPAT_TAG=R_2_4_1 \
    LD_LIBRARY_PATH=${PREFIX}/lib \
    DEBCONF_NOWARNINGS=yes

# Open-UCX : high speed network abstraction
WORKDIR /home/ymluser
RUN  wget --no-check-certificate --content-disposition "https://github.com/openucx/ucx/releases/download/v1.11.2/ucx-1.11.2.tar.gz" \
	&& tar xzf ucx-1.11.2.tar.gz \
	&& rm -rf /home/ymluser/ucx-1.11.2.tar.gz \
	&& cd /home/ymluser/ucx-1.11.2 \
	&& ./configure --prefix=${PREFIX} \
    && make \
    && make install \
    && ldconfig
	

# https://download.open-mpi.org/release/open-mpi/v3.1/openmpi-3.1.6.tar.gz

# OpenMPI v3.1 : use Open-UCX
RUN repo="https://download.open-mpi.org/release/open-mpi/v3.1" \
    && curl --location --silent --show-error --output openmpi.tar.gz \
      "${repo}/openmpi-${OPENMPI_VERSION}.tar.gz" \
    && tar xzf openmpi.tar.gz -C /tmp/ \
    && cd /tmp/openmpi-${OPENMPI_VERSION} \
    && ./configure --prefix=${PREFIX} --with-ucx --with-ucx-libdir=/usr/local/lib \
    && make \
    && make install \
    && ldconfig \
    && cd / \
    && rm -rf /tmp/openmpi-${OPENMPI_VERSION} /home/ymluser/openmpi.tar.gz


# libxepat : needed by libutil-0.1.5
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

# load full bundle
WORKDIR /home/ymluser
RUN wget --no-check-certificate --content-disposition http://www.hpcs.cs.tsukuba.ac.jp/~tsuji/fp2c-140619.tar.gz && \
	tar xzf fp2c-140619.tar.gz && \
    rm fp2c-140619.tar.gz && \
    mv fp2c-140619 YMLEnvironment

# load omni-compiler 1.3.4
WORKDIR /home/ymluser/YMLEnvironment
RUN wget --no-check-certificate --content-disposition https://omni-compiler.org/download/stable/omnicompiler-1.3.4.tar.bz2
RUN bunzip2 omnicompiler-1.3.4.tar.bz2 \
	&& tar xvf omnicompiler-1.3.4.tar \
	&& rm /home/ymluser/YMLEnvironment/omnicompiler-1.3.4.tar

# omnicompiler-1.3.4 : requested javac
WORKDIR /home/ymluser/YMLEnvironment/omnicompiler-1.3.4
RUN cd /home/ymluser/YMLEnvironment/omnicompiler-1.3.4 \
	&& libtoolize --force \
	&& aclocal \
	&& autoheader \
	&& autoconf \
	&& env CFLAGS="-I${PREFIX}/include -Wl,--copy-dt-needed-entries,--allow-multiple-definition" ./configure --prefix=${PREFIX} --with-libxml2-include=/usr/include/libxml2 \
	&& make && make install && make clean && ldconfig

# omnirpc-mpi-2.2.2 : mpi 3.1
WORKDIR /home/ymluser/YMLEnvironment/omnirpc-mpi-2.2.2
RUN cd /home/ymluser/YMLEnvironment/omnirpc-mpi-2.2.2 \
	&& env CFLAGS="-Wl,--no-relax" ./configure --prefix=${PREFIX} --enable-gcc --with-cc=mpicc --with-opt=-fPIC \
	&& make && make install && make clean && ldconfig

# zlib : needed by libutil-0.1.5
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

# libutil-0.1.5 : needed by yml 2.3, binding with -lutil
WORKDIR /home/ymluser/YMLEnvironment/libutil-0.1.5
RUN cd /home/ymluser/YMLEnvironment/libutil-0.1.5 \
	&& env CPPFLAGS=-std=c++03 ./configure --prefix=${PREFIX} --with-expat-incdir=${PREFIX}/include --with-expat-libdir=${PREFIX}/lib \
	&& make && make install && make clean && ldconfig

WORKDIR /home/ymluser/YMLEnvironment/yml-2.3.0
RUN cd /home/ymluser/YMLEnvironment/yml-2.3.0 \
	&& env CPPFLAGS="-I${PREFIX}/include -std=c++03 -DOMPI_SKIP_MPICXX" ./configure --prefix=${PREFIX} --with-expat-incdir=${PREFIX}/include --with-expat-libdir=${PREFIX}/lib --with-mpi-lib="-L/usr/local/lib -lmpi -lutil" --disable-xtremweb-net \
	&& make && make install && make clean && ldconfig

WORKDIR /home/ymluser

ENTRYPOINT ["/tini", "--"]
# do not detach (-D), log to stderr (-e)
CMD ["/usr/sbin/sshd", "-D", "-e"]