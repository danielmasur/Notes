# add terraform user

    pveum user add terraform-prov@pve --password $PASSWORD
    pveum aclmod / -user terraform-prov@pve -role Administrator

# Create Template
    cd /var/lib/vz/template/iso
    wget http://cdimage.debian.org/images/cloud/bookworm/20240901-1857/debian-12-genericcloud-amd64-20240901-1857.qcow2

    qm create 9000 -name debian-cloudinit -memory 1024 -net0 virtio,bridge=vmbr0 -cores 1 -sockets 1
    qm importdisk 9000 debian-12-genericcloud-amd64-20240901-1857.qcow2 local

    qm set 9000 -scsihw virtio-scsi-pci -virtio0 "local:9000/vm-9000-disk-0.raw"

    qm set 9000 -serial0 socket
    qm set 9000 -boot c -bootdisk virtio0
    qm set 9000 -agent 1
    qm set 9000 -hotplug disk,network,usb
    qm set 9000 -vcpus 1
    qm set 9000 -vga qxl
    qm resize 9000 virtio0 +8G
    qm set 9000 -ide2 local:cloudinit

    qm template 9000

# Terraform
