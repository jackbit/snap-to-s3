It worked. Backup without `--dd` makes the restore does not working. So if you have ami, instance, or snapshot and you want to upload on S3 and restore again successfully, here is  steps:

1. Backup the instance snapshot
```
snap-to-s3 --migrate --snapshots snap-xxxx --bucket bucket-name --validate --dd
```

2. To restore (example, in new aws account or vpc ) :

- Launch new EC2, install snap-to-s3, awscli and aws configure
- Create empty volume with size as big as the backup file
- Login ssh to new instance and run `lsblk` to get volume name, example the output like:
```
NAME        MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
loop0         7:0    0 87.9M  1 loop /snap/core/5742
loop1         7:1    0 16.5M  1 loop /snap/amazon-ssm-agent/784
loop2         7:2    0   91M  1 loop /snap/core/6405
loop3         7:3    0 17.9M  1 loop /snap/amazon-ssm-agent/1068
nvme0n1     259:0    0   10G  0 disk
`-nvme0n1p1 259:1    0   10G  0 part /
nvme2n1     259:4    0    9G  0 disk
```

- Resize your `/dev` volume it the size is smaller than backed file `mount -o remount,size=20G /dev`
- Example the volume name `/dev/nvme2n1`, then embed s3 backup on step 1 to it
```
cp "s3://bucketname/image-name-xxxx.img.lz4" - | lz4 -d | sudo dd bs=1M of=/dev/nvme2n1
```

- It will take some times, once success run again `lsblk`, then you will get some thing like this:
```
NAME        MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
loop0         7:0    0 87.9M  1 loop /snap/core/5742
loop1         7:1    0 16.5M  1 loop /snap/amazon-ssm-agent/784
loop2         7:2    0   91M  1 loop /snap/core/6405
loop3         7:3    0 17.9M  1 loop /snap/amazon-ssm-agent/1068
nvme0n1     259:0    0   10G  0 disk
`-nvme0n1p1 259:1    0   10G  0 part /
nvme2n1     259:4    0    9G  0 disk
`-nvme2n1p1 259:5    0    8G  0 part
```

- Start mounting `mount /dev/nvme2n1p1 /mnt` then check folder /mnt, it should contains backed root ebs

3. Create new AMI :

- Create snapshot from new volume of step 2 (it was empty volume but now it is fullfilled)
- Create Image from the snapshot
- Launch the image (some instance types for selection will be disabled because some raw data was not copied properly)
