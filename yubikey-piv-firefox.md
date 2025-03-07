# Setting up and using a Yubikey 5 with PIV in Firefox

## Limitations
The Yubico implementation of the PIV application on current Yubikey 5 with firmware 5.7.1 follows the [NIST SP 800-73 document "Cryptographic Algorithms and Key Sizes for PIV"](https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-78-5.pdf) according to [the Yubico documentation](https://developers.yubico.com/PIV/Introduction/YubiKey_and_PIV.html). Looking at tables 1 and 2 of said document it becomes clear that only the following key types are currently supported for use with PIV (*ed25519 is supported by the Yubikey 5 with firmware 5.7.1 and above, but only for GPG*):
- RSA2048 (nearing its end of life in 2030)
- RSA3072 (probably currently still good)
- ECDSA on Curve P-256
- ECDSA on Curve P-384

### Notes:
- I just got this to work with RSA4096, Yubikey firmware 5.7.1, yubico-piv-tool 2.7.1, Firefox ESR 128.7.0esr (64-bit) on Ubuntu 24.04
- RSA4096 also works on Windows 11 with Firefox and yubico-piv-tool 2.7.0. (didn't work with yubico-piv-tool 2.5.1)
- The internet seems to strongly believe that P-256 and P-384 might have some backdoors or smth, which really only leaves RSA3072 at the moment.

### My Suspicion:
Maybe the ykcs11 library can actually handle RSA4096 with PIV, but the Windows 11 certificate store using the Yubico minidriver cannot? Needs testing.

## Step 1: Build and install Yubico's PIV tool and PKCS#11 library (libykcs11.so)
The `yubico-piv-tool` allows you to put keys and certificates onto the Yubikey from the CLI. It is also the only way AFAIK to get keys and certs onto slots 82-95. `libykcs11.so` is a module that directly interacts with the hardware and can be used by Firefox (or more accurately Firefox ESR, see below) for authentication. The Ubuntu packaged version of `yubico-piv-tool` is pretty outdated at the moment and not recommended for use.

1. Install the required packages:
```
sudo apt install -y openssl cmake libtool libssl-dev pkg-config check libpcsclite-dev gengetopt help2man zlib1g-dev build-essential pcscd qrencode
```
2. download latest yubico-piv-tool here:
```
https://developers.yubico.com/yubico-piv-tool/Releases/
```
3. (optional but recommended) check gpg signarure using the procedure listed here:
```
https://developers.yubico.com/Software_Projects/Software_Signing.html
```
4. unpack using `tar -xf filename`
5. build and install using  (remove debug flag if output gets too confusing)
```
mkdir build; cd build
cmake .. -DYKCS11_DBG=2
make
sudo make install
```
6. update shared libraries with
```
sudo ldconfig
```
Once installed, the module will be found by default in /usr/local/lib/libykcs11.so otherwise it will be built locally in yubico-piv-tool/build/ykcs11/libykcs11.so

[source](https://developers.yubico.com/yubico-piv-tool/)


## Step 2: Make some keys and get some certificates onto the Yubikey
This is best done on an airgapped machine, and any keys should probably be backed up in a secure location. Using qrencode to print the (encrypted!!) key to paper is a nice way to do this. Generally, one would probably do something like this:
1. `openssl genrsa -out keyname.key 3072` (create new key)
2. `openssl req -new -key keyname.key -out keyname.csr` (create CSR from existing key)
3. submit CSR to your CA, get a PEM certificate back, let's call it `keyname.pem`
4. `yubico-piv-tool -a import-cert -s <slot> -k -i keyname.pem` (import certificate to a slot - requires mamagement-key)
5. `yubico-piv-tool -a import-key -s <slot> -k -i keyname.key` (import key to the same slot - requires mamagement-key)

In case you want to print the key to paper, maybe do:
- `openssl genrsa -aes256 -out keyname.key 3072` instead of step 1 to get an encrypted key
- `qrencode --8bit --dpi=300 --level=M --output=keyname.png` to get a QR-code; print the resulting image on paper and use a pen or pencil to manually write the passphrase used during key creation next to it.

Considering safety: According to NIST recommendations (no source) AES256 is about as safe as RSA15360 or ECC521 (smth to do with how long brute-forcing would approximately take, given the best currently known algorithms). So as long as your password is 40 random ASCII characters (~256 bits of entropy acc. to [this Wikipedia article](https://en.wikipedia.org/wiki/Password_strength#Random_passwords)) and you use AES256 it should be fine to keep a digital backup of smaller keys (RSA up to 7680 and ECC up to 384 roughly). Breaking the key itself would be easier than breaking the encryption. Keep this in mind when dicking around with trying to get 8kB of data somehow into a QR-code for a paper backup!

## Step 3: Test interfacing with the Yubikey
1. Insert your Yubikey
2. Run `yubico-piv-tool -a status`. It should display the firmware version of your Yubikey as well as its serial number and all the PIV slot information it can find on the device.
3. In case this doesn't work smth is wrong with either pcscd or the ykcs11 build went wrong. Could also be some issue with accessing USB devices due to UDEV-rules or some such. Do not proceed until fixed.

## Step 4 (very optional): Get and "install" the Yubico Authenticator for a nice GUI
1. download latest authenticator release here:
```
[https://developers.yubico.com/yubioath-flutter/Releases/yubico-authenticator-latest-linux.tar.gz](https://developers.yubico.com/yubioath-flutter/Releases/yubico-authenticator-latest-linux.tar.gz)
```
2. unpack using `tar -xf filename`
3. use authenticator binary directly from the folder (no need to build), or
4. make a pseudo install to have a desktop shortcut using `desktop_integration.sh`

## Step 5: Set up Firefox to use your certificate for client authentication
Ubuntu's snap-version of Firefox is currently unable to interact with PKCS#11 modules and hardware-keys. Yubico states otherwise in [this post](https://support.yubico.com/hc/en-us/articles/14744483466908-Firefox-Snap-with-PIV-Authentication), but unfortunately the proposed solution using `sudo snap connect firefox:raw-usb` does not seem to work for Firefox 135.0.1 Snap for Ubuntu. By far the easiest solution that currently *does* work is to install Firefox ESR (Extended Support Release) using `sudo apt install firefox-esr`. This will give you two Firefox's to use - the snap for everyday tasks, and Firefox ESR for your PIV applications. There are also other guides out there ([like this one](https://askubuntu.com/a/1404401)) for completely replacing the snap with a "normal" apt-version of Firefox.

In Firefox, go to Settings -> Privacy & Security -> Security -> Certificates -> Security Devices, then add a new PKCS#11 module with `/usr/local/lib/libykcs11.so`, plug in the Yubikey, click "View Certificates" in Firefox, enter the PIN, and see your certificate there.
