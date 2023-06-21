```
class Objecter : public md_config_obs_t, public Dispatcher {
public:
  // config observer bits
  const char **get_tracked_conf_keys() const override;
  void handle_conf_change(const ConfigProxy &conf,
                          const std::set<std::string> &changed) override;

public:
  Messenger *messenger;
  MonClient *monc;
  Finisher *finisher;
  ZTracer::Endpoint trace_endpoint;

private:
  std::unique_ptr<OSDMap> osdmap;

public:
  template <typename Callback, typename... Args>
  decltype(auto) with_osdmap(Callback &&cb, Args &&...args) {
    shared_lock l(rwlock);
    return std::forward<Callback>(cb)(*osdmap, std::forward<Args>(args)...);
  }
};

int Client::start_reclaim(Objecter *objecter) {
{
  ...
  bool blacklisted =
      objecter->with_osdmap([this](const OSDMap &osd_map) -> bool {
        return osd_map.is_blacklisted(reclaim_target_addrs);
      });
  ...
}      
``` 
