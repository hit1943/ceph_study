```
[root@n172-123-091 tmp]# ceph-kvstore-tool rocksdb store.db get mdsmap 12192 out mdsmap.12192
2023-07-12T16:20:06.840+0000 7f17ab6a7240  1 rocksdb: do_open column families: [default]
(mdsmap, 12192)
[root@n172-123-091 tmp]# ceph-dencoder import mdsmap.12192 type FSMap decode dump_json
{
    "epoch": 12192,
    "default_fscid": 411,
    "compat": {
        "compat": {},
        "ro_compat": {},
        "incompat": {
            "feature_1": "base v0.20",
            "feature_2": "client writeable ranges",
            "feature_3": "default file layouts on dirs",
            "feature_4": "dir inode in separate object",
            "feature_5": "mds uses versioned encoding",
            "feature_6": "dirfrag is stored in omap",
            "feature_7": "mds uses inline data",
            "feature_8": "no anchor table",
            "feature_9": "file layout v2",
            "feature_10": "snaprealm v2"
        }
    },
    "feature_flags": {
        "enable_multiple": true,
        "ever_enabled_multiple": false
    },
    "standbys": [
        {
            "gid": 88187566,
            "name": "vees-root-cephfs-b",
            "rank": -1,
            "incarnation": 0,
            "state": "up:standby",
            "state_seq": 2,
            "addr": "10.172.123.87:6865/317615280",
            "addrs": {
                "addrvec": [
                    {
                        "type": "v2",
                        "addr": "10.172.123.87:6864",
                        "nonce": 317615280
                    },
                    {
                        "type": "v1",
                        "addr": "10.172.123.87:6865",
                        "nonce": 317615280
                    }
                ]
            },
            "join_fscid": 411,
            "export_targets": [],
            "features": 4540138292836696063,
            "flags": 0,
            "epoch": 12187
        },
        {
            "gid": 88196750,
            "name": "vees-root-cephfs-a",
            "rank": -1,
            "incarnation": 0,
            "state": "up:standby",
            "state_seq": 2,
            "addr": "10.172.123.89:6837/542041067",
            "addrs": {
                "addrvec": [
                    {
                        "type": "v2",
                        "addr": "10.172.123.89:6833",
                        "nonce": 542041067
                    },
                    {
                        "type": "v1",
                        "addr": "10.172.123.89:6837",
                        "nonce": 542041067
                    }
                ]
            },
            "join_fscid": 411,
            "export_targets": [],
            "features": 4540138292836696063,
            "flags": 0,
            "epoch": 12192
        }
    ],
    "filesystems": [
        {
            "mdsmap": {
                "epoch": 12190,
                "flags": 18,
                "ever_allowed_features": 0,
                "explicitly_allowed_features": 0,
                "created": "2023-07-08T08:09:32.448439+0000",
                "modified": "2023-07-10T08:46:17.688155+0000",
                "tableserver": 0,
                "root": 0,
                "session_timeout": 60,
                "session_autoclose": 300,
                "min_compat_client": "0 (unknown)",
                "max_file_size": 1099511627776,
                "last_failure": 0,
                "last_failure_osd_epoch": 24687,
                "compat": {
                    "compat": {},
                    "ro_compat": {},
                    "incompat": {
                        "feature_1": "base v0.20",
                        "feature_2": "client writeable ranges",
                        "feature_3": "default file layouts on dirs",
                        "feature_4": "dir inode in separate object",
                        "feature_5": "mds uses versioned encoding",
                        "feature_6": "dirfrag is stored in omap",
                        "feature_7": "mds uses inline data",
                        "feature_8": "no anchor table",
                        "feature_9": "file layout v2",
                        "feature_10": "snaprealm v2"
                    }
                },
                "max_mds": 1,
                "in": [
                    0
                ],
                "up": {
                    "mds_0": 87332435
                },
                "failed": [],
                "damaged": [],
                "stopped": [],
                "info": {
                    "gid_87332435": {
                        "gid": 87332435,
                        "name": "vees-root-cephfs-c",
                        "rank": 0,
                        "incarnation": 12185,
                        "state": "up:active",
                        "state_seq": 43753,
                        "addr": "10.172.123.91:6865/2943949842",
                        "addrs": {
                            "addrvec": [
                                {
                                    "type": "v2",
                                    "addr": "10.172.123.91:6864",
                                    "nonce": 2943949842
                                },
                                {
                                    "type": "v1",
                                    "addr": "10.172.123.91:6865",
                                    "nonce": 2943949842
                                }
                            ]
                        },
                        "join_fscid": 411,
                        "export_targets": [],
                        "features": 4540138292836696063,
                        "flags": 0
                    }
                },
                "data_pools": [
                    774
                ],
                "metadata_pool": 773,
                "enabled": true,
                "fs_name": "vees-root-cephfs",
                "balancer": "",
                "standby_count_wanted": 1
            },
            "id": 411
        }
    ]
}

root@node11.tjmp02:~# ceph fs dump
dumped fsmap epoch 12192
e12192
enable_multiple, ever_enabled_multiple: 1,0
compat: compat={},rocompat={},incompat={1=base v0.20,2=client writeable ranges,3=default file layouts on dirs,4=dir inode in separate object,5=mds uses versioned encoding,6=dirfrag is stored in omap,7=mds uses inline data,8=no anchor table,9=file layout v2,10=snaprealm v2}
legacy client fscid: 411

Filesystem 'vees-root-cephfs' (411)
fs_name	vees-root-cephfs
epoch	12190
flags	12
created	2023-07-08T08:09:32.448439+0000
modified	2023-07-10T08:46:17.688155+0000
tableserver	0
root	0
session_timeout	60
session_autoclose	300
max_file_size	1099511627776
min_compat_client	0 (unknown)
last_failure	0
last_failure_osd_epoch	24687
compat	compat={},rocompat={},incompat={1=base v0.20,2=client writeable ranges,3=default file layouts on dirs,4=dir inode in separate object,5=mds uses versioned encoding,6=dirfrag is stored in omap,7=mds uses inline data,8=no anchor table,9=file layout v2,10=snaprealm v2}
max_mds	1
in	0
up	{0=87332435}
failed	
damaged	
stopped	
data_pools	[774]
metadata_pool	773
inline_data	disabled
balancer	
standby_count_wanted	1
[mds.vees-root-cephfs-c{0:87332435} state up:active seq 43753 join_fscid=411 addr [v2:10.172.123.91:6864/2943949842,v1:10.172.123.91:6865/2943949842]]


Standby daemons:

[mds.vees-root-cephfs-b{-1:88187566} state up:standby seq 2 join_fscid=411 addr [v2:10.172.123.87:6864/317615280,v1:10.172.123.87:6865/317615280]]
[mds.vees-root-cephfs-a{-1:88196750} state up:standby seq 2 join_fscid=411 addr [v2:10.172.123.89:6833/542041067,v1:10.172.123.89:6837/542041067]]
```
