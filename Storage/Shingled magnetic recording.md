[Zoned Storages](https://zonedstorage.io/docs/introduction/smr) is an excellent read!!!!!

**Shingled magnetic recording** (**SMR**) is a [magnetic storage](https://en.wikipedia.org/wiki/Magnetic_storage "Magnetic storage") data recording technology used in [hard disk drives](https://en.wikipedia.org/wiki/Hard_disk_drive "Hard disk drive") (HDDs) to increase [storage density](https://en.wikipedia.org/wiki/Storage_density "Storage density") and overall per-drive storage capacity. Conventional hard disk drives record data by writing non-overlapping magnetic tracks parallel to each other ([perpendicular magnetic recording](https://en.wikipedia.org/wiki/Perpendicular_magnetic_recording "Perpendicular magnetic recording"), PMR), while shingled recording writes new tracks that overlap part of the previously written magnetic track, leaving the previous track narrower and allowing for higher track density.

This approach was selected because, due to physical limitations, recording magnetic heads are wider than reading heads.

### Fundamental Implications of SMR

Because of the shingled format of SMR, all data streams must be organized and written sequentially to the media. While the methods of SMR implementation may differ (see SMR Implementations section below), the data nonetheless must be written to the media sequentially. Consequently, should a particular data sector need to be modified or re-written, the entire “band” of tracks (zone) must be re-written. Because the modified data sector is potentially under another “shingle” of data, direct modification is not permitted, unlike traditional CMR drives

In the case of SMR, the entire row of shingles above the modified track of the sector to modify needs to be rewritten in the process. SMR hard disks still provide true random-read capability, allowing rapid data access like any traditional CMR drive. This makes SMR an excellent technology candidate for both active archive and higher-performance sequential workloads.

**SMR Interfaces**
To use these drives efficiently, it is necessary to switch to using their backward-incompatible _zone interface_.
-   **Host Managed** This model accommodates only sequential write workloads to deliver both predictable performance and control at the host level. Host-software modifications are required to use host managed SMR drives.
-   **Drive Managed** This model deals with the sequential write constraint internally, providing a bacward compatible interface. Drive managed disks accommodate both sequential and random writing.
-   **Host Aware** This model offers the convenience and flexibility of the drive managed model, that is, backward compatibility with regular disks, while also providing the same host control interface as host managed models.