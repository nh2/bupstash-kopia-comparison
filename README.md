# Comparing `bupstash` and `kopia`

Here I collect some benchmarks and of the data backup/deduplicaton tools `bupstash` and `kopia`.

The goal is to share my own evaluation notes with others.

Methodology:

* When the inputs fit into RAM, each command is run twice (with repo re-creation in between) to ensure hot input file system caches.
  When they don't fit into RAM, we don'tMethodology:
 bother running twice.
* For readers unfamiliar iwth `du` output:
  * `apparent size` is the number of actual bytes in a file, e.g. when `cat`ed
  * `disk usage` is the amount of disk space required to store the file physically; it can be:
    * less than `apparent size` if e.g. file system compression is enabled
    * more than `apparent size` if the files are small so metadata overhead or minimum allocations carry weight
    Thus, for checking how well a deduplication program deduplicates, `apparent size` of the inputs should be considered.


## 4 GB, small files

Environment:

    Computer: Thinkpad T25
    SSD:      Kingston Data Center DC1000B (SEDC1000BM8480G)
    CPU:      Intel Core i7-7500U
    RAM:      48 GB
    OS:       NixOS 20.09
    FS:       ZFS with encryption and lz4 compression, no dedup
    bupstash: 0.9.0
    kopia:    0.8.4

Settings:

    kopia policy set --global --ignore-cache-dirs false

Inputs:

    disk usage (GiB)   apparent size (GiB)   files   dir
                 4.1                   4.7  437.1k   /haskell

* Various Haskell source code repositories and built object files, including a dump of all all Hackage package versions metadata.

bupstash initial:

    command time bupstash put ~/src/haskell
    8b9df9479337ad9775d14bef640efa62
    48.47user 66.59system 2:47.03elapsed 68%CPU (0avgtext+0avgdata 85256maxresident)k
    15703703inputs+11635339outputs (124major+109578minor)pagefaults 0swaps

bupstash nochange:

    $ command time bupstash put ~/src/haskell
    c0424baf8b409e479cd571bb3c3d618b
    6.42user 6.01system 0:13.22elapsed 94%CPU (0avgtext+0avgdata 58360maxresident)k
    2560168inputs+1955096outputs (1major+18025minor)pagefaults 0swaps

kopia initial:

    $ command time kopia snapshot create ~/src/haskell          
    Snapshotting niklas@t25:/home/niklas/src/haskell ...
     * 0 hashing, 280772 hashed (5 GB), 0 cached (0 B), uploaded 4.9 GB, estimating...                          
    Created snapshot with root k561a0372f62d8ccab7738af642a43694 and ID 44d574172490ee1399cca644183d1eaf in 1m20s
    Running full maintenance...
    Looking for active contents...
      ...
    Looking for unreferenced contents...
    Not enough time has passed since previous successful Snapshot GC. Will try again next time.
    Rewriting contents from short packs...
    Skipping blob deletion because not enough time has passed yet (59m59s left).
    Finished full maintenance.
    88.12user 24.88system 1:31.42elapsed 123%CPU (0avgtext+0avgdata 583708maxresident)k
    10237521inputs+10084132outputs (3868major+136243minor)pagefaults 0swaps

kopia nochange:

    $ command time kopia snapshot create ~/src/haskell
    Snapshotting niklas@t25:/home/niklas/src/haskell ...
     * 0 hashing, 0 hashed (0 B), 280772 cached (5 GB), uploaded 4.6 KB, estimating...                   
    Created snapshot with root k561a0372f62d8ccab7738af642a43694 and ID 229f8b99fbd817b8d25e9eadad6d2be8 in 30s
    30.12user 11.26system 0:30.96elapsed 133%CPU (0avgtext+0avgdata 113140maxresident)k
    219725inputs+65144outputs (8major+14789minor)pagefaults 0swaps

Performance summary:

    tool      time initial (s)   time nochange (s)   mem initial (MB)   mem nochange (MB)
    bupstash               167                  13                 85                  58
    kopia                   91                  31                583                 113

Repo sizes:

    tool      disk usage (GiB)   apparent size (GiB)    files
    bupstash               3.7                   3.5   141809
    kopia                  4.5                   4.5      582

Notes:

* Kopia's deduplication was ineffective for the initial snapshot on these inputs.
* bupstash does not manage to saturate a single CPU, which is why kopia is faster for initial snapshotting.
* bupstash uses 7x less max memory.
* kopia uses 300x less files in the repo.


## 300 GB, 24 files

Environment:

    Computer: Hetzner SX133
    HDD:      Seagate Exos X16 (ST16000NM003G-2KH113), 10 disks in CephFS local RAID-1
    CPU:      Intel Xeon W-2145
    RAM:      128 GB
    OS:       NixOS 20.03
    FS:       CephFS
    bupstash: 0.9.0
    kopia:    0.8.4

Settings:

    kopia policy set --global --ignore-cache-dirs false

Inputs:

    disk usage (GiB)   apparent size (GiB)   files   dir
                 399                   399       3   /shared/corp/root/backup/ceph-logs

* Ceph log files: Plain text with many lines being very similar but varying slightly (e.g. time as first word).

bupstash initial:

    # command time bupstash put --key keys/bupstash.key --send-log /tmp/bupstash-tmpfs/bupstash-bench.sendlog --print-stats /shared/corp/root/backup/ceph-logs
    37m44.528s put duration
    129167 chunk(s), 428.66GB processed
    129115 chunk(s), 52.49GB (compressed) sent
    129115 chunk(s), 52.49GB (compressed) added
    8.2:1 effective compression ratio
    ccd12e6c08423c60d41a5f743cbc7e9b
    1077.74user 252.39system 37:44.97elapsed 58%CPU (0avgtext+0avgdata 54848maxresident)k
    3736inputs+103109080outputs (20major+3303303minor)pagefaults 0swaps

bupstash nochange:

    # command time bupstash put --key keys/bupstash.key --send-log /tmp/bupstash-tmpfs/bupstash-bench.sendlog --print-stats /shared/corp/root/backup/ceph-logs
    0m0.79s put duration
    129167 chunk(s), 428.66GB processed
    0 chunk(s), 0B (compressed) sent
    0 chunk(s), 0B (compressed) added
    infinite effective compression ratio
    d115b8bf437906cf6f5c3221b5eb88b4
    0.11user 0.03system 0:00.69elapsed 20%CPU (0avgtext+0avgdata 55580maxresident)k
    1704inputs+24outputs (9major+11623minor)pagefaults 0swaps

kopia initial:

    # command time kopia snapshot create /shared/corp/root/backup/ceph-log
    Snapshotting root@corpfs-1:/shared/corp/root/backup/ceph-logs ...
     * 0 hashing, 24 hashed (428.7 GB), 0 cached (0 B), uploaded 428.7 GB, estimating...                         
    Created snapshot with root k5601e14d360d09059a7426ed671896ef and ID 1e7487f654eb1252db4032a6be65ab6a in 35m10s
    3141.73user 303.20system 35:10.89elapsed 163%CPU (0avgtext+0avgdata 409376maxresident)k
    60272inputs+837625056outputs (472major+72100minor)pagefaults 0swaps

kopia nochange:

    # command time kopia snapshot create /shared/corp/root/backup/ceph-logs
    Snapshotting root@corpfs-1:/shared/corp/root/backup/ceph-logs ...
     * 0 hashing, 0 hashed (0 B), 24 cached (428.7 GB), uploaded 4.6 KB, estimating...
    Created snapshot with root k5601e14d360d09059a7426ed671896ef and ID 42f601eeafe6e14e4647d3e8ffac846b in 0s
    0.21user 0.53system 0:00.84elapsed 90%CPU (0avgtext+0avgdata 135552maxresident)k
    23472inputs+12960outputs (111major+26294minor)pagefaults 0swaps

Performance summary:

    tool      time initial (s)   time nochange (s)   mem initial (MB)   mem nochange (MB)
    bupstash              2265                   1                 55                  56
    kopia                 2111                   1                409                 136

Repo sizes:

    tool      disk usage (GiB)   apparent size (GiB)    files
    bupstash                49                    49   129126
    kopia                  399                   399    37345

Notes:

* Kopia's deduplication was ineffective for the initial snapshot on these inputs.
* Time is equal for these large files, but bupstash needs 3x less CPU.
* bupstash uses 7x less max memory.
* Backup was from CephFS to same CephFS.



## Storage server (12 TB, 30M files)

Environment:

    Computer: Hetzner SX133
    HDD:      Seagate Exos X16 (ST16000NM003G-2KH113), 10 disks in CephFS local RAID-1
    CPU:      Intel Xeon W-2145
    RAM:      128 GB
    OS:       NixOS 20.03
    FS:       CephFS
    bupstash: 0.9.0
    kopia:    0.8.4

Settings:

    kopia policy set --global --ignore-cache-dirs false

Inputs:

    disk usage (GiB)   apparent size (GiB)      files   dir
               12600                 12600   33462500   /shared/corp/root

Estimation time:

```
# command time kopia snapshot estimate /shared/corp/root
...
Snapshot includes 27626340 files, total size 13.7 TB
   100 GB...  1 TB:       3 files, total size 457.4 GB
   10 GB ...100 GB:      96 files, total size 2.1 TB
   1 GB  ... 10 GB:     996 files, total size 2.7 TB
   100 MB...  1 GB:   11293 files, total size 3.7 TB
   10 MB ...100 MB:   80728 files, total size 3.2 TB
   1 MB  ... 10 MB:  317028 files, total size 1.3 TB
   100 KB...  1 MB:  278942 files, total size 134.6 GB
   10 KB ...100 KB:  100035 files, total size 3.9 GB
   1 KB  ... 10 KB:  502857 files, total size 1.4 GB
   0 B   ...  1 KB: 26334362 files, total size 3.3 GB

Snapshots excludes no files.
Snapshots excludes 1 directories. Examples:
 - ./cache

Estimated upload time: 3049h42m7s at 10 Mbit/s

512.02user 4632.95system 54:45.60elapsed 156%CPU (0avgtext+0avgdata 2974040maxresident)k
6920inputs+80576outputs (72major+1122850minor)pagefaults 0swaps
```

`find` performance to compare with estimation time and file count (traversal only, no `stat`ing):

```
# command time find /shared/corp/root -not -path "./cache/*" | wc -l
40929716

65.41user 49.51system 1:05:09elapsed 2%CPU (0avgtext+0avgdata 38756maxresident)k
0inputs+0outputs (0major+77031minor)pagefaults 0swaps
```

kopia initial:

```
  ...
  Processed 23756387 contents, discovered 23756411...
Looking for unreferenced contents...
Found safe time to drop indexes: 2021-06-13 11:46:51.271252705 +0000 UTC
Dropping contents deleted before 2021-06-13 11:46:51.271252705 +0000 UTC
Previous content rewrite has not been finalized yet, waiting until the next blob deletion.
Looking for unreferenced blobs...
Deleted total 0 unreferenced blobs (0 B)
Finished full maintenance.

146415.98user 22482.12system 28:14:47elapsed 166%CPU (0avgtext+0avgdata 38142740maxresident)k
1114128inputs+19506871832outputs (8717major+38119805minor)pagefaults 0swaps
```

kopia incremental:

```
# command time kopia snapshot create /shared/corp/root
Snapshotting root@corpfs-1:/shared/corp/root ...
 * 0 hashing, 86882 hashed (0.9 TB), 33121302 cached (13 TB), uploaded 42.3 GB, estimated 13.9 TB (100.0%) 0s left
Created snapshot with root kecce7399c154a79e2d0cc5bcbc47db41 and ID dc41df9ce8d393de55f7da2ed5b42bf2 in 1h35m15s
Running quick maintenance...
Rewriting contents from short packs...
Skipping blob deletion because not enough time has passed yet (59m59s left).
Compacting indexes...
Cleaned up 107 logs.
Finished quick maintenance.

8926.66user 9108.96system 1:36:21elapsed 311%CPU (0avgtext+0avgdata 17959532maxresident)k
23910712inputs+98948016outputs (117310major+5989812minor)pagefaults 0swaps
```

bupstash initial:

```
# command time bupstash put --key keys/bupstash-corpfs-to-dropbox.key --send-log /tmp/bupstash-tmpfs/bupstash-corpfs-to-dropbox.sendlog --one-file-system --exclude /shared/corp/root/cache --print-stats name=corpfs-backup.tar /shared/corp/root
[08:56:31] /shared/corp/root/temp/bupstash-bench-repo/data [2.75TB sent, 95.37MB/[09216m1.635s put duration
11857271 chunk(s), 13.90TB processed
8076658 chunk(s), 8.58TB (compressed) sent
8076658 chunk(s), 8.58TB (compressed) added
1.6:1 effective compression ratio

52091.70user 27225.64system 153:36:08elapsed 14%CPU (0avgtext+0avgdata 6384344maxresident)k
21296inputs+16796551784outputs (122major+52918263minor)pagefaults 0swaps
```

bupstash incremental:

```
# ./run-corpfs-to-dropbox-bupstash-backup-sendlog-on-tmpfs.sh
943m20.109s put duration
12098946 chunk(s), 14.19TB processed
41167 chunk(s), 43.89GB (compressed) sent
21043 chunk(s), 22.34GB (compressed) added
635.2:1 effective compression ratio

785.59user 408.82system 15:43:21elapsed 2%CPU (0avgtext+0avgdata 6280728maxresident)k
12864inputs+43727112outputs (83major+4003460minor)pagefaults 0swaps
```

bupstash somewhat-less-than-initial (and on a smaller part of the file system, but with > 90% of the small files) that I did 35 days earlier:

```
68.13675m44.227s put duration
10181153 chunk(s), 11.70TB processed
1050705 chunk(s), 1.03TB (compressed) sent
1049775 chunk(s), 1.03TB (compressed) added
11.3:1 effective compression ratio

8032.15user 4674.08system 61:15:50elapsed 5%CPU (0avgtext+0avgdata 5611020maxresident)k
78216inputs+2030658840outputs (358major+28604226minor)pagefaults 0swaps
```

Performance summary:

    tool      time initial (h)   time nochange (h)   mem initial (MB)   mem nochange (MB)
    bupstash               154                15.5               6384                6280
    kopia                   28                 1.5              38142               17959


Repo sizes:

    tool      disk usage (GiB)   apparent size (GiB)     files
    bupstash              7800                  7800   8076668
    bupstash              6600                  6600   6679776   (from 35 days ago)
    kopia                 9000                  9000    787436


## Other notes

* bupstash is 5x slower for the initial backup of this data set, likely mainly because it does not stat small files in parallel.
* It takes a long time to `ncdu` the bupstash repository, because it contains so many small files.
* With `kopia mount`, `pv < ... > /dev/null` on a large file in the backup runs at stable 50 MB/s (repo backed by local CephFS).
* kopia creates partial snapshots during the backup, and also when you Ctrl+C it.
  Those partial backups can be restored/mounted like normal backups.
* Directory listing on `kopia mount` is slow, e.g. when using `find`: `getdents64()` syscall on a large directory takes 20 seconds per 18 entries.
  `strace -T` output is `getdents64(/kopiamount/.../largedir, /* 18 entries */, 32768) = 1584 <20.273433>`
  On the original local CephFS, `find` runs on `255010` dirents in 8 seconds -- 35000x faster.
  Filed as https://github.com/kopia/kopia/issues/1135.
  * Results of newer commit from https://github.com/kopia/kopia/issues/1135#issuecomment-860255925:
    * Did not help.
* `kopia snapshot estimate` takes a lot of memory if there are many files (3 GB).
* For small files, kopia snapshots only 300 per second on CephFS.
* kopia'r `Readdir()` [stats with 16 threads](https://github.com/kopia/kopia/blob/8b0296cdf2c37d4f0f3de408eb653b80b56a23a5/fs/localfs/local_fs.go#L170-L171) for estimation and snapshotting.
  * This makes it slow on CephFS and not use a full core when small files are involved.
  * Should be easy to increase (but hardcoded).
  * Snapshotting small files does only 300 per second; for 30M this would take 8.3 hours.
    However, `estimate` ran through in 55 minutes.
    This suggests that statting is not the bottleneck for snapshotting.
    Maybe `stat()` is > 8x faster than `open()`, thus making estimation faster.
    Oddly, `estimate` is faster than `find` (55 minutes vs 65 minutes).
* The `Readdirnames()` calls currently do [at most 100](https://github.com/kopia/kopia/blob/8b0296cdf2c37d4f0f3de408eb653b80b56a23a5/fs/localfs/local_fs.go#L18) dirents in one batch -- but unclear if raising that would make the Go runtime pass bigger buffers to `getdents()`.
* kopia used 5.5 GB RES memory after being 90% through the data Bytes and 30 M of the files, and now working on many small files.
  Overall `time` maxresident was 38 GB.
* `kopia snapshot create` defaults to `--parallel 0` which is the [number of CPUs](https://github.com/kopia/kopia/blob/8b0296cdf2c37d4f0f3de408eb653b80b56a23a5/snapshot/snapshotfs/upload.go#L798-L799).
  That's 16 in my case; I should try with a higher value.
* kopia currently [does not traverse subdirectories in parallel](https://github.com/kopia/kopia/blob/8b0296cdf2c37d4f0f3de408eb653b80b56a23a5/snapshot/snapshotfs/upload.go#L691-L693) for snapshotting
  It does process a single directory's files [in parallel](https://github.com/kopia/kopia/blob/8b0296cdf2c37d4f0f3de408eb653b80b56a23a5/snapshot/snapshotfs/upload.go#L805), but [cannot do so across directory boundaries](https://github.com/kopia/kopia/blob/8b0296cdf2c37d4f0f3de408eb653b80b56a23a5/snapshot/snapshotfs/upload.go#L671-L677).
