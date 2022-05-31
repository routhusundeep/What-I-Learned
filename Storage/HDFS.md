#distributed_file_system #real_world_system 

I won't summarize it since it is mostly similar to [[Google File System]]

[Here](https://hadoop.apache.org/docs/r1.2.1/hdfs_design.html) is the design.
Read and Write flows in HDFS are given [here](https://knpcode.com/hadoop/hdfs/hdfs-data-flow-file-read-write-hdfs/).

Recovery processes are explained [here](https://blog.cloudera.com/understanding-hdfs-recovery-processes-part-1/) and [here](https://blog.cloudera.com/understanding-hdfs-recovery-processes-part-2/). These are very insightful blog posts and explains some significant deviation from GFS with how leases are handled by HDFS.


