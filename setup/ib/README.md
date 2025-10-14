#!/bin/bash
set -x
# sudo apt update && sudo apt install ifupdown -y
#configure IP,masks,gateway
sudo chattr -i /etc/network/interfaces
sudo chmod a+w /etc/network/interfaces
cat <<EOF > /etc/network/interfaces
auto ibp59s0f1
iface ibp59s0f1 inet static
    address 10.10.1.6
    netmask 255.255.255.0
    mtu 4092
EOF
sudo chmod 644 /etc/network/interfaces
sudo chattr +i /etc/network/interfaces
#restart networking
sudo systemctl restart networking # sudo /etc/init.d/networking restart
