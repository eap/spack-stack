

## spack-stack install for AWS Rocky 9

**Note.** These instructions were used to create this site config. These steps are **not** necessary when building ``spack-stack`` - simply use the existing site config in this directory or update as necessary.

### Using this Site Config

JCSDA publishes a fully configured and built installation of spack stack derived
from this config as an Amazon Machine Images (AMI). The easiest way to use this
configuration of spack stack is to launch a VM using that AMI. The instructions
below are included for maintainance of the history of this site config and
their possible relevance to debugging issues should they arise.


### Base instance

This AMI was built on instance with the following properties.
- AMI Name: Rocky-9-EC2-Base-9.4-20240523.0.x86_64
- AMI ID: ami-051a0f669bb174783
- Instance m6i.4xlarge  (uses Intel Xeon processor)
- 300GB of gp3 storage as /



### Tmux

Rocky 9 doesn't have screen so here's a guide to tmux
```
# Start a session
tmux new -s my-session-name

# Detach from a session
ctr-b d

# list sessions
tmux ls

# Attach to existing session
tmux attach -t my-session-name
```


### Installing Prerequisites


```
# Update system software and start a screen session.
sudo dnf -y update
sudo dnf -y install tmux
tmux new -s setup
sudo su -


# Compilers
dnf install gcc-toolset-12-gcc-c++ \
        gcc-toolset-12-gcc-gfortran \
        gcc-toolset-12-gdb


#Install other requirements.
dnf install binutils-devel \
        m4 \
        wget \
        git \
        git-lfs \
        bash-completion \
        bzip2 bzip2-devel \
        unzip \
        patch \
        automake \
        xorg-x11-xauth \
        xterm \
        perl-IPC-Cmd \
        pearl-core \
        gettext-devel \
        texlive \
        tcl-devel \
        nano \
        bison

# Python develop.
dnf install python3-devel

# Install clang.
dnf install clang clang-devel

# Enable gcc toolset. This is needed for later builds. Once lmod is installed
# this will be configured as a pass-through module.
scl enable gcc-toolset-12 bash

# Configure git credential caching and git lfs for the rocky user and root.
git config --global credential.helper 'cache --timeout=3600'
sudo git config --global credential.helper 'cache --timeout=3600'
git lfs install
sudo git lfs install

# Configure x11 forwarding.
echo "X11Forwarding yes" >> /etc/ssh/sshd_config
service sshd restart
```

##
## DELETE 1. Install apt build prerequisites
##
##
##
### Additional external dependencies for Intel spack builds.
##apt install -y \
##    liblcms2-dev \
##    liblcms2-2
##
### Install QT5, needed for install of ecflow
##apt install -y \
##    qtcreator \
##    qtbase5-dev \
##    qt5-qmake \
##    libqt5svg5-dev \
##    qt5dxcb-plugin
##
### This is because boost doesn't work with the Intel compiler
##apt install -y \
##    libboost1.74-dev \
##    libboost-chrono1.74-dev \
##    libboost-date-time1.74-dev \
##    libboost-exception1.74-dev \
##    libboost-filesystem1.74-dev \
##    libboost-program-options1.74-dev \
##    libboost-python1.74-dev \
##    libboost-regex1.74-dev \
##    libboost-serialization1.74-dev \
##    libboost-system1.74-dev \
##    libboost-test1.74-dev \
##    libboost-thread1.74-dev \
##    libboost-timer1.74-dev
##
##



2. The PCluster base image includes the Intel MPI library but does not include
the Intel compiler toolchain. Installing the Intel compiler toolchain with apt
shuffles the `/opt/intel` directory badly and places the libraries and tools in
nonstandard locations due to conflicts with the installed MPI library. The
following instructions are used to clear the existing Intel MPI and install a
clean and unified Intel toolchain.

```
rm -rf /opt/intel
rm -rf /var/intel

mkdir -p /opt/intel/src
pushd /opt/intel/src

# Download Intel install assets.
wget -O cpp-compiler.sh https://registrationcenter-download.intel.com/akdlm/IRC_NAS/d85fbeee-44ec-480a-ba2f-13831bac75f7/l_dpcpp-cpp-compiler_p_2023.2.3.12_offline.sh
wget -O fortran-compiler.sh https://registrationcenter-download.intel.com/akdlm/IRC_NAS/0ceccee5-353c-4fd2-a0cc-0aecb7492f87/l_fortran-compiler_p_2023.2.3.13_offline.sh
wget -O tbb.sh https://registrationcenter-download.intel.com/akdlm/IRC_NAS/c95cd995-586b-4688-b7e8-2d4485a1b5bf/l_tbb_oneapi_p_2021.10.0.49543_offline.sh
wget -O mpi.sh https://registrationcenter-download.intel.com/akdlm/IRC_NAS/4f5871da-0533-4f62-b563-905edfb2e9b7/l_mpi_oneapi_p_2021.10.0.49374_offline.sh
wget -O math.sh https://registrationcenter-download.intel.com/akdlm/IRC_NAS/adb8a02c-4ee7-4882-97d6-a524150da358/l_onemkl_p_2023.2.0.49497_offline.sh

# Install the Intel assets.
sh cpp-compiler.sh -a --silent --eula accept 2>&1 | tee install.cpp-compiler.log
sh fortran-compiler.sh -a --silent --eula accept | tee install.fortran-compiler.log
sh tbb.sh -a --silent --eula accept | tee install.tbb.log
sh mpi.sh -a --silent --eula accept | tee install.mpi.log
sh math.sh -a --silent --eula accept | tee install.math.log

popd
```



4. Install lmod. This step must be done as `root`.
```
# Enable gcc using system module
scl enable gcc-toolset-12 bash

# Install lua/lmod manually because apt only has older versions
# that are not compatible with the modern lua modules spack produces
# https://lmod.readthedocs.io/en/latest/030_installing.html#install-lua-x-y-z-tar-gz
sudo su -
mkdir -p /opt/lua/5.1.4.9/src && cd $_
wget https://sourceforge.net/projects/lmod/files/lua-5.1.4.9.tar.bz2
tar -xvf lua-5.1.4.9.tar.bz2
cd lua-5.1.4.9
./configure --prefix=/opt/lua/5.1.4.9 2>&1 | tee log.config
make VERBOSE=1 2>&1 | tee log.make
make install 2>&1 | tee log.install

cat << 'EOF' >> /etc/profile.d/02-lua.sh
# Set environment variables for lua
export PATH="/opt/lua/5.1.4.9/bin:$PATH"
export LD_LIBRARY_PATH="/opt/lua/5.1.4.9/lib:$LD_LIBRARY_PATH"
export CPATH="/opt/lua/5.1.4.9/include:$CPATH"
export MANPATH="/opt/lua/5.1.4.9/man:$MANPATH"
EOF

source /etc/profile.d/02-lua.sh
mkdir -p /opt/lmod/8.7/src
cd /opt/lmod/8.7/src
wget https://sourceforge.net/projects/lmod/files/Lmod-8.7.tar.bz2
tar -xvf Lmod-8.7.tar.bz2
cd Lmod-8.7
# Note the weird prefix, lmod installs in PREFIX/lmod/X.Y automatically
./configure --prefix=/opt/ \
            --with-lmodConfigDir=/opt/lmod/8.7/config \
            2>&1 | tee log.config
make install 2>&1 | tee log.install
ln -sf /opt/lmod/lmod/init/profile /etc/profile.d/z00_lmod.sh
ln -sf /opt/lmod/lmod/init/cshrc /etc/profile.d/z00_lmod.csh
ln -sf /opt/lmod/lmod/init/profile.fish /etc/profile.d/z00_lmod.fish


# Add custom module for system gcc
cat << 'EOF' >> /opt/rh/gcc-toolset-12/gcc-toolset.lua
--%Module
family("compiler")
help([[Wrapper module for gcc-toolset-12 using Software Collections]])
whatis("Description: GCC Toolset 12")

-- Execute 'scl enable' to load gcc-toolset-12 environment
execute {cmd="scl enable gcc-toolset-12 bash", modeA={"load"}}
EOF

# Add a number of default module locations to the lmod startup script.
cat << 'EOF' >> /etc/profile.d/z01_lmod.sh
module use /opt/rh/gcc-toolset-12
EOF

# Log out completely, ssh back into the instance and check if lua modules work
exit
exit
```

3. Install docker
```
# Do this as the "rocky" user, not as root.
# Instructions from https://www.digitalocean.com/community/tutorials/how-to-install-and-use-docker-on-rocky-linux-9

# Install Docker
sudo dnf config-manager --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo
sudo dnf install docker-ce docker-ce-cli containerd.io
sudo systemctl start docker
sudo systemctl enable docker
sudo systemctl status docker

# Add user
sudo usermod -aG docker $(whoami)
# Log out then log back in, and verify docker is working.
docker run hello-world
```

5. Install msql community server
```
# Install mysql.
dnf config-manager --set-enabled crb

dnf install mysql-server mysql-devel
sudo systemctl start mysqld.service
sudo systemctl enable mysqld

# Use the mysql server.
mysql -u root
```

6. Install spack stack for gcc
```
tmux -s gcc
sudo su -
module load gcc-toolset

cd /opt
git clone --recurse-submodules -b release/1.8.0 https://github.com/JCSDA/spack-stack.git spack-stack-1.8
cd /opt/spack-stack-1.8
source setup.sh

spack stack create env --site linux.default --template=unified-dev --name=unified-env-gcc --compiler gcc
cd envs/unified-dev-gcc
spack env activate -p .

export SPACK_SYSTEM_CONFIG_PATH="$PWD/site"

spack external find --scope system \
    --exclude cmake \
    --exclude curl --exclude openssl \
    --exclude openssh --exclude python
spack external find --scope system wget

# Needed for developing R2D2 on localhost.
spack external find --scope system mysql

# Find the compilers. Don't forget to edit the compilers
# yaml to set the fortran compiler for clang.
spack compiler find --scope system

unset SPACK_SYSTEM_CONFIG_PATH

gcc --version
spack config add "packages:all:compiler:[gcc@12.2.1]"
spack config add "packages:all:providers:mpi:[openmpi@5.0.3]"
spack config add "packages:fontconfig:variants:+pic"
spack config add "packages:pixman:variants:+pic"
spack config add "packages:cairo:variants:+pic"

spack concretize 2>&1 | tee log.concretize
${SPACK_STACK_DIR}/util/show_duplicate_packages.py -d [-c] log.concretize
spack install --fail-fast 2>&1 | tee log.install

```

# Install for clang

```
# copy gcc config from above into new directory
cd /opt/spack-stack-1.8/envs
cp -r unified-env-gcc unified-env-clang
cd unified-env-clang

spack env activate -p .
# Edit spack.yaml with these settings
# Compilers: "%clang"
# Providers: clang@17.0.6 and mpich@4.2.1

```
6. Option 1: Testing existing site config in spack-stack (skip steps
8-9 afterwards) this install is done directly on the NFS drive. If you are
testing an update to the configuration, do this on the faster EBS volume (use a
directory in /home/ubuntu) in order to ensure a faster build. Once you have
a verified working spack-stack install you can install it on EFS.

Note: The instructions below focus on the Intel toolchain because it is harder
to build and has some performance benefits over the GNU toolchain, but the
submitted site config can also be used to build the gnu toolchain

```

cd /mnt/experiments-efs
git clone --recurse-submodules -b release/1.7.0 https://github.com/JCSDA/spack-stack.git spack-stack-1.7
cd spack-stack-1.7/
. setup.sh
spack stack create env --site aws-pcluster --template=unified-dev --name=unified-intel
cd envs/unified-intel
spack env activate -p .

# Edit envs/unified-intel/spack.yaml.
# 1) Find this line:
#      compilers: ['%aocc', '%apple-clang', '%gcc', '%intel']
# 2) Delete all compilers except for your target compiler. In the case of intel
#    the line should look like this:
#      compilers: [%intel']

spack concretize 2>&1 | tee log.concretize.001
${SPACK_STACK_DIR}/util/show_duplicate_packages.py -d log.concretize.001
spack install -j 12 --verbose 2>&1 | tee log.install.001
spack module lmod refresh
spack stack setup-meta-modules
```

7. Option 2: Test configuring site from scratch
```
mkdir -p /home/ubuntu/jedi && cd /home/ubuntu/jedi
git clone -b develop --recursive https://github.com/jcsda/spack-stack spack-stack
cd spack-stack/
. setup.sh
spack stack create env --site linux.default --template=unified-dev --name=unified-env
spack env activate -p envs/unified-env

export SPACK_SYSTEM_CONFIG_PATH=/home/ubuntu/jedi/spack-stack/envs/unified-env/site

spack external find --scope system
spack external find --scope system perl
spack external find --scope system python
spack external find --scope system wget
spack external find --scope system texlive
spack external find --scope system mysql

# No external find for pre-installed intel-oneapi-mpi (from pcluster AMI),
# and no way to add object entry to list using "spack config add".
cat << 'EOF' >> ${SPACK_SYSTEM_CONFIG_PATH}/packages.yaml
  intel-oneapi-mpi:
    externals:
    - spec: intel-oneapi-mpi@2021.10.0%intel@2022.1.0
      prefix: /opt/intel
      modules:
      - libfabric-aws/1.19.0amzn4.0
      - intelmpi
EOF

# Add external openmpi
cat << 'EOF' >> ${SPACK_SYSTEM_CONFIG_PATH}/packages.yaml
  openmpi:
    externals:
    - spec: openmpi@4.1.6%gcc@9.4.0~cuda~cxx~cxx_exceptions~java~memchecker+pmi~static~wrapper-rpath
        fabrics=ofi schedulers=slurm
      prefix: /opt/amazon/openmpi
      modules:
      - libfabric-aws/1.19.0amzn4.0
      - openmpi/4.1.6
EOF

# Can't find qt5 because qtpluginfo is broken,
# and no way to add object entry to list using "spack config add".
cat << 'EOF' >> ${SPACK_SYSTEM_CONFIG_PATH}/packages.yaml
  qt:
    buildable: false
    externals:
    - spec: qt@5.15.3
      prefix: /usr
EOF

spack compiler find --scope system

export -n SPACK_SYSTEM_CONFIG_PATH

spack config add "packages:mpi:buildable:False"
spack config add "packages:python:buildable:False"
spack config add "packages:openssl:buildable:False"
spack config add "packages:all:providers:mpi:[intel-oneapi-mpi@2021.10.0, openmpi@4.1.6]"
spack config add "packages:all:compiler:[intel@2021.10.0, gcc@11.4.0]"

# edit envs/unified-env/site/compilers.yaml and replace the following line in the **Intel** compiler section:
#     environment: {}
# -->
#     environment:
#       prepend_path:
#         LD_LIBRARY_PATH: '/opt/intel/oneapi/compiler/2023.2.3/linux/compiler/lib/intel64_lin'
#       set:
#         I_MPI_PMI_LIBRARY: '/opt/slurm/lib/libpmi.so'
```

8. Option 2: To avoid duplicate library versions edit ``envs/unified-dev/site/packages.yaml``
and remove entries for meson, ninja, hdf5, cmake and remove the external
`cmake` and `openssl` entries.


9. Concretize and install
```
spack concretize 2>&1 | tee log.concretize.unified-env.001
./util/show_duplicate_packages.py -d log.concretize.unified-env.001
spack install --verbose 2>&1 | tee log.install.unified-env.001
spack module lmod refresh
spack stack setup-meta-modules
```

10. Test spack-stack installation using your favorite application.

```
# Example given for building jedi-bundle
module use /mnt/experiments-efs/spack-stack-1.7/envs/unified-intel/install/modulefiles/Core
module load stack-gcc/11.4.0
module load stack-openmpi/4.1.6
module load base-env
module load jedi-mpas-env
module load jedi-fv3-env
module load ewok-env
module load sp

git clone https://github.com/JCSDA-internal/jedi-bundle.git
cd jedi-bundle
mkdir build && cd build
ecbuild ../
make update
make -j10
ctest
```

11. (Optional) Remove test installs of spack-stack environments, if desired.

12. Create the AMI for use in the AWS parallelcluster config. You can follow
the official instructions for [Modifying an AWS ParallelCluster AMI](https://docs.aws.amazon.com/parallelcluster/latest/ug/building-custom-ami-v3.html#modify-an-aws-parallelcluster-ami-v3)

13. Use the install to build

```
# Load the intel toolchain into your environment.
source /opt/intel/oneapi/compiler/2023.2.3/env/vars.sh
source /opt/intel/oneapi/mpi/2021.10.0/env/vars.sh

# Activate spack stack modules.
module use /mnt/experiments-efs/spack-stack-1.7/envs/unified-intel/install/modulefiles/Core
module use $HOME/spack-stack-1.7/envs/unified-intel/install/modulefiles/Core
module load stack-intel/2021.10.0
module load stack-intel-oneapi-mpi/2021.10.0
module load base-env
module load jedi-mpas-env
module load jedi-fv3-env
module load ewok-env
module load sp

# Build and test.
git clone https://github.com/JCSDA-internal/jedi-bundle.git
cd jedi-bundle
mkdir build && cd build
ecbuild -DCMAKE_CXX_COMPILER=mpiicpc \
    -DCMAKE_C_COMPILER=mpiicc \
    -DCMAKE_Fortran_COMPILER=mpiifort \
    ../
make update
make -j10
ctest
```

14. Once the parallel cluster head node image is fully configured and tested you
can create an AMI snapshot based on the configured instance using the
[instructions](https://docs.aws.amazon.com/parallelcluster/latest/ug/building-custom-ami-v3.html)
published by AWS. Included here is a short summary of those instructions.

```
# 1) In a SSH session on the instance delete any unwanted files and directories
#    in your home directory and in the /root directory.

# 2) Use the AMI cleanup script to prepare the instance for imaging.
sudo /usr/local/sbin/ami_cleanup.sh
sudo apt clean

# 3) From the AWS console navigate to the build instance and choose
#    "Instance state" and select "Stop instance"

# 4) Create a new AMI from the instance. From at the instance view in the prior
#    step select "Actions" and choose "Image" and then "Create image".
```
