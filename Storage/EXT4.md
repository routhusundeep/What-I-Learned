#EXT4 #linux
[kernelwiki](https://ext4.wiki.kernel.org/index.php/Main_Page)

**Features**
- Compatible with ext3
- Bigger File System and File Sizes
- Sub directory scalability
- Exents
Huge files are split in several extents. Extents improve the performance and also help to reduce the fragmentation, since an extent encourages continuous layouts on the disk.
- Multiblock allocation
- Delayed allocation
- Fast fsck
- Journal checksumming
- "No Journaling" mode
- Online degragmentation
- Inode related features
	- Larger Inodes
	- Inode reservation for directories
	- Nanosecond timestamps
- Persistent Preallocation
- Barriers on by default

**Data Modes**
- writeback mode
ext4 does not journal data at all. This mode provides a similar level of journaling as that of XFS, JFS, and ReiserFS in its default mode - metadata journaling. A crash+recovery can cause incorrect data to appear in files which were written shortly before the crash. This mode will typically provide the best ext4 performance.
- ordered mode
ext4 only officially journals metadata, but it logically groups metadata information related to data changes with the data blocks into a single unit called a transaction. When it’s time to write the new metadata out to disk, the associated data blocks are written first. In general, this mode performs slightly slower than writeback but significantly faster than journal mode.
- journal mode
mode provides full data and metadata journaling. All new data is written to the journal first, and then to its final location. In the event of a crash, the journal can be replayed, bringing both data and metadata into a consistent state. This mode is the slowest except when data needs to be read from and written to disk at the same time where it outperforms all others modes.

[disk layout](https://ext4.wiki.kernel.org/index.php/Ext4_Disk_Layout) containes on-disk format for ext4 filesystems #too_advanced #not_needed