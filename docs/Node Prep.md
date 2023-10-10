<!-- markdownlint-disable MD013 -->
# 💻 Machine Preparation

## System requirements

📍 _k3s default behaviour is that all nodes are able to run workloads, including control nodes. Worker nodes are therefore optional._

📍 _If you have 3 or more nodes it is strongly recommended to make 3 of them control nodes for a highly available control plane._

📍 _Ideally you will run the cluster on bare metal machines. If you intend to run your cluster on Proxmox VE, my thoughts and recommendations about that are documented [here](https://onedr0p.github.io/home-ops/notes/proxmox-considerations.html)._

| Role    | Cores    | Memory        | System Disk               |
|---------|----------|---------------|---------------------------|
| Control | 4 _(6*)_ | 8GB _(24GB*)_ | 100GB _(500GB*)_ SSD/NVMe |
| Worker  | 4 _(6*)_ | 8GB _(24GB*)_ | 100GB _(500GB*)_ SSD/NVMe |
| _\* recommended_ |

## Debian for AMD64

1. Download the latest stable release of Debian from [here](https://cdimage.debian.org/debian-cd/current/amd64/iso-dvd), then follow [this guide](https://www.linuxtechi.com/how-to-install-debian-12-step-by-step) to get it installed. Deviations from the guide:

    ```txt
    Choose "Guided - use entire disk"
    Choose "All files in one partition"
    Delete Swap partition
    Uncheck all Debian desktop environment options
    ```

2. [Post install] Remove CD/DVD as apt source

    ```sh
    su -
    sed -i '/deb cdrom/d' /etc/apt/sources.list
    apt update
    exit
    ```

3. [Post install] Enable sudo for your non-root user

    ```sh
    su -
    apt update
    apt install -y sudo
    usermod -aG sudo ${username}
    echo "${username} ALL=(ALL) NOPASSWD:ALL" | tee /etc/sudoers.d/${username}
    exit
    newgrp sudo
    sudo apt update
    ```

4. [Post install] Add SSH keys (or use `ssh-copy-id` on the client that is connecting)

    📍 _First make sure your ssh keys are up-to-date and added to your github account as [instructed](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/adding-a-new-ssh-key-to-your-github-account)._

    ```sh
    mkdir -m 700 ~/.ssh
    sudo apt install -y curl
    curl https://github.com/${github_username}.keys > ~/.ssh/authorized_keys
    chmod 600 ~/.ssh/authorized_keys
    ```

## Debian for RasPi4

📍 _If you choose to use a Raspberry Pi 4 for the cluster, it is recommended to have an 8GB model. Most important is to **boot from an external SSD/NVMe** rather than an SD card. This is supported [natively](https://www.raspberrypi.com/documentation/computers/raspberry-pi.html), however if you have an early model you may need to [update the bootloader](https://www.tomshardware.com/how-to/boot-raspberry-pi-4-usb) first._

📍 _Be sure to check the [power requirements](https://www.raspberrypi.com/documentation/computers/raspberry-pi.html#power-supply) if using a PoE Hat and a SSD/NVMe dongle._

1. Download the latest stable release of Debian from [here](https://raspi.debian.net/tested-images). _**Do not** use Raspbian or DietPi or any other flavor Linux OS._

2. Flash the image onto an SSD/NVMe drive.

3. Re-mount the drive to your workstation and then do the following (per the [official documentation](https://raspi.debian.net/defaults-and-settings)):

    ```txt
    Open 'sysconf.txt' in a text editor and save it upon updating the information below
      - Change 'root_authorized_key' to your desired public SSH key
      - Change 'root_pw' to your desired root password
      - Change 'hostname' to your desired hostname
    ```

4. Connect SSD/NVMe drive to the Raspberry Pi 4 and power it on.

5. [Post install] SSH into the device with the `root` user and then create a normal user account with `adduser ${username}`

6. [Post install] Follow steps 3 and 4 from [Debian for AMD64](#debian-for-amd64).

7. [Post install] Install `python3` which is needed by Ansible.

    ```sh
    sudo apt install -y python3
    ```

## 🚀 Getting Started

Once you have installed Debian on your nodes, there are 6 stages to getting a Flux-managed cluster up and running.