# linux.default

The site config here is a basic site config used for a variety of linux
hosts. Installation on specific significant hosts are described below,


# AWS Ubuntu 24.04 VM

1. Create instance

```
aws ec2 run-instances \
   --image-id "ami-0ea3c35c5c3284d82" --instance-type "m6i.4xlarge" \
   --key-name "eparker-usaf-us-east-2" \
   --block-device-mappings '{"DeviceName":"/dev/sda1","Ebs":{"Encrypted":false,"DeleteOnTermination":true,"Iops":3000,"SnapshotId":"snap-05fb00e35af5550e7","VolumeSize":150,"VolumeType":"gp3","Throughput":125}}' \
   --network-interfaces '{"SubnetId":"subnet-072fb62ff85b32a7a","AssociatePublicIpAddress":true,"DeviceIndex":0,"Groups":["sg-0091fa8e748fbe355"]}' \
   --tag-specifications '{"ResourceType":"instance","Tags":[{"Key":"Name","Value":"ami-gen-clang"}]}' \
   --metadata-options '{"HttpEndpoint":"enabled","HttpPutResponseHopLimit":2,"HttpTokens":"required"}' \
   --private-dns-name-options '{"HostnameType":"ip-name","EnableResourceNameDnsARecord":false,"EnableResourceNameDnsAAAARecord":false}' \
   --count "1" 
```

2. Install base system, configure the environment, and
install local dependencies using the apt package manger.

```
# Configure git credential caching, using 12-hour cache.
git config --global credential.helper 'cache --timeout=43200'

# Install and configure system using root user.
sudo su -
apt update
apt upgrade

apt install \
        bc \
        clang-14 \
        libclang-14-dev \
        libc++-14-dev \
        libomp5-14 \
        libomp-14-dev \
        libc++abi-14-dev \
        gfortran \
        cpp-12 \
        libgomp1 \
        g++-12 \
        gcc-12 \
        gfortran-12 \
        git \
        git-lfs \
        make \
        automake \
        autoconf \
        autopoint \
        mysql-server \
        libmysqlclient-dev \
        qtbase5-dev \
        qt5-qmake \
        libqt5svg5-dev \
        qt5dxcb-plugin \
        wget \
        curl \
        file \
        tcl-dev \
        gnupg2 \
        iproute2 \
        locales \
        python3 \
        python3-pip \
        python3-setuptools \
        unzip \
        vim \
        nano \
        less
```

3. Install lmod

```
# Install lua/lmod manually because apt only has older versions
# that are not compatible with the modern lua modules spack produces
# https://lmod.readthedocs.io/en/latest/030_installing.html#install-lua-x-y-z-tar-gz
sudo su -
mkdir -p /opt/lua/5.1.4.9/src
cd /opt/lua/5.1.4.9/src
wget https://sourceforge.net/projects/lmod/files/lua-5.1.4.9.tar.bz2
tar -xvf lua-5.1.4.9.tar.bz2
cd lua-5.1.4.9
./configure --prefix=/opt/lua/5.1.4.9 2>&1 | tee log.config
make VERBOSE=1 2>&1 | tee log.make
make install 2>&1 | tee log.install

# Set lua variables on startup.
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
```

4. Install spack-stack gcc

Use a persistent virtual terminal (such a 'screen') since installing takes a
long time and a disconnected ssh session may 

```
screen -S install-gcc
sudo su -
git clone --depth 1 -b spack-stack-1.8.0 \
    --recursive https://github.com/jcsda/spack-stack \
    /opt/spack-stack
cd /opt/spack-stack
source setup.sh
# Swap default module type for default linux.
sed -i 's/tcl/lmod/g' configs/sites/tier2/linux.default/modules.yaml


spack stack create env --site linux.default --template unified-dev --name unified-env-gcc --compiler=gcc
cd envs/unified-env-gcc 
spack env activate -p .
export SPACK_SYSTEM_CONFIG_PATH="$PWD/site"
spack external find --scope system \
    --exclude cmake \
    --exclude curl --exclude openssl \
    --exclude openssh --exclude python
spack external find --scope system wget
spack external find --scope system mysql
spack compiler find --scope system
unset SPACK_SYSTEM_CONFIG_PATH

# ACTION: Edit the site/compilers.yaml with the following.
#   1) Delete or comment gcc-13 refs and preserve only gcc-12
#   2) Delete or comment clang refs.

# ACTION: Edit the site/packages.yaml and add these packages
# If not present.
  gcc:
    buildable: false
    externals:
    - spec: gcc@12.3.0
      prefix: /usr
  gcc-runtime:
    buildable: false
    externals:
    - spec: gcc-runtime@12.3.0
      prefix: /usr
  qt:
    buildable: false
    externals:
    - spec: qt@5.15.3
      prefix: /usr
      version: [5.15.3]


# Continue configuration.
spack config add "packages:all:compiler:[gcc@12.3.0]"
spack config add "packages:all:providers:mpi:[openmpi@5.0.3]"
spack config add "packages:fontconfig:variants:+pic"
spack config add "packages:pixman:variants:+pic"
spack config add "packages:cairo:variants:+pic"
spack config add "packages:ewok-env:variants:+mysql"


# Concretize and install
spack concretize 2>&1 | tee log.concretize
${SPACK_STACK_DIR}/util/show_duplicate_packages.py -d -c log.concretize
spack install --verbose --fail-fast 2>&1 | tee log.install

# Install modules
spack module lmod refresh
spack stack setup-meta-modules

# Add a number of default module locations to the lmod startup script.
cat << 'EOF' >> /etc/profile.d/z01_lmod.sh
module use /opt/spack-stack/envs/unified-env-gcc/install/modulefiles/Core
EOF
```


3. Install spack-stack clang

Use a persistent virtual terminal (such a 'screen') since installing takes a
long time and a disconnected ssh session may 

```
screen -S install-clang
sudo su -

# Use spack-stack source directory from gcc install.
cd /opt/spack-stack
source setup.sh

spack stack create env \
        --site linux.default \
        --template unified-dev \
        --name unified-env-clang \
        --compiler=clang

cd envs/unified-env-clang
spack env activate -p .
export SPACK_SYSTEM_CONFIG_PATH="$PWD/site"
spack external find --scope system \
    --exclude cmake \
    --exclude curl --exclude openssl \
    --exclude openssh --exclude python
spack external find --scope system wget
spack external find --scope system mysql
spack compiler find --scope system
unset SPACK_SYSTEM_CONFIG_PATH

# Continue configuration.
spack config add "packages:all:compiler:[gcc@14.0.6]"
spack config add "packages:all:providers:mpi:[mpich@4.2.1]"
spack config add "packages:fontconfig:variants:+pic"
spack config add "packages:pixman:variants:+pic"
spack config add "packages:cairo:variants:+pic"
spack config add "packages:ewok-env:variants:+mysql"


# ACTION: Edit the site/compilers.yaml with the following.
#   1) Delete or comment gcc-12 refs and preserve only gcc-12
#   2) Under clang, set f77 and fc paths to "/usr/bin/gfortran-12".
#   3) Under clang add the following to environment:
#      prepend_path:
#        LD_LIBRARY_PATH: '/usr/lib/llvm-14/lib'"

# ACTION: Edit the site/packages.yaml and add these packages
# If not present.
  gcc:
    buildable: false
    externals:
    - spec: gcc@12.3.0
      prefix: /usr
  gcc-runtime:
    buildable: false
    externals:
    - spec: gcc-runtime@12.3.0
      prefix: /usr
  qt:
    buildable: false
    externals:
    - spec: qt@5.15.3
      prefix: /usr
      version: [5.15.3]


# Concretize and install
spack concretize 2>&1 | tee log.concretize
${SPACK_STACK_DIR}/util/show_duplicate_packages.py -d -c log.concretize
spack install --verbose --fail-fast 2>&1 | tee log.install

# Install modules
spack module lmod refresh
spack stack setup-meta-modules
```

5. Install spack-stack intel

Use a persistent virtual terminal (such a 'screen') since installing takes a
long time and a disconnected ssh session may 

```
# First install the intel toolchain
sudo su -
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

rm -rf /opt/intel/src/*

popd
```

Install spack-stack

```
cd /opt/spack-stack
source setup.sh

spack stack create env \
        --site linux.default \
        --template unified-dev \
        --name unified-env-intel \
        --compiler=intel


cd envs/unified-env-clang
spack env activate -p .

source /opt/intel/oneapi/compiler/2023.2.3/env/vars.sh
source /opt/intel/oneapi/mpi/2021.10.0/env/vars.sh


export SPACK_SYSTEM_CONFIG_PATH="$PWD/site"
spack external find --scope system \
    --exclude cmake \
    --exclude curl --exclude openssl \
    --exclude openssh --exclude python
spack external find --scope system wget
spack external find --scope system mysql
spack compiler find --scope system
unset SPACK_SYSTEM_CONFIG_PATH

# Continue configuration.
spack config add "packages:all:compiler:[intel@2021.10.0]"
spack config add "packages:all:providers:mpi:[intel-oneapi-mpi@2021.10.0]"
spack config add "packages:fontconfig:variants:+pic"
spack config add "packages:pixman:variants:+pic"
spack config add "packages:cairo:variants:+pic"
spack config add "packages:ewok-env:variants:+mysql"


# ACTION: Edit the site/compilers.yaml with the following.
#   1) Delete or comment gcc-13 refs and preserve only gcc-12
#   2) Under intel add the following to environment:
#      environment:
#        prepend_path:
#          LD_LIBRARY_PATH: '/opt/intel/oneapi/compiler/2023.2.3/linux/compiler/lib/intel64_lin'

# ACTION: Edit the site/packages.yaml and add these packages
# If not present.
  gcc:
    buildable: false
    externals:
    - spec: gcc@12.3.0
      prefix: /usr
  gcc-runtime:
    buildable: false
    externals:
    - spec: gcc-runtime@12.3.0
      prefix: /usr
  qt:
    buildable: false
    externals:
    - spec: qt@5.15.3
      prefix: /usr
      version: [5.15.3]


# Concretize and install
spack concretize 2>&1 | tee log.concretize
${SPACK_STACK_DIR}/util/show_duplicate_packages.py -d -c log.concretize
spack install --verbose --fail-fast 2>&1 | tee log.install

# Install modules
spack module lmod refresh
spack stack setup-meta-modules
```

6. Add module files to environment
````
cat << 'EOF' >> /etc/profile.d/z01_lmod.sh
module use /opt/spack-stack/envs/unified-env-gcc/install/modulefiles/Core
module use /opt/spack-stack/envs/unified-env-clang/install/modulefiles/Core
EOF

```
