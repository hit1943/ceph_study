// src/client/client.cc

```
int Client::_open(Inode *in, int flags, mode_t mode, Fh **fhp,
                  const UserPerm &perms) {
  if (in->snapid != CEPH_NOSNAP &&
      (flags & (O_WRONLY | O_RDWR | O_CREAT | O_TRUNC | O_APPEND))) {
    return -EROFS;
  }

  // use normalized flags to generate cmode
  int cflags = ceph_flags_sys2wire(flags);
  if (cct->_conf.get_val<bool>("client_force_lazyio"))
    cflags |= CEPH_O_LAZY;

  int cmode = ceph_flags_to_mode(cflags);
  int want = ceph_caps_for_mode(cmode);
  int result = 0;

  in->get_open_ref(
      cmode); // make note of pending open, since it effects _wanted_ caps.
  ...
}

in up source code, flags, cflags, cmode, want
flags: 调用方传入的通用的flag，linux系统下规定的flag
定义在/use/include/sys/fcntl.h
#define O_RDONLY        0x0000          /* open for reading only */
#define O_WRONLY        0x0001          /* open for writing only */
#define O_RDWR          0x0002          /* open for reading and writing */
#define O_ACCMODE       0x0003          /* mask for above modes */

#define O_CREAT         0x00000200      /* create if nonexistant */
#define O_TRUNC         0x00000400      /* truncate to zero length */
#define O_EXCL          0x00000800      /* error if already exists */

cflags: 通过ceph_flags_sys2wire将flags转化成cephfs自身定义的cflags
#define CEPH_O_RDONLY 00000000
#define CEPH_O_WRONLY 00000001
#define CEPH_O_RDWR 00000002
#define CEPH_O_CREAT 00000100
#define CEPH_O_EXCL 00000200
#define CEPH_O_TRUNC 00001000
#define CEPH_O_LAZY 00020000
#define CEPH_O_DIRECTORY 00200000
#define CEPH_O_NOFOLLOW 00400000

cmode: 通过ceph_flags_to_mode将cflags转化成cephfs中的文件访问模式cmode，可以看到cmode的取值其实是比较少的，只有pin，读，写，读写，lazy几种
/* file access modes */
#define CEPH_FILE_MODE_PIN 0
#define CEPH_FILE_MODE_RD 1
#define CEPH_FILE_MODE_WR 2
#define CEPH_FILE_MODE_RDWR 3 /* RD | WR */
#define CEPH_FILE_MODE_LAZY 4 /* lazy io */
#define CEPH_FILE_MODE_NUM 8  /* bc these are bit fields.. mostly */

open_by_mode是inode所属一个map，get_open_ref这个函数会向map中插入元素，key=cmode, value = 1, 或者对已有key的value++
因此open_by_mode中记录的实际是当前inode被哪些模式打开，并且以这种模式打开的次数有多少次
void Inode::get_open_ref(int mode) {
  open_by_mode[mode]++;
  break_deleg(!(mode & CEPH_FILE_MODE_WR));
}

want: 执行对应的操作需要的caps，通过ceph_caps_for_mode将cmode转化成caps
int ceph_caps_for_mode(int mode) {
  int caps = CEPH_CAP_PIN;

  if (mode & CEPH_FILE_MODE_RD)
    caps |= CEPH_CAP_FILE_SHARED | CEPH_CAP_FILE_RD | CEPH_CAP_FILE_CACHE;
  if (mode & CEPH_FILE_MODE_WR)
    caps |= CEPH_CAP_FILE_EXCL | CEPH_CAP_FILE_WR | CEPH_CAP_FILE_BUFFER |
            CEPH_CAP_AUTH_SHARED | CEPH_CAP_AUTH_EXCL | CEPH_CAP_XATTR_SHARED |
            CEPH_CAP_XATTR_EXCL;
  if (mode & CEPH_FILE_MODE_LAZY)
    caps |= CEPH_CAP_FILE_LAZYIO;

  return caps;
}


