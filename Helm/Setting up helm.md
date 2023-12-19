# Setup Helm on online or offiline (air-gapped) machine 

## Install using the binary release (offline)
Every release of Helm provides binary releases for a variety of OSes. These binary versions can be manually downloaded and installed.
	1. Download your desired version
	2. Unpack it (tar -zxvf helm-v3.0.0-linux-amd64.tar.gz)
Find the helm binary in the unpacked directory, and move it to its desired destination (mv linux-amd64/helm /usr/local/bin/helm)
```
tar -zxvf helm-v3.0.0-linux-amd64.tar.gz
mv linux-amd64/helm /usr/local/bin/helm
helm version
```

## Install using script (online)
You can fetch that script, and then execute it locally. It's well documented so that you can read through it and understand what it is doing before you run it.
```
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```