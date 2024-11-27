# Nginx with Encrypted Client Hello support

This is a fork of an experimental rough implementation of ECH in Nginx (https://github.com/yaroslavros/nginx). We have improved the logic and provided comprehensive, detailed instructions for developers to implement ECH on their websites.

## Environment Configuration

Operating System: Ubuntu 20.04 LTS


Before building [BoringSSL](https://github.com/google/boringssl/tree/master) and [Nginx](https://github.com/nginx/nginx) you need to install build dependencies for your platform such as C11 and C++14 compilers, CMake 3.12 or newer, Perl, Ninja and Gnu Make. Default Nginx modules require libpcre (Ubuntu calls it libpcre3 / libpcre3-dev).

```bash
sudo apt update
sudo apt install build-essential cmake ninja-build perl libpcre3 libpcre3-dev
```

### BoringSSL

BoringSSL does not have releases, so check it out from git

```bash
git clone https://boringssl.googlesource.com/boringssl
cd boringssl
```

Switch to a known stable commit `9bed373` 

```bash
git checkout 9bed373
```

The module requires Go 1.19.1

```bash
cd ../
wget https://go.dev/dl/go1.19.1.linux-amd64.tar.gz
tar -C /usr/local -xzf go1.19.1.linux-amd64.tar.gz
# add go in system environment variable
nano ~/.bashrc
export GOROOT=/usr/local/go
export GOPATH=$HOME/go
export PATH=$PATH:$GOROOT/bin:$GOPATH/bin
source ~/.bashrc
```

Build it with CMake and Ninja

```bash
cmake -DCMAKE_BUILD_TYPE=Release -GNinja -B build
ninja -C build	#This may consume some time
cd ../
```

### Nginx

Clone this repository

```bash
git clone https://github.com/william-hrwang/nginx-with-ECH.git
cd nginx
```

Run configure. Make sure to point includes and libs to your boringssl path. You may need to adjust enabled modules according to your needs and set --prefix if you don't like default /usr/local.

```bash
auto/configure --with-http_v2_module --with-pcre --without-pcre2 --with-http_ssl_module --with-cc-opt="-I../boringssl/include" --with-ld-opt="-L../boringssl/build/ssl -L../boringssl/build/crypto"
```

- with-http_v2_module
  - Enables the HTTP/2 module in the build.
  - HTTP/2 is a faster and more efficient protocol compared to HTTP/1.1.
- --with-pcre
  - Enables support for the Perl Compatible Regular Expressions (PCRE) library.
  - PCRE is used for handling complex regular expressions in configuration files
- --without-pcre2
  - Disables the use of PCRE2, which is a newer version of PCRE
  - If your system has PCRE2 installed, this ensures the build sticks to the older PCRE version.
- --with-http_ssl_module
  - Enables the SSL/TLS module for HTTPS support.
  - This is required for handling encrypted traffic over HTTPS.

Build it and install it.

```bash
make -j4
sudo make install
```

Check that resulting binary is using BoringSSL

```bash
/usr/local/nginx/sbin/nginx -V
nginx version: nginx/1.25.4
built by gcc 9.4.0 (Ubuntu 9.4.0-1ubuntu1~20.04.2) 
built with OpenSSL 1.1.1 (compatible; BoringSSL) (running with BoringSSL)
TLS SNI support enabled
configure arguments: --with-http_v2_module --with-pcre --without-pcre2 --with-http_ssl_module --with-cc-opt=-I../boringssl/include --with-ld-opt='-L../boringssl/build/ssl -L../boringssl/build/crypto'
```