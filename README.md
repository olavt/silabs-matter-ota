# How to perform Over-The-Air firmware upgrades for Silicon Labs based Matter devices
### Author: [Olav Tollefsen](https://www.linkedin.com/in/olavtollefsen/)

## Introduction

This article shows how to perform Over-The-Air firmware upgrades for Silicon Labs based Matter devices.

This article is based on Silicon Labs Gecko SDK version 4.4.1 with Silicon Labs Matter 2.2.0 extensions.

## Prerequisites

You need to build a "Matter Hub" using the Matter repository on Github (https://github.com/project-chip/connectedhomeip). In addition you need an Open Thread Border Router (OTBR).

I will not go into details on how to prepare the Matter Hub / OTBR here.

### Build the OTA Provider Application

```
cd connectedhomeip
```

```
scripts/examples/gn_build_example.sh examples/ota-provider-app/linux out/provider chip_config_network_layer_ble=false
```

When it’s complete, the OTA provider application is located here ./out/provider/chip-ota-provider-app.

## Download and extract the Silicon Labs Commander CLI tool

Download the Silicon Labs Commander command-line tool:
```
wget https://www.silabs.com/documents/public/software/SimplicityCommander-Linux.zip
```

Unzip it:

```
unzip SimplicityCommander-Linux.zip
```

Extract the application files from an archive file for your platform. This command is for Ubuntu Server 64-bit on Raspberry Pi 4:
```
 tar -xf SimplicityCommander-Linux/Commander-cli_linux_aarch64_1v16p5b1567.tar.bz -C ./
 ```

You can now invoke the commander CLI using this command:
```
 ./commander-cli/commander-cli
```

## Prepare the bootloader for Over-The-Air firmware upgrades

See this article for more details on Creating a Gecko Bootloader for Use in Matter OTA Software Update: https://docs.silabs.com/matter/2.0.0/matter-overview-guides/ota-bootloader

Find the "Bootloader - SoC Internal Storage (single image on 1536kB device)" example project and create a new project from it. This bootloader is for the Silicon Labs xG24-DK2601B EFR32xG24 Dev Kit, which has 1536kB Flash.

![Bootloader](./images/bootloader.png)

Open the .slcp file in your bootloader project and select "SOFTWARE COMPONENTS".

Note! Need to add some information about using Simplicity Commander to look at the Flash Map to figure out the Bootloader Storage Slot Setup settings.

Install the "GBL Compression (LZMA)" component under Platform->Bootloader->Core:

![GBL Compression (LZMA)](./images/bootloader-core-gbl-compression-lzma.png)

### Bootloader Storage Slot Setup

This step requires that you have already flashed the Booloader (using default settings) and the Application to the development kit.

To determine the settings, launch the Simplicity Commander tool from Simplicity Studio (click on the Tools button in the menu bar).

Select your development kit from the menu in the upper left corner:

![Simplicity Commander Select Kit](./images/simplicity-commander-select-kit.png)

Select Device Info in the menu to the left:

![Simplicity Commander Device Info](./images/simplicity-commander-device-info.png)

Click on the "Flash Map" button.

![Simplicity Commander Device Info](./images/simplicity-commander-flash-map.png)

In the flash map you will see the used / free space for the flash memory on your development kit.

The OTA firmware needs to fit into the free space (white squares). It's also a good idea to leave some space before the OTA firmware to allow for the application size to grow a bit.

In the example shown above, I use 0x080E000 as the Slot 0 Start Address.

Each white squire is 8192 bytes in length. I will use 5 rows of white squares for the OTA firmware. Each row consist of 14 squares. The setting for the Slot 0 Slot Size can then be calculated to 573,440 bytes (5 * 14 * 8192). This is 0x8C000 in hexadecimal.

In the Software Components settings for your Bootloader project, select the 

![Platform Bootloader Storage Slot Setup](./images/platform-bootloader-storage-slot-setup.png)

Enter the Start Address and Slot Size as determined above in the dialog.

![Platform Bootloader Storage Slot Setup](./images/platform-bootloader-storage-slot-settings.png)

Build the bootloader project, find the .s37 image file (under the Binaries folder) and flash it to your Silicon Labs Dev Kit.

## Prepare the application firmware files

You can use any of the Matter example projects as a starting point.

Remember to set the version numbers for the application here:

![Matter Core Components](./images/matter-core-components.png)

![Matter Core Components Configuration](./images/matter-core-components-configuration.png)

## Prepare the Over-The-Air firmware files

Convert the .s37 firmware file to a GBL file:
```
 ./commander-cli/commander-cli gbl create  --compress lzma <output_file>.gbl --app <input_file>.s37
```

Check that the size of the GBL file fits into the Bootloader Storage Slot 0 size.

```
$ ls -l *.gbl
-rw-rw-r-- 1 ubuntu ubuntu 516592 Mar 26 17:17 MatterSensorOverThread.gbl
```

In the above example the size if 516,592 bytes and will fit into the slot size of 573,440 that was set for the bootloader.

Create the OTA firmware file. Make sure the version number you specifiy in the below command is higher than the version currently running in the device. The Vendor Id and Product Id must also match the settings for your application project.

```
 ./commander-cli/commander-cli ota create --type matter --input <input_file>.gbl --vendorid 0xFFF1 --productid 0x8005 --swstring "3.0" --swversion 3 --digest sha256 -o <output_file>.ota
```

An example:
```
$ ./commander-cli/commander-cli ota create --type matter --input MatterSensorOverThread.gbl --vendorid 0x
FFF1 --productid 0x8002 --swstring "3.0" --swversion 3 --digest sha256 -o MatterSensorOverThread.ota
Creating OTA file...
Writing header data...
Vendor ID              : 0xfff1
Product ID             : 0x8002
Software Version       : 0x00000003
Software Version String: 3.0
Digest Type            : sha256
Writing OTA file MatterSensorOverThread.ota...
DONE
$
```

## Run the OTA Provider application

```
 ./connectedhomeip/out/provider/chip-ota-provider-app -f <name_of_OTA_file>.ota
```

## Commission the OTA Provider into the Matter network

```
cd connectedhomeip/out
./chip-tool pairing onnetwork 999 20202021
```

## Configure the Matter device with the default OTA Provider

This step assumes that the target node for the firmware update already is commissioned to the network.

```
./chip-tool otasoftwareupdaterequestor write default-otaproviders '[{"fabricIndex": 1, "providerNodeID": 999, "endpoint": 0}]' <node_id> 0
```

## Configure the OTA Provider

Configure the OTA Provider with the access control list (ACL) that grants Operate privileges to all nodes in the fabric. This is necessary to allow the nodes to send cluster commands to the OTA Provider:
```
./chip-tool accesscontrol write acl '[{"fabricIndex": 1, "privilege": 5, "authMode": 2, "subjects": [112233], "targets": null}, {"fabricIndex": 1, "privilege": 3, "authMode": 2, "subjects": null, "targets": null}]' 999 0
```

## Initiate the Device Firmware Upgrade procedure
```
./chip-tool otasoftwareupdaterequestor announce-otaprovider <ProviderNodeID> <VendorID> <AnnouncementReason> <ProviderEndpointId> <TargetNodeId> <TargetEndpointId>
```

Example:
```
./chip-tool otasoftwareupdaterequestor announce-otaprovider 999 0 0 0 2 0
```

https://www.silabs.com/documents/public/user-guides/ug489-gecko-bootloader-user-guide-gsdk-4.pdf

https://community.silabs.com/s/article/Matter-Software-Update-Over-The-Air?language=en_US&t=1710664906672

./chip-tool pairing ble-thread 2 hex:0e080000000000010000000300001535060004001fffe00208a042cc9a280a00110708fdf36817046590c20510afccd781c0454410d47d72663f216706030f4f70656e5468726561642d343339330102439304100b8c32cd8ae1c7f3464b2f95fcd6c8dd0c0402a0f7f8 20202021 3840


./commander-cli/commander-cli gbl create MatterDS18B20OverThread.gbl --app MatterDS18B20OverThread.s37

./commander-cli/commander-cli ota create --type matter --input MatterDS18B20OverThread.gbl --vendorid 0xFFF1 --productid 0x8001 --swstring "2.0" --swversion 2 --digest sha256 -o MatterDS18B20OverThread.ota

./connectedhomeip/out/provider/chip-ota-provider-app -f MatterDS18B20OverThread.ota


./connectedhomeip/out/chip-tool otasoftwareupdaterequestor write default-otaproviders '[{"fabricIndex": 1, "providerNodeID": 1, "endpoint": 0}]' 2 0


./connectedhomeip/out/chip-tool accesscontrol write acl '[{"fabricIndex": 1, "privilege": 5, "authMode": 2, "subjects": [112233], "targets": null}, {"fabricIndex": 1, "privilege": 3, "authMode": 2, "subjects": null, "targets": null}]' 1 0

./connectedhomeip/out/chip-tool otasoftwareupdaterequestor announce-otaprovider 1 0 0
