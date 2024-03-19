# silabs-matter-ota

https://docs.silabs.com/matter/2.0.0/matter-overview-guides/ota-bootloader

https://www.silabs.com/documents/public/user-guides/ug489-gecko-bootloader-user-guide-gsdk-4.pdf

https://community.silabs.com/s/article/Matter-Software-Update-Over-The-Air?language=en_US&t=1710664906672

./chip-tool pairing ble-thread 2 hex:0e080000000000010000000300001535060004001fffe00208a042cc9a280a00110708fdf36817046590c20510afccd781c0454410d47d72663f216706030f4f70656e5468726561642d343339330102439304100b8c32cd8ae1c7f3464b2f95fcd6c8dd0c0402a0f7f8 20202021 3840


./commander-cli/commander-cli gbl create MatterDS18B20OverThread.gbl --app MatterDS18B20OverThread.s37

./commander-cli/commander-cli ota create --type matter --input MatterDS18B20OverThread.gbl --vendorid 0xFFF1 --productid 0x8001 --swstring "2.0" --swversion 2 --digest sha256 -o MatterDS18B20OverThread.ota

./connectedhomeip/out/provider/chip-ota-provider-app -f MatterDS18B20OverThread.ota


./connectedhomeip/out/chip-tool otasoftwareupdaterequestor write default-otaproviders '[{"fabricIndex": 1, "providerNodeID": 1, "endpoint": 0}]' 2 0


./connectedhomeip/out/chip-tool accesscontrol write acl '[{"fabricIndex": 1, "privilege": 5, "authMode": 2, "subjects": [112233], "targets": null}, {"fabricIndex": 1, "privilege": 3, "authMode": 2, "subjects": null, "targets": null}]' 1 0

./connectedhomeip/out/chip-tool otasoftwareupdaterequestor announce-otaprovider 1 0 0
