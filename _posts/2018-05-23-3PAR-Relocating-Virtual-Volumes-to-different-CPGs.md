---
title: "3PAR - Relocating Virtual Volumes to Different CPGs"
excerpt_separator: "<!--more-->"
categories:
  - Blog
tags:
  - 3PAR
  - Storage
  - HPE
  - SAN
---

There may come a time when you wish to move a 3PAR Virtual volume over to a different CPG, either there was a hardware/OS change that has new features in the CPG level that you would like to take advantage of, you do a report based on CPG’s or maybe to get some dedup optimization.

There are two ways to initiate a VV migration to a different CPG, you can either use the CLI, or you can use the StorServ Management Console.


When using the CLI, you need the same of the new CPG and the VV that’s moved. Remember that the CLI is case sensitive.

In my case I wanted to move the snapshots over the same CPG with the full volume, where snp_cpg is the flag for relocating snapshot data, ‘Infra_SSD_Raid6’ is the CPG that I want to move the VV to and ‘ESXVV-1’ is the VV that I want to move.

```
tunevv snp_cpg Infra_SSD_Raid6 ESXVV-1
```

If you want to move the actual volume and not the snapshot, the command uses the usr_cpg flag instead.

```
tunevv usr_cpg Infra_SSD_Raid6 ESXVV-1
```

When using the SSMC find the volume

Actions > tune

Select either “user space or “copy space” > select either “current CPG” or “move to another CPG,” if moving to another CPG select the new one.

Click tune.

Once the migration completes, you should compact the CPG’s after your done. To reclaim the space, the CPG has expanded to that was taken up from the volume that you’ve migrated to the different CPG.
