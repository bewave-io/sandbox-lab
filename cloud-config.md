```bash
echo "== preparing iscsi for bewave=="
set -x
date
apt-get update
apt-get install xfsprogs
mkfs -t xfs /dev/nvme1n1
mkdir /mnt/blabla
mount /dev/nvme1n1 /mnt/blabla
sh -c 'echo "/dev/nvme1n1 /mnt/blabla auto defaults,nofail,comment=cloudconfig 0 2" >> /etc/fstab'
apt-get -y install open-iscsi
 grep "@reboot root sleep 120;service open-iscsi restart" /etc/crontab || sudo sh -c 'echo "@reboot root sleep 120;service open-iscsi restart" >> /etc/crontab'
systemctl enable open-iscsi
echo "== preparing iscsi for bewave DONE=="
reboot
```
