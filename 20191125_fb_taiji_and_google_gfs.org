#+TITLE: Daily Notes Monday, 25/11/2019
** [[https://blog.acolyer.org/2019/11/15/facebook-taiji/][link]] - Taiji - managing global user traffic               :load_balancing:
Taiji is a connection(user) aware load balancing system, currently in use in facebook.
It divided the users into buckets and allocates these buckets into datacenters. Social hash is the algorithm which determines what users the bucket consists of.
A lot more details are present in the paper.
It highlights the use of LB at L7.
** [[https://static.googleusercontent.com/media/research.google.com/en/us/archive/gfs-sosp2003.pdf][link]] - Google File System                        :distributed_file_system:
Once again read about GFS, it defines the core principles and motivations of a distributed file system
