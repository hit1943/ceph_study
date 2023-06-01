// src/ceph_mon.cc
int main(int argc, const char **argv)


```
  void _open(const string &kv_type) {
    string::const_reverse_iterator rit;
    int pos = 0;
    for (rit = path.rbegin(); rit != path.rend(); ++rit, ++pos) {
      if (*rit != '/')
        break;
    }
    ostringstream os;
    os << path.substr(0, path.size() - pos) << "/store.db";
    string full_path = os.str();

    KeyValueDB *db_ptr = KeyValueDB::create(g_ceph_context, kv_type, full_path);
    if (!db_ptr) {
      derr << __func__ << " error initializing " << kv_type
           << " db back storage in " << full_path << dendl;
      ceph_abort_msg(
          "MonitorDBStore: error initializing keyvaluedb back storage");
    }
    db.reset(db_ptr);
``` 

