

```

ceph mgr services

curl -X POST http://mgr_ip:mgr_port/ceph-dashboard/api/auth -H "Content-Type: application/json" -d '{"username": "username", "password": "password123456"}'

curl -v http://mgr_ip:mgr_port/ceph-dashboard/api/settings -H "Authorization: Bearer token"|jp -q

curl -v http://mgr_ip:mgr_port/ceph-dashboard/ui-api/standard_settings -H "Authorization: Bearer token"|jq
```


# auth process
command: 
    ceph dashboard ac-user-create username password123456 administrator
    
source code:
```
@CLIWriteCommand('dashboard ac-user-create',
                 'name=username,type=CephString '
                 'name=password,type=CephString,req=false '
                 'name=rolename,type=CephString,req=false '
                 'name=name,type=CephString,req=false '
                 'name=email,type=CephString,req=false '
                 'name=enabled,type=CephBool,req=false '
                 'name=force_password,type=CephBool,req=false '
                 'name=pwd_expiration_date,type=CephInt,req=false '
                 'name=pwd_update_required,type=CephBool,req=false',
                 'Create a user')
def ac_user_create_cmd(_, username, password=None, rolename=None, name=None,
                       email=None, enabled=True, force_password=False,
                       pwd_expiration_date=None, pwd_update_required=False):
    try:
        role = mgr.ACCESS_CTRL_DB.get_role(rolename) if rolename else None
    except RoleDoesNotExist as ex:
        if rolename not in SYSTEM_ROLES:
            return -errno.ENOENT, '', str(ex)
        role = SYSTEM_ROLES[rolename]

    try:
        if not force_password:
            pw_check = PasswordPolicy(password, username)
            pw_check.check_all()
        user = mgr.ACCESS_CTRL_DB.create_user(username, password, name, email,
                                              enabled, pwd_expiration_date,
                                              pwd_update_required)
    except PasswordPolicyException as ex:
        return -errno.EINVAL, '', str(ex)
    except UserAlreadyExists as ex:
        return 0, str(ex), ''

    if role:
        user.set_roles([role])
    mgr.ACCESS_CTRL_DB.save()
    return 0, json.dumps(user.to_dict()), ''
    
    def create_user(self, username, password, name, email, enabled=True,
                    pwd_expiration_date=None, pwd_update_required=False):
        logger.debug("creating user: username=%s", username)
        with self.lock:
            if username in self.users:
                raise UserAlreadyExists(username)
            if pwd_expiration_date and \
               (pwd_expiration_date < int(time.mktime(datetime.utcnow().timetuple()))):
                raise PwdExpirationDateNotValid()
            user = User(username, password_hash(password), name, email, enabled=enabled,
                        pwd_expiration_date=pwd_expiration_date,
                        pwd_update_required=pwd_update_required)
            self.users[username] = user
            return user
            
    def save(self):
        with self.lock:
            db = {
                'users': {un: u.to_dict() for un, u in self.users.items()},
                'roles': {rn: r.to_dict() for rn, r in self.roles.items()},
                'version': self.version
            }
            # key = accessdb_v2
            mgr.set_store(self.accessdb_config_key(), json.dumps(db))
            
  src/pybind/mgr/mgr_module.py
      def set_store(self, key, val):
        """
        Set a value in this module's persistent key value store.
        If val is None, remove key from store

        :param str key:
        :param str val:
        """
        self._ceph_set_store(key, val)
    
    src/mgr/BaseMgrModule.cc
    PyMethodDef BaseMgrModule_methods[] = {
    
    {"_ceph_set_store", (PyCFunction)ceph_store_set, METH_VARARGS, "Set a stored field"},
    }
    
static PyObject *ceph_store_set(BaseMgrModule *self, PyObject *args) {
  char *key = nullptr;
  char *value = nullptr;
  if (!PyArg_ParseTuple(args, "sz:ceph_store_set", &key, &value)) {
    return nullptr;
  }
  boost::optional<string> val;
  if (value) {
    val = value;
  }
  PyThreadState *tstate = PyEval_SaveThread();
  self->py_modules->set_store(self->this_module->get_name(), key, val);
  PyEval_RestoreThread(tstate);

  Py_RETURN_NONE;
}

src/mgr/ActivePyModules.cc
void ActivePyModules::set_store(const std::string &module_name,
                                const std::string &key,
                                const boost::optional<std::string> &val) {
  // config_prefix: "mgr/"
  // module_name: "dashboard"
  // key: accessdb_v2
  const std::string global_key =
      PyModule::config_prefix + module_name + "/" + key;

  Command set_cmd;
  {
    std::lock_guard l(lock);
    if (val) {
      store_cache[global_key] = *val;
    } else {
      store_cache.erase(global_key);
    }

    std::ostringstream cmd_json;
    JSONFormatter jf;
    jf.open_object_section("cmd");
    if (val) {
      jf.dump_string("prefix", "config-key set");
      jf.dump_string("key", global_key);
      jf.dump_string("val", *val);
    } else {
      jf.dump_string("prefix", "config-key del");
      jf.dump_string("key", global_key);
    }
    jf.close_section();
    jf.flush(cmd_json);
    set_cmd.run(&monc, cmd_json.str());
  }
  set_cmd.wait();

  if (set_cmd.r != 0) {
    // config-key set will fail if mgr's auth key has insufficient
    // permission to set config keys
    // FIXME: should this somehow raise an exception back into Python land?
    dout(0) << "`config-key set " << global_key << " " << val
            << "` failed: " << cpp_strerror(set_cmd.r) << dendl;
    dout(0) << "mon returned " << set_cmd.r << ": " << set_cmd.outs << dendl;
  }
}

// src/mgr/MgrContext.h
class Command {
protected:
  C_SaferCond cond;

public:
  ceph::buffer::list outbl;
  std::string outs;
  int r;

  void run(MonClient *monc, const std::string &command) {
    monc->start_mon_command({command}, {}, &outbl, &outs, &cond);
  }
}
//此处向mon发送请求没有指定mon，则
void MonClient::start_mon_command(const std::vector<string> &cmd,
                                  const ceph::buffer::list &inbl,
                                  ceph::buffer::list *outbl, string *outs,
                                  Context *onfinish) {
  ldout(cct, 10) << __func__ << " cmd=" << cmd << dendl;
  std::lock_guard l(monc_lock);
  if (!initialized || stopping) {
    if (onfinish) {
      onfinish->complete(-ECANCELED);
    }
    return;
  }
  MonCommand *r = new MonCommand(++last_mon_command_tid);
  r->cmd = cmd;
  r->inbl = inbl;
  r->poutbl = outbl;
  r->prs = outs;
  r->onfinish = onfinish;
  if (cct->_conf->rados_mon_op_timeout > 0) {
    class C_CancelMonCommand : public Context {
      uint64_t tid;
      MonClient *monc;

    public:
      C_CancelMonCommand(uint64_t tid, MonClient *monc)
          : tid(tid), monc(monc) {}
      void finish(int r) override { monc->_cancel_mon_command(tid); }
    };
    r->ontimeout = new C_CancelMonCommand(r->tid, this);
    timer.add_event_after(cct->_conf->rados_mon_op_timeout, r->ontimeout);
  }
  mon_commands[r->tid] = r;
  _send_command(r);
}
