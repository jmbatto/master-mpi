FROM debian:stretch-slim
# an image with MPI3.1, used without root privileges
# ------------------------------------------------------------
# Do basic install
# ------------------------------------------------------------
RUN apt-get update \
    && mkdir -p /usr/share/man/man1 \
    && apt-get install -y gcc ssh wget vim curl net-tools bison flex autoconf make libtool m4 automake bzip2 libxml2 libxml2-dev gfortran g++ iputils-ping pkg-config colordiff nano git sudo lsof gawk emacs jq libtdl* \
    && curl --location --silent --show-error --output /tini \
       https://github.com/krallin/tini/releases/download/v0.19.0/tini \
    && chmod +x /tini \
    && adduser --uid 1000 --home /home/mpiuser --shell /bin/bash \
       --disabled-password --gecos '' mpiuser \
    && passwd -d mpiuser \
    && apt-get install -y openssh-server \
    && mkdir -p /run/sshd /home/mpiuser/.ssh /home/mpiuser/.ssh-source \
    && echo "StrictHostKeyChecking no" > /home/mpiuser/.ssh/config \
    && chown -R mpiuser /home/mpiuser \
    && sed -i s/#PermitRootLogin.*/PermitRootLogin\ no/ /etc/ssh/sshd_config \
    && sed -i s/#PubkeyAuthentication.*/PubkeyAuthentication\ no/ /etc/ssh/sshd_config \
    && sed -i s/.*UsePAM.*/UsePAM\ no/ /etc/ssh/sshd_config \
    && sed -i s/#PasswordAuthentication.*/PasswordAuthentication\ yes/ /etc/ssh/sshd_config \
    && sed -i s/#PermitEmptyPasswords.*/PermitEmptyPasswords\ yes/ /etc/ssh/sshd_config \
    && sed -i s/#ChallengeResponse.*/ChallengeResponseAuthentication\ no/ /etc/ssh/sshd_config \
    && sed -i s/#PermitUserEnvironment.*/PermitUserEnvironment\ yes/ /etc/ssh/sshd_config \
	&& adduser mpiuser sudo

ENV PREFIX=/usr/local \
	OPENMPI_VERSION=3.1.6 \
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
    && rm -rf /tmp/openmpi-${OPENMPI_VERSION} /home/mpiuser/openmpi.tar.gz

# ------------------------------------------------------------
# Add some parameters for MPI, mpishare - a folder shared through the nodes
# ------------------------------------------------------------	
RUN mkdir -p /usr/local/var/mpishare

RUN chown -R 1000:1000 /usr/local/var/mpishare

RUN echo "mpiuser ALL=(ALL) NOPASSWD:ALL\n" >> /etc/sudoers

RUN rm -fr /home/mpiuser/.openmpi && mkdir -p /home/mpiuser/.openmpi
RUN cd /home/mpiuser/.openmpi \
	&& echo "btl = tcp,self \n" \
	"btl_tcp_if_include = eth0 \n" \
	"plm_rsh_no_tree_spawn = 1 \n" >> default-mca-params.conf

RUN chown -R 1000:1000 /home/mpiuser/.openmpi

RUN echo "rmaps_base_oversubscribe = 1\n" >> /usr/local/etc/openmpi-mca-params.conf
RUN echo "rmaps_base_inherit = 1\n" >> /usr/local/etc/openmpi-mca-params.conf


# ------------------------------------------------------------
# Start mpi python install / user mpiuser
# ------------------------------------------------------------
RUN apt-get install -y --no-install-recommends python-dev python-numpy python-pip python-virtualenv python-scipy \
    && apt-get clean && apt-get purge && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*

RUN pip install --upgrade pip

# in order to have python related to mpiuser account
USER mpiuser
RUN  pip install --user -U setuptools \
    && pip install --user mpi4py
USER root


# ------------------------------------------------------------
# Copy MPI4PY example scripts
# ------------------------------------------------------------



ADD ./mpi4py_benchmarks /home/mpiuser/mpi4py_benchmarks
RUN chown -R mpiuser:mpiuser /home/mpiuser/mpi4py_benchmarks

# ------------------------------------------------------------
# Install Golang 1.17.3
# ------------------------------------------------------------	

ENV GOLANG_VERSION=1.17.3
RUN cd /home/mpiuser/ && curl --location --silent --show-error --output golang.tar.gz \
    "https://storage.googleapis.com/golang/go${GOLANG_VERSION}.linux-amd64.tar.gz"
	
RUN cd /home/mpiuser/ && tar -xvf golang.tar.gz && mv go /usr/local && rm golang.tar.gz
RUN mkdir -p /home/mpiuser/go && chown -R 1000:1000 /home/mpiuser/go
RUN echo "export GOPATH=/home/mpiuser/go \n" >> /home/mpiuser/.profile
RUN echo "export PATH=$PATH:/usr/local/go/bin:$GOPATH/bin \n" >> /home/mpiuser/.profile 

RUN echo "export GOPATH=/home/mpiuser/go \n" >> /home/mpiuser/.bashrc
RUN echo "export PATH=$PATH:/usr/local/go/bin:$GOPATH/bin \n" >> /home/mpiuser/.bashrc

# ------------------------------------------------------------
# The .ssh-source dir contains RSA keys - put in place with docker-compose
# ------------------------------------------------------------


RUN touch /home/mpiuser/.ssh-source/authorized_keys
RUN touch /home/mpiuser/.ssh-source/id_rsa


# ------------------------------------------------------------
# Do SSHd parameter to enable mpiuser to run it
# ------------------------------------------------------------
RUN sed -i s/#UsePrivilegeSeparation.*/UsePrivilegeSeparation\ no/ /etc/ssh/sshd_config
RUN mkdir -p /home/mpiuser/ssh
RUN ssh-keygen -q -N "" -t dsa -f /home/mpiuser/ssh/ssh_host_dsa_key \
	&& ssh-keygen -q -N "" -t rsa -b 4096 -f /home/mpiuser/ssh/ssh_host_rsa_key \
	&& ssh-keygen -q -N "" -t ecdsa -f /home/mpiuser/ssh/ssh_host_ecdsa_key \
	&& ssh-keygen -q -N "" -t ed25519 -f /home/mpiuser/ssh/ssh_host_ed25519_key

RUN cp /etc/ssh/sshd_config /home/mpiuser/ssh/

RUN sed -i s/#HostKey\ \\/etc\\/ssh/HostKey\ \\/home\\/mpiuser\\/ssh/ /home/mpiuser/ssh/sshd_config
RUN sed -i s/#PidFile\ \\/var\\/run/PidFile\ \\/home\\/mpiuser\\/ssh/ /home/mpiuser/ssh/sshd_config
RUN sed -i s/#LogLevel.*/LogLevel\ DEBUG3/ /home/mpiuser/ssh/sshd_config
RUN sed -i s/PubkeyAuthentication\ no/PubkeyAuthentication\ yes/ /home/mpiuser/ssh/sshd_config

RUN chown -R mpiuser:mpiuser /home/mpiuser/ssh

WORKDIR /home/mpiuser
ENTRYPOINT ["/tini", "--"]
USER mpiuser

CMD cp -R /home/mpiuser/.ssh-source/* /home/mpiuser/.ssh \
	&& chmod 700 /home/mpiuser/.ssh/authorized_keys \
	&& chmod 700 /home/mpiuser/.ssh/id_rsa \
	&& exec /usr/sbin/sshd -D -f /home/mpiuser/ssh/sshd_config -E /tmp/sshd.log
# do not detach (-D), log to stderr (-e)
# CMD ["/usr/sbin/sshd", "-D", "-e"]