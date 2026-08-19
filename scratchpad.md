### scratchpad - random notes ###


#### ARM binary exploitation: ####

https://github.com/bkerler/exploit_me


##### Setup #####

From exploit_me/scripts/setup.sh:
```
#!/bin/sh
sudo apt-get update
sudo apt-get install -y git libssl-dev libffi-dev build-essential libncurses5-dev 
sudo apt-get install -y gcc-arm-linux-gnueabi g++-arm-linux-gnueabi
sudo apt-get install -y gcc-aarch64-linux-gnu g++-aarch64-linux-gnu
sudo apt-get install -y qemu qemu-user-binfmt binfmt-support
sudo systemctl restart systemd-binfmt
sudo apt-get install -y gdb-multiarch
sudo cp /usr/arm-linux-gnueabi/lib/ld-linux.so.3 /lib 
sudo cp /usr/arm-linux-gnueabi/lib/libgcc_s.so.1 /lib
sudo cp /usr/arm-linux-gnueabi/lib/libc.so.6 /lib
```

pwndbg install:
```
curl --proto '=https' --tlsv1.2 -LsSf 'https://install.pwndbg.re' | sh -s -- -t pwndbg-gdb
```
OR grab a release:
```
https://github.com/pwndbg/pwndbg/releases
```

pwntools:
```
python3 -m venv ./pwntools
cd pwntools
bin/python -m pip install --upgrade pip
bin/python -m pip install --upgrade pwntools
```
