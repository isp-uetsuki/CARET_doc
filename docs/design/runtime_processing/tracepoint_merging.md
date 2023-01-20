# Tracepoint merging
As explained in [Tracepoint filtering](./tracepoint_filtering.md), trace data discarding occurs when recording real application such as Autoware. To mitigate this problem, we merge some events to single event.

## Basic idea
Some events allways occure consecutively. For instance, `callback_start` is always followed by `callback_end`. When focusing on a single thread, consecutive `callback_start` and `callback_end` are always related to the same callback object, which means CARET always merge these two events on analyzation phase. Therefore, we can merge these two events at recording time without changing the meaning of these events. Also, we can utilize thread local storage for merging these events as follows:

```c++
static thread_local StructToStoreCallbackStart callback_start_storage;

void ros_trace_callback_start(const void * callback, bool is_intra_process)
{
  // save current timestamp and parameters to callback_start_storage
}

void ros_trace_callback_end(const void * callback)
{
  // get timestamp and parameters of callback_start from callback_start_storage and record merged event
}
```

By merging multiple events, we can reduce data size recorded in RTTng buffer, which reduce the possibility of trace data discarding. In this case, original data size is as follows:

| event | parameter | bytes |
|---|---|---|
| callback_start | timestamp | 8 |
| | callback_object | 8 |
| | is_intra_process | 4 |
| | overhead for each event | some |
| callback_end | timestamp | 8 |
| | callback_object | 8 | 
| | overhead for each event | some |

Merged data size is as follows:

| event | parameter | bytes |
|---|---|---|
| merged_callback_timing | timestamp | 8 |
| | callback_start_timestamp | 8 |
| | callback_object | 8 |
| | is_intra_process | 4 |
| | overhead for each event | some |

## Naming of merged event
We name merged event as `merged_*` to clearly indicate that these events are merged by CARET.

## Merging `callback_start` and `callback_end`
As discussed in upper section, `callback_start` and `callback_end` can be merged. We merge these events to a single `merged_callback_timing` event.

## Merging consecutive events on publishing
Following events has the same relation with  `callback_start` and `callback_end`: they are always consecutive, and events from the same thread are always merged by CARET on analyzation phase.

- rclcpp_publish
- rcl_publish
- dds_write
- dds_bind_addr_to_stamp

We merge these events to a single `merged_publish_timing` event.

## Another combination of events
Another combination of events are not always consecutive. Therefore, they cannot be merged.

For instance, `callback_start` sometimes followed by single `rclcpp_publish`, but not always: `callback_start` can be followed by multipe `rclcpp_publish`'s, also `callback_start` can be followed by no `rclcpp_publish`. It means we cannot merge these events because we cannot create fixed-size merged tracepoint for these events.
