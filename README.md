```
#install qemu4
yum -y install glib2-devel pixman-devel python2 python3
cwd=$(pwd)
wget https://download.qemu.org/qemu-4.0.0.tar.bz2  -P /opt/.
cd /opt
tar xvjf qemu-4.0.0.tar.bz2
cd qemu-4.0.0
./configure --python=/usr/bin/python2
make -j && make install
cd $cwd
#ln -s /usr/local/bin/qemu-system-x86_64 /usr/bin/qemu-system-x86_64

yum -y install libguestfs-tools libguestfs-xfs virt-top
echo "Checking QEMU-KVM Support, Resolve any error it throws"
virt-host-validate

echo "Checking for BIOS Intel VTd/IOMMU Support"
dmesg | grep -e DMAR -e IOMMU -e Virtualization

echo "Checking kernel parameter for PCI Passthrough /IOMMU"
cat /proc/cmdline | grep intel_iommu
cat /proc/cmdline | grep vfio_iommu_type1.allow_unsafe_interrupts

echo "please add ==>intel_iommu=on vfio_iommu_type1.allow_unsafe_interrupts=1<== in grub.cg"
echo "For Ubuntu -> Edit etc/default/grub  , sudo update-grub , check /boot/grub/grub.cfg"

#remove qed* driver for pci passthrough
modprobe -rv qedr
modprobe -rv qede
modprobe -rv qed

#hack for dns
sed -i '1s/^/nameserver 8.8.8.8\n/' /etc/resolv.conf
echo "==================SUPPORTED OS ========================="
virt-builder --list | grep centos
echo "========================================================"
#echo "Enter OS from above list- Recommended centos-7.8"
#read OS

OS=centos-7.8

sudo chmod 0666 /dev/kvm
# /etc/libvirt/qemu.conf user=root /group=root restart service

#generate pub key
ssh-keygen -o

#P4 REPO
echo $OS
if [[ "$OS" < centos-8 ]]; then
    echo "centos7."
    baseurl="http://package.perforce.com/yum/rhel/7/x86_64"
else
    echo "centos8."
    baseurl="http://package.perforce.com/yum/rhel/8/x86_64"
fi
echo $baseurl

#create devel-provision file for vm(to be use in firstboot
cat <<'EOF' > /tmp/provision2.sh
sudo rpm --import https://package.perforce.com/perforce.pubkey
cat <<EOT >> /etc/yum.repos.d/perforce.repo
[perforce]
name=Perforce
baseurl=BASEURL
enabled=1
gpgcheck=1
EOT
sudo yum -y install helix-p4d
sudo yum -y install indent
sudo yum -y install iproute2
EOF
##endo of file
sed  "s|BASEURL|$baseurl|g" /tmp/provision2.sh > /tmp/provision.sh

echo "Enter perforce User name"
read USER
#create devel-provision file for vm(to be use in firstboot
cat <<'EOF' > p4config
#creating p4config environment variable file
export P4PORT=qidlp401.qlc.com:1667
#export P4PORT=avp401.qlc.com:1667
export P4USER=$USER
export P4CLIENT=$USER_$OS-VM
export P4CONFIG=p4config
export P4DIFF="diff --show-c-function"
EOF
##endo of file

sed -i '1s/^/user="root"\n group="root"\n/' /etc/libvirt/qemu.conf


#--selinux-relabel \

export LIBGUESTFS_BACKEND=direct

virt-builder $OS \
--size=40G --format qcow2 -o $HOME/$OS.qcow2 \
--hostname $OS-VM \
--network \
--timezone Asia/Kolkata \
--root-password password:qlogic \
--ssh-inject root:file:$HOME/.ssh/id_rsa.pub \
--install "lshw,vim,pciutils,kernel-devel,m4,make,module-init-tools,ncurses-devel,net-tools,\
newt-devel,numactl-devel,openssl,asciidoc,bc,binutils,binutils-devel,bison,\
diffutils,elfutils,rpm-build,sh-utils,tar,xmlto,xz,zlib-devel,dhclient,bind-export-libs,gcc,elfutils-libelf-devel,git" \
--firstboot-command "cp /usr/lib64/bind9-export/* /usr/lib64/.;dhclient enp2s0" \
--firstboot /tmp/provision.sh \
--upload p4config:/root \
--edit '/etc/default/grub:s/^GRUB_CMDLINE_LINUX_DEFAULT=.*/GRUB_CMDLINE_LINUX_DEFAULT="console=tty0 console=ttyS0,115200n8"/' \
--edit '/etc/sysconfig/selinux:s/^SELINUX=.*/SELINUX=disabled/' \
--run-command "grub2-mkconfig -o /boot/grub2/grub.cfg"


#################MACHINE XML SECTION######################
cat <<'EOF' >  vm-test-builder.xml
<domain type='kvm' id='5'>
<name>VM_NAME</name>
  <memory unit='KiB'>4194304</memory>
  <currentMemory unit='KiB'>4194304</currentMemory>
  <vcpu placement='static'>4</vcpu>
  <resource>
    <partition>/machine</partition>
  </resource>
  <os>
    <type arch='x86_64' machine='q35'>hvm</type>
    <boot dev='hd'/>
  </os>
  <devices>
    <emulator>/usr/local/bin/qemu-system-x86_64</emulator>
    <disk type='file' device='disk'>
      <driver name='qemu' type='qcow2'/>
      <source file='OS_QCOW2'/>
      <backingStore/>
      <target dev='vda' bus='virtio'/>
      <alias name='virtio-disk0'/>
      <address type='pci' domain='0x0000' bus='0x04' slot='0x00' function='0x0'/>
    </disk>
    <controller type='virtio-serial' index='0'>
      <alias name='virtio-serial0'/>
      <address type='pci' domain='0x0000' bus='0x03' slot='0x00' function='0x0'/>
    </controller>
    <interface type='bridge'>
      <source bridge='virbr0'/>
      <target dev='vnet0'/>
      <model type='virtio'/>
      <alias name='net0'/>
      <address type='pci' domain='0x0000' bus='0x01' slot='0x00' function='0x0'/>
    </interface>
    <interface type='network'>
      <source network='default' bridge='virbr0'/>
      <target dev='vnet1'/>
      <model type='rtl8139'/>
      <alias name='net1'/>
      <address type='pci' domain='0x0000' bus='0x02' slot='0x00' function='0x0'/>
    </interface>
    <console type='pty' tty='/dev/pts/0'>
      <source path='/dev/pts/0'/>
      <target type='serial' port='0'/>
      <alias name='serial0'/>
    </console>
    <graphics type='vnc' port='5901' autoport='yes' listen='127.0.0.1'>
      <listen type='address' address='127.0.0.1'/>
    </graphics>
  </devices>
</domain>

EOF
############################################################
sed  "s|OS_QCOW2|$HOME/$OS.qcow2|g" vm-test-builder.xml > vm-builder2.xml
sed  "s|VM_NAME|$OS|g" vm-builder2.xml > vm-builder3.xml

#permission issue
setenforce 0

virsh create vm-builder3.xml
virsh list
echo "Connect to any VM list listed above using virsh console <VMNAME>"
echo "to exit out of VM -> Press ctrl+ ]"

##TODO ATTACHING PF as passthrough
modprobe -v vfio-pci


lspci | grep QL
echo -e "Please Enter detials of PF to passthrough inside VM"
echo -e "Enter Bus of PCI:(3b)"
read BUSX
echo -e "Enter dev of PCI:(00)"
read DEVX
echo -e "Enter function of PCI:(01)"
read FUNCX


#build pf passthrough file
cat <<'EOF' >  add_pf_pci.xml
<hostdev mode='subsystem' type='pci' managed='yes'>
<driver name='vfio'/>
<source>
<address domain='0x0000' bus='0xBUSX' slot='0xDEVX' function='0xFUNCX'/>
</source>
<alias name='hostdev0'/>
</hostdev>
EOF
sed -i "s/BUSX/$BUSX/" add_pf_pci.xml
sed -i "s/DEVX/$DEVX/" add_pf_pci.xml
sed -i "s/FUNCX/$FUNCX/" add_pf_pci.xml
##endo of file


virsh attach-device centos-7.8 add_pf_pci.xml --live
virsh reboot centos-7.8
echo "Please execute "virsh console centos-7.8" to enter inside VM"
#for unknown feature amd-sev-es errro - "yum downgrade edk2-ovmf"
```
