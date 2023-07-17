/*
ContextType必须是ContextInstanceType的基类，或者至少相同，ContextInstanceType必须是默认可构造的
*/
template <class ContextType, class ContextInstanceType> 
class C_GatherBase {
private:
  CephContext *cct;
  int result = 0;
  ContextType *onfinish;
#ifdef DEBUG_GATHER
  std::set<ContextType *> waitfor;
#endif
  int sub_created_count = 0;
  int sub_existing_count = 0;
  mutable ceph::recursive_mutex lock =
      ceph::make_recursive_mutex("C_GatherBase::lock"); // disable lockdep
  bool activated = false;

  void sub_finish(ContextType *sub, int r) {
    lock.lock();
#ifdef DEBUG_GATHER
    ceph_assert(waitfor.count(sub));
    waitfor.erase(sub);
#endif
    --sub_existing_count;
    mydout(cct, 10) << "C_GatherBase " << this << ".sub_finish(r=" << r << ") "
                    << sub
#ifdef DEBUG_GATHER
                    << " (remaining " << waitfor << ")"
#endif
                    << dendl;
    if (r < 0 && result == 0)
      result = r;
    if ((activated == false) || (sub_existing_count != 0)) {
      lock.unlock();
      return;
    }
    lock.unlock();
    delete_me();
  }

  void delete_me() {
    if (onfinish) {
      onfinish->complete(result);
      onfinish = 0;
    }
    delete this;
  }

  class C_GatherSub : public ContextInstanceType {
    C_GatherBase *gather;

  public:
    C_GatherSub(C_GatherBase *g) : gather(g) {}
    void complete(int r) override {
      // Cancel any customized complete() functionality
      // from the Context subclass we're templated for,
      // we only want to hit that in onfinish, not at each
      // sub finish.  e.g. MDSInternalContext.
      Context::complete(r);
    }
    void finish(int r) override {
      gather->sub_finish(this, r);
      gather = 0;
    }
    ~C_GatherSub() override {
      if (gather)
        gather->sub_finish(this, 0);
    }
  };

public:
  C_GatherBase(CephContext *cct_, ContextType *onfinish_)
      : cct(cct_), onfinish(onfinish_) {
    mydout(cct, 10) << "C_GatherBase " << this << ".new" << dendl;
  }
  ~C_GatherBase() {
    mydout(cct, 10) << "C_GatherBase " << this << ".delete" << dendl;
  }
  void set_finisher(ContextType *onfinish_) {
    std::lock_guard l{lock};
    ceph_assert(!onfinish);
    onfinish = onfinish_;
  }
  void activate() {
    lock.lock();
    ceph_assert(activated == false);
    activated = true;
    if (sub_existing_count != 0) {
      lock.unlock();
      return;
    }
    lock.unlock();
    delete_me();
  }
  ContextType *new_sub() {
    std::lock_guard l{lock};
    ceph_assert(activated == false);
    sub_created_count++;
    sub_existing_count++;
    ContextType *s = new C_GatherSub(this);
#ifdef DEBUG_GATHER
    waitfor.insert(s);
#endif
    mydout(cct, 10) << "C_GatherBase " << this << ".new_sub is "
                    << sub_created_count << " " << s << dendl;
    return s;
  }

  inline int get_sub_existing_count() const {
    std::lock_guard l{lock};
    return sub_existing_count;
  }

  inline int get_sub_created_count() const {
    std::lock_guard l{lock};
    return sub_created_count;
  }
};

template <class ContextType, class GatherType> class C_GatherBuilderBase {
public:
  C_GatherBuilderBase(CephContext *cct_)
      : cct(cct_), c_gather(NULL), finisher(NULL), activated(false) {}
  C_GatherBuilderBase(CephContext *cct_, ContextType *finisher_)
      : cct(cct_), c_gather(NULL), finisher(finisher_), activated(false) {}
  ~C_GatherBuilderBase() {
    if (c_gather) {
      ceph_assert(activated); // Don't forget to activate your C_Gather!
    } else {
      delete finisher;
    }
  }
  ContextType *new_sub() {
    if (!c_gather) {
      c_gather = new GatherType(cct, finisher);
    }
    return c_gather->new_sub();
  }
  void activate() {
    if (!c_gather)
      return;
    ceph_assert(finisher != NULL);
    activated = true;
    c_gather->activate();
  }
  void set_finisher(ContextType *finisher_) {
    finisher = finisher_;
    if (c_gather)
      c_gather->set_finisher(finisher);
  }
  GatherType *get() const { return c_gather; }
  bool has_subs() const { return (c_gather != NULL); }
  int num_subs_created() {
    ceph_assert(!activated);
    if (c_gather == NULL)
      return 0;
    return c_gather->get_sub_created_count();
  }
  int num_subs_remaining() {
    ceph_assert(!activated);
    if (c_gather == NULL)
      return 0;
    return c_gather->get_sub_existing_count();
  }

private:
  CephContext *cct;
  GatherType *c_gather;
  ContextType *finisher;
  bool activated;
};

typedef C_GatherBase<Context, Context> C_Gather;
typedef √<Context, C_Gather> C_GatherBuilder;

