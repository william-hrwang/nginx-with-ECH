
# Experimental fork of Nginx with Encrypted Client Hello support

This is a fork of nginx master with added [ECH](https://datatracker.ietf.org/doc/draft-ietf-tls-esni/) support. It implements `ssl_ech` configuration directive, `$ssl_ech` and `$ssl_ech_config` variables.

## Building
Before building BoringSSL and Nginx you need to install build dependencies for your platform such as C11 and C++14 compilers, CMake 3.12 or newer, Perl, Ninja and Gnu Make. Default Nginx modules require libpcre (Ubuntu calls it libpcre3 / libpcre3-dev).

### BoringSSL
BoringSSL does not have releases, so check it out from git

    git clone https://boringssl.googlesource.com/boringssl
    cd boringssl
Master branch should be fine. If something breaks, switch to "known good" tag `9bed373` that worked for me.
Build it with CMake and Ninja:

    cmake -DCMAKE_BUILD_TYPE=Release -GNinja -B build
    ninja -C build
    cd ..
### Nginx
Clone this repository

    git clone https://github.com/yaroslavros/nginx
    cd nginx
Run configure. Make sure to point includes and libs to your boringssl path. You may need to adjust enabled modules according to your needs and set --prefix if you don't like default /usr/local.

    auto/configure --with-http_v2_module --with-pcre --without-pcre2 --with-http_ssl_module --with-cc-opt="-I../boringssl/include" --with-ld-opt="-L../boringssl/build/ssl -L../boringssl/build/crypto"

Build it and install it.

    make -j4
    sudo make install
Check that resulting binary is using BoringSSL

    /usr/local/bin/nginx -V
    nginx version: nginx/1.25.4
    built by gcc 13.2.1 20230826 (Gentoo 13.2.1_p20230826 p7)
    built with OpenSSL 1.1.1 (compatible; BoringSSL) (running with BoringSSL)
    TLS SNI support enabled
    configure arguments: --prefix=/usr/local --with-http_v2_module --with-pcre --without-pcre2 --with-http_ssl_module --with-cc-opt=-I../boringssl/include --with-ld-opt='-L../boringssl/build/ssl -L../boringssl/build/crypto'
## Configuration
### ssl_ech configuration directive
To enable ECH for a given server configure ssl_ech as follows:
> ssl_ech *public_name* *config_id* *[key=file]* [noretry]

- *public_name* is mandatory. It needs to be set to FQDN to be populated in clear-text SNI of Outer ClientHello. It's highly recommended to have a server block matching that *public_name* and providing a valid certificate for it, otherwise ECH retry mechanism will not work.
- *config_id* is mandatory. It is a number between 0 and 255 identifying ECH configuration. Running multiple configurations with the same id is possible but will reduce performance as server will need to try multiple encryption keys.
- *key=file* is optional. It specifies a *file* with PEM encoded X25519 private key. If it is not specified, key will be generated dynamically on each restart/configuration reload. It is highly recommended to generate and use a static key unless you have DNS automation to update HTTPS DNS record each time new key is generated.
- *noretry* is an optional flag to remove given configuration from retry list or generated ECHConfigList for DNS record. It should be used for historic rotated out keys that may still be used by clients due to caching. Valid configuration requires at least one `ssl_ech` entry without `noretry` flag.

It is possible to have multiple `ssl_ech` configurations in a given server block. `ssl_ech` configurations from multiple server blocks under the same listener will be automatically aggregated. Note that TLS 1.3 must be enabled for `ssl_ech` to be accepted.
### Generating ECH key
The only KEM supported for ECH in BoringSSL is X25519, HKDF-SHA256, so X25519 key is required. To generate one with OpenSSL run

    openssl genpkey -out ech.key -algorithm X25519
### Populating DNS records
After parsing configuration Nginx will dump encoded ECHConfigList into error_log similarly to 

    server ech.example.com ECH config for HTTPS DNS record ech="AEX+DQBB8QAgACBl2nj6LhmbUqJJseiydASRUkdmEQGq/u/e5fXDLsFJSAAEAAEAAQASY2xvdWRmbGFyZS1lY2guY29tAAA="
For ECH to work this encoded configuration needs to be added to HTTPS record. Typical HTTPS record looks like this:

    kdig +short crypto.cloudflare.com https
    1 . alpn=http/1.1,h2 ipv4hint=162.159.137.85,162.159.138.85 ech=AEX+DQBB8QAgACBl2nj6LhmbUqJJseiydASRUkdmEQGq/u/e5fXDLsFJSAAEAAEAAQASY2xvdWRmbGFyZS1lY2guY29tAAA= ipv6hint=2606:4700:7::a29f:8955,2606:4700:7::a29f:8a55
For ECH operation only `ech` is required, other attributes are optional.
## Testing
If everything is configured correctly, Chrome 117+ and Firefox 118+ will be using *public_name* in  in clear-text SNI of Outer ClientHello - confirm this with Wireshark. Note that for ECH to work Firefox requires DoH. Chrome supports ECH with any DNS as long as it resolves HTTPS records.
### Variables
There are two variables set for ECH: `$ssl_ech` and `$ssl_ech_config`

- `$ssl_ech` is set to "1" if ECH was successfully negotiated
- `$ssl_ech_config` is set to server-specific ECHConfigList mentioned above. It can be used to generate json according to [draft-ietf-tls-wkech-04](https://datatracker.ietf.org/doc/draft-ietf-tls-wkech/)

      location /.well-known/origin-svcb {
          add_header Content-Type application/json;
          return 200 '{"endpoints":[{"ech":"$ssl_ech_config"}]}';
      }
