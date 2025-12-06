# Code Review and Analysis

## Prelude

[patch0 url](https://lore.kernel.org/all/874k812fl8.fsf@yhuang6-desk2.ccr.corp.intel.com/)

## Patch 1

[patch1 url](https://lore.kernel.org/all/e3db05f4bd7dd9363c2a895875505088d3273bb8.1637778851.git.hasanalmaruf@fb.com/)

Main Goal: Add Promotion and demoting statistics to rate limit the migration across NUMA nodes

- PG_demote bit
- statistics

    ```text
    promotion related statistics:
    =============================
    pgpromote_candidate - candidates that get selected for promotion
    pgpromote_candidate_demoted - promotion candidate that got demoted earlier
    pgpromote_candidate_anon - promotion candidate that are anon
    pgpromote_candidate_file - promotion candidate that are file
    pgpromote_tried - pages that had a try to migrate via NUMA Balancing
    pgpromote_file- successfully promoted file pages
    pgpromote_anon - successfully promoted anon pages

    promotion failure related statistics:
    =====================================
    pgmigrate_fail_dst_node_full - failed as the target node is full
    pgmigrate_fail_numa_isolate - failed in isolating numa page
    pgmigrate_fail_nomem - failed as no memory left in the system
    pgmigrate_fail_refcount - failed as ref count mismatched

    demotion related statistics:
    ============================
    pgdemote_file - successfully demoted file pages
    pgdemote_anon - successfully demoted anon pages
    ```

## Patch 2

[patch2 url](https://lore.kernel.org/all/06f961992a2c119ed0904825d8ab3f2b2a2c682b.1637778851.git.hasanalmaruf@fb.com/)

If a system has single toptier node online, default NUMA balancing will
automatically be downgraded to the tiered-memory mode to avoid the
unnecessary scanning in the toptier node mentioned above.

```c
+/*
+ * If there is only one toptier node available, pages on that
+ * node can not be promotrd to anywhere. In that case, downgrade
+ * to numa_promotion_tiered_enabled mode
+ */
+static void check_numa_promotion_mode(void)
+{
+	int node, toptier_node_count = 0;
+
+	for_each_online_node(node) {
+		if (node_is_toptier(node))
+			++toptier_node_count;
+	}
+	if (toptier_node_count == 1) {
+		numa_promotion_tiered_enabled = true;
+	}
+}
```