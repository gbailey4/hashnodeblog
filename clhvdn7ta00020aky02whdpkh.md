---
title: "Slow SMB on Unraid"
datePublished: Sat May 20 2023 02:35:15 GMT+0000 (Coordinated Universal Time)
cuid: clhvdn7ta00020aky02whdpkh
slug: slow-smb-on-unraid
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1684550313136/8cea87ef-0996-4774-acf8-ac3e7b6bdad6.png
tags: homeserver, homelab, smb, unraid

---

By default, my SMB from my Unraid server was very slow (&lt;10% of what it should be, maybe 2-3 MB/s). If you're running into this issue, you may be able to quickly fix this by turning the "Enable SMB Multi Channel" setting to "Yes" from "No".

In order to change this setting, you'll first need to stop your array. You can do that by going to the "Main" tab in your Unraid instance, scrolling to the bottom, and clicking "Stop" in the Array Operation section. NOTE: the array may take a few minutes to stop.

Once the array is stopped find your SMB settings under "Settings" and then click on the "SMB" icon (it's a Windows icon).

From there, see the below photo for an example, you'l turn the "Enable SMB Multi Channel" from "No" (mine was defaulted to this) to "Yes". Click "Apply" and then navigate back to "Main" and start your array again.

The next time you try and send a file over SMB you should see it perform &gt;10x faster! Enjoy.

![Turn Enable SMB Multi Channel to Yes to increase SMB speeds with Unraid](https://cdn.hashnode.com/res/hashnode/image/upload/v1684549823605/dd0e4bf0-cff2-4a84-b7c0-fba8c29657a7.png align="center")