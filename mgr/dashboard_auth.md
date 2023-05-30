

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

# mon侧的处理
// src/mon/Monitor.cc
void Monitor::handle_command(MonOpRequestRef op) {
...
  if (module == "config-key") {
    config_key_service->dispatch(op);
    return;
  }
...
}

// src/mon/ConfigKeyService.cc
bool ConfigKeyService::service_dispatch(MonOpRequestRef op) {
  Message *m = op->get_req();
  ceph_assert(m != NULL);
  dout(10) << __func__ << " " << *m << dendl;

  if (!in_quorum()) {
    dout(1) << __func__ << " not in quorum -- waiting" << dendl;
    paxos->wait_for_readable(op, new Monitor::C_RetryMessage(mon, op));
    return false;
  }

  ceph_assert(m->get_type() == MSG_MON_COMMAND);

  MMonCommand *cmd = static_cast<MMonCommand *>(m);

  ceph_assert(!cmd->cmd.empty());

  int ret = 0;
  stringstream ss;
  bufferlist rdata;

  string prefix;
  cmdmap_t cmdmap;

  if (!TOPNSPC::common::cmdmap_from_json(cmd->cmd, &cmdmap, ss)) {
    return false;
  }

  cmd_getval(cmdmap, "prefix", prefix);
  string key;
  cmd_getval(cmdmap, "key", key);

  if (prefix == "config-key get") {
    ret = store_get(key, rdata);
    if (ret < 0) {
      ceph_assert(!rdata.length());
      ss << "error obtaining '" << key << "': " << cpp_strerror(ret);
      goto out;
    }
    ss << "obtained '" << key << "'";

  } else if (prefix == "config-key put" || prefix == "config-key set") {
    if (!mon->is_leader()) {
      mon->forward_request_leader(op);
      // we forward the message; so return now.
      return true;
    }

    bufferlist data;
    string val;
    if (cmd_getval(cmdmap, "val", val)) {
      // they specified a value in the command instead of a file
      data.append(val);
    } else if (cmd->get_data_len() > 0) {
      // they specified '-i <file>'
      data = cmd->get_data();
    }
    if (data.length() > (size_t)g_conf()->mon_config_key_max_entry_size) {
      ret = -EFBIG; // File too large
      ss << "error: entry size limited to "
         << g_conf()->mon_config_key_max_entry_size << " bytes. "
         << "Use 'mon config key max entry size' to manually adjust";
      goto out;
    }

    ss << "set " << key;

    // we'll reply to the message once the proposal has been handled
    store_put(key, data, new Monitor::C_Command(mon, op, 0, ss.str(), 0));
    // return for now; we'll put the message once it's done.
    return true;

  } else if (prefix == "config-key del" || prefix == "config-key rm") {
    if (!mon->is_leader()) {
      mon->forward_request_leader(op);
      return true;
    }

    if (!store_exists(key)) {
      ret = 0;
      ss << "no such key '" << key << "'";
      goto out;
    }
    store_delete(key, new Monitor::C_Command(mon, op, 0, "key deleted", 0));
    // return for now; we'll put the message once it's done
    return true;

  } else if (prefix == "config-key exists") {
    bool exists = store_exists(key);
    ss << "key '" << key << "'";
    if (exists) {
      ss << " exists";
      ret = 0;
    } else {
      ss << " doesn't exist";
      ret = -ENOENT;
    }

  } else if (prefix == "config-key list" || prefix == "config-key ls") {
    stringstream tmp_ss;
    store_list(tmp_ss);
    rdata.append(tmp_ss);
    ret = 0;

  } else if (prefix == "config-key dump") {
    string prefix;
    cmd_getval(cmdmap, "key", prefix);
    stringstream tmp_ss;
    store_dump(tmp_ss, prefix);
    rdata.append(tmp_ss);
    ret = 0;
  }

out:
  if (!cmd->get_source().is_mon()) {
    string rs = ss.str();
    mon->reply_command(op, ret, rs, rdata, 0);
  }

  return (ret == 0);
}

根据get set list等命令字相应调用store_get，store_put，store_list等操作

int ConfigKeyService::store_get(const string &key, bufferlist &bl) {
  return mon->store->get(CONFIG_PREFIX, key, bl);
}

src/mon/MonitorDBStore.h
  int get(const string &prefix, const string &key, bufferlist &bl) {
    ceph_assert(bl.length() == 0);
    return db->get(prefix, key, &bl);
  }
  
# ceph中config-key操作对应的命令行
1. 以上将账户信息保存到kvstore中的流程本质就是向mon主发送一个命令，类似于下列config-key的命令行
ceph --mon-host 10.172.88.88:6789 config-key ls
ceph --mon-host 10.172.88.88:6789 config-key get mgr/dashboard/accessdb_v2|jq

