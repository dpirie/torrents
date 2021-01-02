# ZFS Management on Linux

## Which Drive is Which?
I label my drives above the SATA connectors with the Serial Number before I install them so I can tell which one is which. When I migrated from FreeNAS, my zpool device names were just `sda`, `sdb`, etc. Luckily, `smartctl` will tell you the serial number for a given device. If you know that `sdc` is starting to fail, you can run:
```sh
sudo smartctl -a /dev/sdc | grep Serial
Serial Number:    Z302101V
```

## Actually Replacing the Drive
When I had to replace the drive, I tried all sorts of parameters to `zpool replace` to make it happen. The surest way to make it work successfully, was to run `zdb` and find the correct child based on the path property, then grab the guid for that drive. You can then replace the drive with this command (`495...` was the guid of my failing drive). Using the disk by-id path to the drive means that `zpool status` will show the drive id's, including serial numbers, in it's output.
```sh
sudo zpool replace vault 4953133124666804571 /dev/disk/by-id/ata-WDC_WD40EFRX-68N32N0_....
```

## Adding a Spare Drive to the Pool
I bought two drives during a Boxing Day sale, one to replace the failing drive and one more as a hot-spare. I added the spare to the pool with this command:
```sh
sudo zpool add vault spare /dev/disk/by-id/ata-WDC_WD80EFAX-68KNBN0_...
```

