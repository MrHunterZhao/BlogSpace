---
title: 【JVM源码探秘】深入System.gc()底层实现
date: 2018-04-05 19:30:00
categories: OpenJDK
tags:
    - OpenJDK
    - JVM
    - HotSpot
---

![openjdk](/images/post/2018/01/29/openjdk.jpg)

 
对于Java语言来说是不用手动释放内存的，并且不需要手动干预JVM的GC行为，但在一些监控和agent工具里却是必要的。

Hotspot为我们开放了Java语言级别的GC手动触发入口`System.gc()`，本文将深入介绍JVM底层实现。

<!-- more -->

# java.lang.System#gc()

```java
    /**
     * Runs the garbage collector.
     * <p>
     * Calling the <code>gc</code> method suggests that the Java Virtual
     * Machine expend effort toward recycling unused objects in order to
     * make the memory they currently occupy available for quick reuse.
     * When control returns from the method call, the Java Virtual
     * Machine has made a best effort to reclaim space from all discarded
     * objects.
     * <p>
     * The call <code>System.gc()</code> is effectively equivalent to the
     * call:
     * <blockquote><pre>
     * Runtime.getRuntime().gc()
     * </pre></blockquote>
     *
     * @see     java.lang.Runtime#gc()
     */
    public static void gc() {
        Runtime.getRuntime().gc();
    }
```

# java.lang.Runtime#gc()

```java
    /**
     * Runs the garbage collector.
     * Calling this method suggests that the Java virtual machine expend
     * effort toward recycling unused objects in order to make the memory
     * they currently occupy available for quick reuse. When control
     * returns from the method call, the virtual machine has made
     * its best effort to recycle all discarded objects.
     * <p>
     * The name <code>gc</code> stands for "garbage
     * collector". The virtual machine performs this recycling
     * process automatically as needed, in a separate thread, even if the
     * <code>gc</code> method is not invoked explicitly.
     * <p>
     * The method {@link System#gc()} is the conventional and convenient
     * means of invoking this method.
     */
    public native void gc();
```

# Runtime.c #  Java_java_lang_Runtime_gc()
这里调用了native方法`gc()`，对应的方法在Hotspot源码`src/java.base/share/native/libjava/Runtime.c`

```c
JNIEXPORT void JNICALL
Java_java_lang_Runtime_gc(JNIEnv *env, jobject this)
{
    JVM_GC();
}
```

# jvm.cpp

方法实现在`src/hotspot/share/prims/jvm.cpp`

```c
JVM_ENTRY_NO_ENV(void, JVM_GC(void))
  JVMWrapper("JVM_GC");
  if (!DisableExplicitGC) {
     // 调用具体堆实现的collect方法
    Universe::heap()->collect(GCCause::_java_lang_system_gc);
  }
JVM_END
```

通过Universe调用具体堆实现的collect方法，取决于使用当前实例使用的GC模式，在JVM中目前堆实现主要有：
- 串行回收堆
    - src/hotspot/share/gc/serial/defNewGeneration.cpp（年轻代）
    - src/hotspot/share/gc/serial/tenuredGeneration.cpp（年老代）
- 并行回收堆
    - src/hotspot/share/gc/parallel/parallelScavengeHeap.cpp
- CMS并发回收堆
    - src/hotspot/share/gc/shared/genCollectedHeap.cpp
- G1并发回收堆
    - src/hotspot/share/gc/g1/g1CollectedHeap.cpp



# 串行回收堆（年轻代）
```c
void DefNewGeneration::collect(bool   full,
                               bool   clear_all_soft_refs,
                               size_t size,
                               bool   is_tlab) {
  assert(full || size > 0, "otherwise we don't want to collect");

  GenCollectedHeap* gch = GenCollectedHeap::heap();

  _gc_timer->register_gc_start();
  DefNewTracer gc_tracer;
  gc_tracer.report_gc_start(gch->gc_cause(), _gc_timer->gc_start());

  _old_gen = gch->old_gen();

  // 如果下个代空间不足以容纳当前代空间中即将晋升的对象，则标记让下个代空间先进行回收
  // If the next generation is too full to accommodate promotion
  // from this generation, pass on collection; let the next generation
  // do it.
  if (!collection_attempt_is_safe()) {
    log_trace(gc)(":: Collection attempt not safe ::");
    gch->set_incremental_collection_failed(); // Slight lie: we did not even attempt one
    return;
  }
  assert(to()->is_empty(), "Else not collection_attempt_is_safe");

  init_assuming_no_promotion_failure();

  GCTraceTime(Trace, gc, phases) tm("DefNew", NULL, gch->gc_cause());

  gch->trace_heap_before_gc(&gc_tracer);

  // These can be shared for all code paths
  IsAliveClosure is_alive(this);
  ScanWeakRefClosure scan_weak_ref(this);

  age_table()->clear();
  to()->clear(SpaceDecorator::Mangle);
  // The preserved marks should be empty at the start of the GC.
  _preserved_marks_set.init(1);

  gch->rem_set()->prepare_for_younger_refs_iterate(false);

  assert(gch->no_allocs_since_save_marks(),
         "save marks have not been newly set.");

  // Not very pretty.
  CollectorPolicy* cp = gch->collector_policy();

  FastScanClosure fsc_with_no_gc_barrier(this, false);
  FastScanClosure fsc_with_gc_barrier(this, true);

  KlassScanClosure klass_scan_closure(&fsc_with_no_gc_barrier,
                                      gch->rem_set()->klass_rem_set());
  CLDToKlassAndOopClosure cld_scan_closure(&klass_scan_closure,
                                           &fsc_with_no_gc_barrier,
                                           false);

  set_promo_failure_scan_stack_closure(&fsc_with_no_gc_barrier);
  FastEvacuateFollowersClosure evacuate_followers(gch,
                                                  &fsc_with_no_gc_barrier,
                                                  &fsc_with_gc_barrier);

  assert(gch->no_allocs_since_save_marks(),
         "save marks have not been newly set.");

  {
    // DefNew needs to run with n_threads == 0, to make sure the serial
    // version of the card table scanning code is used.
    // See: CardTableModRefBSForCTRS::non_clean_card_iterate_possibly_parallel.
    StrongRootsScope srs(0);

    gch->young_process_roots(&srs,
                             &fsc_with_no_gc_barrier,
                             &fsc_with_gc_barrier,
                             &cld_scan_closure);
  }

  // "evacuate followers".
  evacuate_followers.do_void();

  FastKeepAliveClosure keep_alive(this, &scan_weak_ref);
  ReferenceProcessor* rp = ref_processor();
  rp->setup_policy(clear_all_soft_refs);
  ReferenceProcessorPhaseTimes pt(_gc_timer, rp->num_q());
  const ReferenceProcessorStats& stats =
  rp->process_discovered_references(&is_alive, &keep_alive, &evacuate_followers,
                                    NULL, &pt);
  gc_tracer.report_gc_reference_stats(stats);
  gc_tracer.report_tenuring_threshold(tenuring_threshold());
  pt.print_all_references();

  if (!_promotion_failed) {
    // Swap the survivor spaces.
    eden()->clear(SpaceDecorator::Mangle);
    from()->clear(SpaceDecorator::Mangle);
    if (ZapUnusedHeapArea) {
      // This is now done here because of the piece-meal mangling which
      // can check for valid mangling at intermediate points in the
      // collection(s).  When a young collection fails to collect
      // sufficient space resizing of the young generation can occur
      // an redistribute the spaces in the young generation.  Mangle
      // here so that unzapped regions don't get distributed to
      // other spaces.
      to()->mangle_unused_area();
    }
    swap_spaces();

    assert(to()->is_empty(), "to space should be empty now");

    adjust_desired_tenuring_threshold();

    // A successful scavenge should restart the GC time limit count which is
    // for full GC's.
    AdaptiveSizePolicy* size_policy = gch->gen_policy()->size_policy();
    size_policy->reset_gc_overhead_limit_count();
    assert(!gch->incremental_collection_failed(), "Should be clear");
  } else {
    assert(_promo_failure_scan_stack.is_empty(), "post condition");
    _promo_failure_scan_stack.clear(true); // Clear cached segments.

    remove_forwarding_pointers();
    log_info(gc, promotion)("Promotion failed");
    // Add to-space to the list of space to compact
    // when a promotion failure has occurred.  In that
    // case there can be live objects in to-space
    // as a result of a partial evacuation of eden
    // and from-space.
    swap_spaces();   // For uniformity wrt ParNewGeneration.
    from()->set_next_compaction_space(to());
    gch->set_incremental_collection_failed();

    // Inform the next generation that a promotion failure occurred.
    _old_gen->promotion_failure_occurred();
    gc_tracer.report_promotion_failed(_promotion_failed_info);

    // Reset the PromotionFailureALot counters.
    NOT_PRODUCT(gch->reset_promotion_should_fail();)
  }
  // We should have processed and cleared all the preserved marks.
  _preserved_marks_set.reclaim();
  // set new iteration safe limit for the survivor spaces
  from()->set_concurrent_iteration_safe_limit(from()->top());
  to()->set_concurrent_iteration_safe_limit(to()->top());

  // We need to use a monotonically non-decreasing time in ms
  // or we will see time-warp warnings and os::javaTimeMillis()
  // does not guarantee monotonicity.
  jlong now = os::javaTimeNanos() / NANOSECS_PER_MILLISEC;
  update_time_of_last_gc(now);

  gch->trace_heap_after_gc(&gc_tracer);

  _gc_timer->register_gc_end();

  gc_tracer.report_gc_end(_gc_timer->gc_end(), _gc_timer->time_partitions());
}
```


# 串行回收堆（年老代）
```c
void TenuredGeneration::collect(bool   full,
                                bool   clear_all_soft_refs,
                                size_t size,
                                bool   is_tlab) {
  GenCollectedHeap* gch = GenCollectedHeap::heap();

  // Temporarily expand the span of our ref processor, so
  // refs discovery is over the entire heap, not just this generation
  ReferenceProcessorSpanMutator
    x(ref_processor(), gch->reserved_region());

  STWGCTimer* gc_timer = GenMarkSweep::gc_timer();
  gc_timer->register_gc_start();

  SerialOldTracer* gc_tracer = GenMarkSweep::gc_tracer();
  gc_tracer->report_gc_start(gch->gc_cause(), gc_timer->gc_start());

  gch->pre_full_gc_dump(gc_timer);

  GenMarkSweep::invoke_at_safepoint(ref_processor(), clear_all_soft_refs);

  gch->post_full_gc_dump(gc_timer);

  gc_timer->register_gc_end();

  gc_tracer->report_gc_end(gc_timer->gc_end(), gc_timer->time_partitions());
}
```

# 串行回收堆
```c
// This method is used by System.gc() and JVMTI.
void ParallelScavengeHeap::collect(GCCause::Cause cause) {
  assert(!Heap_lock->owned_by_self(),
    "this thread should not own the Heap_lock");

  uint gc_count      = 0;
  uint full_gc_count = 0;
  {
    MutexLocker ml(Heap_lock);
    // This value is guarded by the Heap_lock
    gc_count      = total_collections();
    full_gc_count = total_full_collections();
  }

  VM_ParallelGCSystemGC op(gc_count, full_gc_count, cause);
  VMThread::execute(&op);
}
```


# CMS分代并发回收堆
```c
// public collection interfaces

void GenCollectedHeap::collect(GCCause::Cause cause) {
  if (should_do_concurrent_full_gc(cause)) {
#if INCLUDE_ALL_GCS
    // Mostly concurrent full collection.
    collect_mostly_concurrent(cause);
#else  // INCLUDE_ALL_GCS
    ShouldNotReachHere();
#endif // INCLUDE_ALL_GCS
  } else if (cause == GCCause::_wb_young_gc) {
    // Young collection for the WhiteBox API.
    collect(cause, YoungGen);
  } else {
#ifdef ASSERT
  if (cause == GCCause::_scavenge_alot) {
    // Young collection only.
    collect(cause, YoungGen);
  } else {
    // Stop-the-world full collection.
    collect(cause, OldGen);
  }
#else
    // Stop-the-world full collection.
    collect(cause, OldGen);
#endif
  }
}
```


# G1并发回收堆
```c
void G1CollectedHeap::collect(GCCause::Cause cause) {
  assert_heap_not_locked();

  uint gc_count_before;
  uint old_marking_count_before;
  uint full_gc_count_before;
  bool retry_gc;

  do {
    retry_gc = false;

    {
      MutexLocker ml(Heap_lock);

      // Read the GC count while holding the Heap_lock
      gc_count_before = total_collections();
      full_gc_count_before = total_full_collections();
      old_marking_count_before = _old_marking_cycles_started;
    }

    if (should_do_concurrent_full_gc(cause)) {
      // Schedule an initial-mark evacuation pause that will start a
      // concurrent cycle. We're setting word_size to 0 which means that
      // we are not requesting a post-GC allocation.
      VM_G1IncCollectionPause op(gc_count_before,
                                 0,     /* word_size */
                                 true,  /* should_initiate_conc_mark */
                                 g1_policy()->max_pause_time_ms(),
                                 cause);
      op.set_allocation_context(AllocationContext::current());

      VMThread::execute(&op);
      if (!op.pause_succeeded()) {
        if (old_marking_count_before == _old_marking_cycles_started) {
          retry_gc = op.should_retry_gc();
        } else {
          // A Full GC happened while we were trying to schedule the
          // initial-mark GC. No point in starting a new cycle given
          // that the whole heap was collected anyway.
        }

        if (retry_gc) {
          if (GCLocker::is_active_and_needs_gc()) {
            GCLocker::stall_until_clear();
          }
        }
      }
    } else {
      if (cause == GCCause::_gc_locker || cause == GCCause::_wb_young_gc
          DEBUG_ONLY(|| cause == GCCause::_scavenge_alot)) {

        // Schedule a standard evacuation pause. We're setting word_size
        // to 0 which means that we are not requesting a post-GC allocation.
        VM_G1IncCollectionPause op(gc_count_before,
                                   0,     /* word_size */
                                   false, /* should_initiate_conc_mark */
                                   g1_policy()->max_pause_time_ms(),
                                   cause);
        VMThread::execute(&op);
      } else {
        // Schedule a Full GC.
        VM_G1CollectFull op(gc_count_before, full_gc_count_before, cause);
        VMThread::execute(&op);
      }
    }
  } while (retry_gc);
}
```



