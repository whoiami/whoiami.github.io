---
layout: post
title: CEPH PG Peering
---

#### PG peering

Here is the definition from [http://docs.ceph.com](http://docs.ceph.com/docs/master/dev/peering/)

The process of bringing all of the OSDs that store a Placement Group (PG) into agreement about the state of all of the objects (and their metadata) in that PG. Note that agreeing on the state does not mean that they all have the latest contents.

From the state machine view, it peering process includes three states.

1. GetInfo
2. GetLog
3. GetMissing

GetInfo state collects peer_info from chosen peer OSD candidates. 

Using peer_info from GetInfo, GetLog state generate an authoritative log.

Using this log, GetMissing calculates missing objects from OSD. Those missing objects info will be used in recovery/backfill process.

This article will go through those states and explain what have done in those states.

![PG peering](/public/images/2017-11-05/peer.png)

---
#### GetInfo

![GetInfo](/public/images/2017-11-05/GetInfo.png)

1. generate_past_intervals()

2. build_prior()

   choose OSD candidates to query peer info from

3. get_infos()

4. proc_replica_info()

```c++
/*--------GetInfo---------*/
//Started/Primary/Peering/GetInfo
GetInfo(my_context ctx)
{
  pg->generate_past_intervals();

  if (!prior_set.get())
    pg->build_prior(prior_set);

  get_infos();
  if (peer_info_requested.empty() 
  		&& !prior_set->pg_down) {
    post_event(GotInfo());
  }
}
```

1 calculate several past intervals
  
```c++
void PG::generate_past_intervals() {
  interval_range // from start(last_epoch_clean) to
                 // end(curent osd map epoch)
}
```
 
2 get probe targets (build prior)

  build probe targets, using past_intervals. 
  
  who to ask for pg_info message

  2.1 add acting set

  2.2 add up set

  2.3 go through past intervals

   > go through interval.acting.If this osd is still up `probe.insert();`


```c++
  PG::PriorSet::PriorSet()
  {
    for (unsigned i=0; i<acting.size(); i++) {
      probe.insert();
    }

    for (unsigned i=0; i<up.size(); i++) {
      probe.insert();
    }

    for (past_intervals) {
      for (interval.acting.size()) {
        if (osdmap.is_up(o)) {
          // include past acting osds if they are up.
          probe.insert(so);
          up_now.insert(so);
        }
      }
    }
  }
```

3 get_infos send request to ask pg_info

```c++
void PG::RecoveryState::GetInfo::get_infos() 
{
  for (prior_set->probe) {
    context< RecoveryMachine >().send_query(
        pg_query_t::INFO);
  }
}
```


4 When PG received MNotifyRec event(pg_info),

```c++
  GetInfo::react(const MNotifyRec& infoevt) 
  {
    if (pg->proc_replica_info()) {
      if (we got an new last_epoch_started) {
        //last_epoch_started moved forward, rebuilding prior
        pg->build_perior();
        get_infos();
      }
      //all requested info came back
      if (peer_info_requested.empty() && !prior_set->pg_down) {
        /*
        * make sure we have at least one !incomplete() osd from the
        * last rw interval.  the incomplete (backfilling) replicas
        * get a copy of the log, but they don't get all the object
        * updates, so they are insufficient to recover changes 
        * during that interval.
        */
        for (map<epoch_t,pg_interval_t>::reverse_iterator 
            p = pg->past_intervals.rbegin();
            p != pg->past_intervals.rend();
            ++p) { 
          /*
           * this mirrors the PriorSet calculation: we wait if we
           * don't have an up (AND !incomplete) node AND there are
           * nodes down that might be usable.
           */
          for (unsigned i=0; i<interval.acting.size(); i++) {
            check if there is any OSD is up & !incomplete 
          }
          if (!any_up_complete_now && any_down_now) {
            return discard_event();
          } 
        }
        post_event(GotInfo());
      }
    }
  }
```

  ```c++
  bool PG::proc_replica_info() 
  {
     assert(is_primary());
     peer_info[from] = oinfo;
     //used for pg recovery
     might_have_unfound.insert(from);
     
     if (new info) {
       update_heartbeat_peers();
     }
  }
  ```
---

#### GetLog

![GetLog](/public/images/2017-11-05/GetLog.png)

1. choose_acting()
2. send_query(LOG)
3. proc_master_log()

```c++
//Started/Primary/Peering/GetLog"
PG::RecoveryState::GetLog::GetLog(my_context ctx)
{
  pg->choose_acting();
  
  // am i the best?
  if (auth_log_shard == pg->pg_whoami) {
    post_event(GotLog());
    return;
  }
  // how much?
  dout(10) << " requesting log from osd." << auth_log_shard << dendl;
  context<RecoveryMachine>().send_query(
    auth_log_shard,
    pg_query_t(
      pg_query_t::LOG,
      auth_log_shard.shard, pg->pg_whoami.shard,
      request_log_from, pg->info.history,
      pg->get_osdmap()->get_epoch()));
}

```

choose_acting()

1. find Authoritative History

   find_best_info()

2. choose acting set

   calc_replicated_acting()

```c++
//1. find Authoritative History
//2. choose acting set
bool PG::choose_acting() 
{
  find_best_info();
  if (cant find best OSD who have authoritative log) {
  	return false;
  }
  calc_replicated_acting();
  //check if recoverable
  if (!(*recoverable_predicate)(have)) {
	want_acting.clear();
	dout(10) << "choose_acting failed, not recoverable" << dendl;
	return false;
  }
  //request pg_temp
  if (want != acting) {
    dout(10) << "choose_acting want " << want << " != acting " << acting
			   << ", requesting pg_temp change" << dendl;
    want_acting = want;
    if (want_acting == up) {
      osd->queue_want_pg_temp(info.pgid.pgid, empty);
    }
    osd->queue_want_pg_temp(info.pgid.pgid, want);
  }
}
```


```c++
/**
 * find_best_info
 *
 * Returns an iterator to the best info in infos sorted by:
 *  1) Prefer newer last_update
 *  2) Prefer longer tail if it brings another info into contiguity
 *  3) Prefer current primary
 */
PG::find_best_info() 
{
  must :max_last_epoch_started_found
        not incomplete
  
  prefer newer last_update
  prefer lognger tail
  prefer current primary
  
}
```



```c++
//calculate the desired acting set.
void PG::calc_replicated_acting()
{
  choose "size" number of candidates 
    up is the first choice then acting then all_info
  
  want = primary + up && no need backfill+ acting + all_info;
  acting_backfill = primary + up && need backfill + up && no need backfill;
  						+ acting + all_info;
  backfill = in up set need backfill;
}
```



After received authoritative log:

```c++
PG::RecoveryState::GetLog::react(const GotLog&) 
{
  //when merge log, primary PG's missing objects is calculated
  //Auth_osd get its missing back in msg, it will be stored in peer_missing
  pg->proc_master_log(*context<RecoveryMachine>().get_cur_transaction(),
      msg->info, msg->log, msg->missing,auth_log_shard);
  return transit< GetMissing >();
}
```

---

#### GetMissing

![GetMissing](/public/images/2017-11-05/GetMissing.png)

```c++
PG::RecoveryState::GetMissing::GetMissing(my_context ctx)
{
  for (actingbackfill) {
    if (need backfill or dont need recovery) {
      continue;
    }
    // We pull the log from the peer's last_epoch_started to ensure we
    // get enough log to detect divergent updates.
    if (log_tail <= last_epoch_started) {
      // request log after last_epoch_started
      context< RecoveryMachine >().send_query();
    } else {
	    // request full log
      context< RecoveryMachine >().send_query();
    }
    peer_missing_requested.insert();
  }
  if (peer_missing_requested.empty()) {
    post_event(Activate(pg->get_osdmap()->get_epoch()));
  }
}
```

After received log from actingbackfill memeber

```c++
PG::RecoveryState::GetMissing::react(const MLogRec& logevt)
{
  peer_missing_requested.erase(logevt.from);
  //will add missing info to peer_missing
  pg->proc_replica_log();
  if (peer_missing_requested.empty()) {
    post_event(Activate(pg->get_osdmap()->get_epoch()));
  }
  return discard_event();
}
```
---


If you think about it, is it a little bit unnecessary to calculate acting_backfill in GetLog while it is just did almost the same thing, calculate probe targets in GetInfo?

To dig deeper, GetInfo states is to collect enough peer_info. So the probe targets is a relatively large range of set. It will cover osds from last_epoch_clean to curent osd map epoch. It needs to cover enough peer_info for the later GetLog statue. However, GetLog is to generate authoritative log. The acting_backfill need to be as accurate as possible. It is a relatively conversative set. The number of the set is fixed not to over PG size.


#### Reference

[Source Code](https://github.com/whoiami/ceph)

[Ceph Document](http://docs.ceph.com/docs/master/)

[Ceph源码分析](https://book.douban.com/subject/26914637/)
