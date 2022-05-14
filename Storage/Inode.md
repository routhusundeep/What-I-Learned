#linux
[wikipedia](https://en.wikipedia.org/wiki/Inode) and [archwiki](https://man.archlinux.org/man/inode.7.en) has some nice info on inode

Within a POSIX system, a file has the following attributes  which may be retrieved by the [stat](https://en.wikipedia.org/wiki/Stat_(Unix) "Stat (Unix)") system call:

-   Device ID (this identifies the device containing the file; that is, the scope of uniqueness of the serial number).
-   File serial numbers.
-   The [file _mode_](https://en.wikipedia.org/wiki/File_system_permissions "File system permissions") which determines the file type and how the file's owner, its group, and others can access the file.
-   A [link count](https://en.wikipedia.org/wiki/Reference_counting "Reference counting") telling how many [hard links](https://en.wikipedia.org/wiki/Hard_link "Hard link") point to the inode.
-   The [User ID](https://en.wikipedia.org/wiki/User_identifier_(Unix) "User identifier (Unix)") of the file's owner.
-   The [Group ID](https://en.wikipedia.org/wiki/Group_identifier_(Unix) "Group identifier (Unix)") of the file.
-   The device ID of the file if it is a [device file](https://en.wikipedia.org/wiki/Device_file "Device file").
-   The size of the file in [bytes](https://en.wikipedia.org/wiki/Byte "Byte").
-   [Timestamps](https://en.wikipedia.org/wiki/Timestamp "Timestamp") telling when the inode itself was last modified (ctime, _inode change time_), the file content last modified (mtime, _modification time_), and last accessed (atime, _access time_).
-   The preferred I/O block size.
-   The number of blocks allocated to this file.


**Some Key Points:**
- Files can have multiple names.
- An inode may have no links.
- It is typically not possible to map from an open file to the filename that was used to open it
- Historically, it was possible to [hard link](https://en.wikipedia.org/wiki/Hard_link "Hard link") directories.
- file's inode number stays the same when it is moved to another directory on the same device
- It is possible for a device to run out of inodes.
- Inlining file data in Inode is possible, [Ext4](https://en.wikipedia.org/wiki/Ext4 "Ext4") has a file system option called `inline_data` that allows ext4 to perform inlining if enabled during file system creation. Because an inode's size is limited, this only works for very small files.

