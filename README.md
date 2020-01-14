# Docker on Riotboard using Buildroot

## Getting buildroot

Buildroot version used: 2019.02.9.

Link: `https://buildroot.org/downloads/buildroot-2019.02.9.tar.gz`

```
wget https://buildroot.org/downloads/buildroot-2019.02.9.tar.gz
tar -xzf buildroot-2019.02.9.tar.gz
```

## Buildroot using BR2_EXTERNAL

One note on BR2_EXTERNAL:
The path to a br2-external tree can be either absolute or relative.
If it is passed as a relative path, it is important to note that
it is interpreted relative to the main Buildroot source directory,
*not to the Buildroot output directory*

```
cd buildroot-2019.02.9/

export BR2_EXTERNAL="../riotboard-docker"
export BR2_JLEVEL=24
export O="../riotboard-docker.build"

make riotboard_docker_defconfig
make menuconfig
```

Before trying to compile, check that that the following buildroot config options are set:

 * `BR2_LINUX_KERNEL_CUSTOM_CONFIG_FILE` which depends on `BR2_LINUX_KERNEL_USE_CUSTOM_CONFIG`

 * `BR2_PACKAGE_BUSYBOX_CONFIG`

These variables contain the full path to the linux kernel defconfig & busybox config.

The values for these vars should be as following:

 * `BR2_LINUX_KERNEL_CUSTOM_CONFIG_FILE="${BR2_EXTERNAL}/board/embest/riotboard/linux.config"`

 * `BR2_PACKAGE_BUSYBOX_CONFIG="${BR2_EXTERNAL}/board/embest/riotboard/busybox.config"`

Now you should be ready to build.

__Note__: *You should never use make -jN with Buildroot: top-level parallel make is currently not supported. Instead, use the BR2_JLEVEL option to tell Buildroot to run the compilation of each individual package with make -jN.*

```
make
```

### Buildroot configuration tips

 * make linux-update-defconfig saves the linux configuration to the path specified by BR2_LINUX_KERNEL_CUSTOM_CONFIG_FILE. It simplifies the config file by removing default values. However, this only works with kernels starting from 2.6.33. For earlier kernels, use make linux-update-config.
 * make busybox-update-config saves the busybox configuration to the path specified by BR2_PACKAGE_BUSYBOX_CONFIG.
 * make uclibc-update-config saves the uClibc configuration to the path specified by BR2_UCLIBC_CONFIG.
 * make barebox-update-defconfig saves the barebox configuration to the path specified by BR2_TARGET_BAREBOX_CUSTOM_CONFIG_FILE.
 * make uboot-update-defconfig saves the U-Boot configuration to the path specified by BR2_TARGET_UBOOT_CUSTOM_CONFIG_FILE.


More details in Buildroot documentation, Chapter 9.

After making a change into Buildroot using `make menuconfig`, you can save the changes into a defconfig file like this:

```
make savedefconfig BR2_DEFCONFIG=<path to defconfig file>
```

Or, if you want to always overwrite your board defconfig:

```
export BR2_DEFCONFIG="${BR2_EXTERNAL}/configs/riotboard_docker_defconfig"
make savedefconfig
```


## Riotboard boot switches

```
SD card
1   0  1   0   0  1   0  1
ON OFF ON OFF OFF ON OFF ON

microSD card
1   0  1   0   0  1   1  0
ON OFF ON OFF OFF ON OFF ON

```

## OS image

The final built image should be at `output/images/sdcard.img`.
However, it seems that when you change the default of `BR2_TARGET_ROOTFS_EXT2_SIZE`, the build system breaks the `sdcard.img` rootfs partition, which leads to a kernel panic.
To overcome this issue, you should resize the SD card partition, and flash only `rootfs.ext4` to the first partition using `dd`.

## Firewalld rules for build host to act as router

Remember to add `--permanent` flag to make it persistent (requires `--reload` afterwards)

```
eth_ext="ens1"
eth_int="eno1"

sudo firewall-cmd --direct --permanent --add-rule ipv4 nat POSTROUTING 0 -o $eth_ext -j MASQUERADE
sudo firewall-cmd --direct --permanent --add-rule ipv4 filter FORWARD 0 -i $eth_int -o $eth_ext -j ACCEPT
sudo firewall-cmd --direct --permanent --add-rule ipv4 filter FORWARD 0 -i $eth_ext -o $eth_int -m state --state RELATED,ESTABLISHED -j ACCEPT
```

## Fetch home assistant docker

More detailed info here: `https://www.home-assistant.io/docs/installation/docker/`

```
# docker-compose.yml:

version: '3'
services:
  homeassistant:
    container_name: home-assistant
    image: homeassistant/home-assistant:stable
    volumes:
      - /PATH_TO_YOUR_CONFIG:/config
    environment:
      - TZ=America/New_York
    restart: always
    network_mode: host
	ports:
	  - 8123:8123
```

To start and refresh the container:

```
# start
docker-compose up -d

# reload configuration
docker-compose restart
```
