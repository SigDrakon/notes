

# 1. PFCP 信令处理函数





PFCP 信令处理的最开始，毫无疑问是 PFCP 消息的处理函数，即 `pfcp_process` 作为一个节点存在









```c
int
upf_pfcp_handle_msg (pfcp_msg_t * msg)
{
  pfcp_decoded_msg_t dmsg;
  pfcp_offending_ie_t *err = NULL;
  u8 type = pfcp_msg_type (msg->data);
  int r;

  if (type >= ARRAY_LEN (msg_handlers) || !msg_handlers[type])
    {
      /* probably non-PFCP datagram, nothing to reply */
      upf_debug ("PFCP: msg type invalid: %d.", type);
      return -1;
    }

  r = pfcp_decode_msg (msg->data, vec_len (msg->data), &dmsg, &err);
  if (r < 0)
    {
      /* not enough info in the message to produce any meaningful reply */
      upf_debug ("PFCP: broken message");
      return -1;
    }

  if (r != 0)
    {
      switch (dmsg.type)
      {
      case PFCP_HEARTBEAT_REQUEST:
      case PFCP_PFD_MANAGEMENT_REQUEST:
      case PFCP_ASSOCIATION_SETUP_REQUEST:
      case PFCP_ASSOCIATION_UPDATE_REQUEST:
      case PFCP_ASSOCIATION_RELEASE_REQUEST:
      case PFCP_SESSION_SET_DELETION_REQUEST:
      case PFCP_SESSION_ESTABLISHMENT_REQUEST:
      case PFCP_SESSION_MODIFICATION_REQUEST:
      case PFCP_SESSION_DELETION_REQUEST:
      case PFCP_SESSION_REPORT_REQUEST:
        send_simple_response (msg, 0, dmsg.type + 1, r, err);
        break;

      default:
        break;
      }

      pfcp_free_dmsg_contents (&dmsg);
      vec_free (err);
      return r;
    }

  r = msg_handlers[type] (msg, &dmsg);
  pfcp_free_dmsg_contents (&dmsg);

  return r;
}
```









```c
typedef int (*msg_handler_t) (pfcp_msg_t * msg, pfcp_decoded_msg_t * dmsg);

static msg_handler_t msg_handlers[] = {
  [PFCP_HEARTBEAT_REQUEST] = handle_heartbeat_request,
  [PFCP_HEARTBEAT_RESPONSE] = handle_heartbeat_response,
  [PFCP_PFD_MANAGEMENT_REQUEST] = handle_pfd_management_request,
  [PFCP_PFD_MANAGEMENT_RESPONSE] = handle_pfd_management_response,
  [PFCP_ASSOCIATION_SETUP_REQUEST] = handle_association_setup_request,
  [PFCP_ASSOCIATION_SETUP_RESPONSE] = handle_association_setup_response,
  [PFCP_ASSOCIATION_UPDATE_REQUEST] = handle_association_update_request,
  [PFCP_ASSOCIATION_UPDATE_RESPONSE] = handle_association_update_response,
  [PFCP_ASSOCIATION_RELEASE_REQUEST] = handle_association_release_request,
  [PFCP_ASSOCIATION_RELEASE_RESPONSE] = handle_association_release_response,
  [PFCP_VERSION_NOT_SUPPORTED_RESPONSE] = 0,	/* handle_version_not_supported_response, */
  [PFCP_NODE_REPORT_REQUEST] = handle_node_report_request,
  [PFCP_NODE_REPORT_RESPONSE] = handle_node_report_response,
  [PFCP_SESSION_SET_DELETION_REQUEST] = handle_session_set_deletion_request,
  [PFCP_SESSION_SET_DELETION_RESPONSE] = handle_session_set_deletion_response,
  [PFCP_SESSION_ESTABLISHMENT_REQUEST] = handle_session_establishment_request,
  [PFCP_SESSION_ESTABLISHMENT_RESPONSE] = handle_session_establishment_response,
  [PFCP_SESSION_MODIFICATION_REQUEST] = handle_session_modification_request,
  [PFCP_SESSION_MODIFICATION_RESPONSE] = handle_session_modification_response,
  [PFCP_SESSION_DELETION_REQUEST] = handle_session_deletion_request,
  [PFCP_SESSION_DELETION_RESPONSE] = handle_session_deletion_response,
  [PFCP_SESSION_REPORT_REQUEST] = handle_session_report_request,
  [PFCP_SESSION_REPORT_RESPONSE] = handle_session_report_response,
};
```



## 1.1. 偶联请求处理













# 2. 流匹配



在处理网络流（Flow）时，一个连接（例如，从 A 到 B）会包含两个方向的数据包：A->B 和 B->A。为了能让这两个方向的数据包都命中同一个流表项（flow entry），系统需要一种与数据包实际传输方向无关的、统一的表示方法。

```c
static inline flow_direction_t
ip4_packet_is_reverse (ip4_header_t * ip4)
{
  return (ip4_address_compare (&ip4->src_address, &ip4->dst_address) < 0) ?
    FT_REVERSE : FT_ORIGIN;
}

int
ip4_address_compare (ip4_address_t * a1, ip4_address_t * a2)
{
  return clib_net_to_host_u32 (a1->data_u32) -
    clib_net_to_host_u32 (a2->data_u32);
}
```

这个函数就是实现这种统一表示的关键。它通过比较数据包的源IP地址和目的IP地址的数值大小来做到这一点：

- 这个内部函数比较源IP和目的IP的数值大小。它会先将网络字节序的IP地址转换为本地主机字节序的无符号32位整数，然后进行减法操作。
- 如果 src ip < dst ip ，ip4_address_compare 返回一个负数
- 如果 src ip >= dst ip ，ip4_address_compare 返回一个非负数

如果 ip4_address_compare 的结果小于 0（即 src ip < dst ip ），函数返回 FT_REVERSE，否则返回 FT_ORIGIN



这个函数定义了一个约定：

- 当数据包的源IP地址数值上**小于**目的IP地址时，这个包的方向被定义为 `FT_REVERSE`。
- 当数据包的源IP地址数值上**大于或等于**目的IP地址时，这个包的方向被定义为 `FT_ORIGIN`。

















# 3. ethernet_input_node



## 3.1 函数签名

```c
static_always_inline void
ethernet_input_inline (vlib_main_t * vm,
		       vlib_node_runtime_t * node,
		       u32 * from, u32 n_packets,
		       ethernet_input_variant_t variant)
```

- **`static_always_inline`**: 这是一个静态内联函数，意味着它会被编译器直接嵌入到调用它的地方，以减少函数调用的开销。
- **`vlib_main_t * vm`**: 指向 VLIB 主循环结构的指针。
- **`vlib_node_runtime_t * node`**: 指向当前节点（`ethernet-input`）的运行时数据。
- **`u32 * from`**: 一个包含 `vlib_buffer_t` 索引的数组，代表了当前帧中所有待处理的数据包。
- **`u32 n_packets`**: `from` 数组中的数据包数量。
- **`ethernet_input_variant_t variant`**: 一个枚举，用于区分不同的调用场景。例如，`ETHERNET_INPUT_VARIANT_ETHERNET` 是最常见的，表示处理标准的以太网帧。



## 3.2 变量初始化



```c
  vnet_main_t *vnm = vnet_get_main ();
  ethernet_main_t *em = &ethernet_main;
  vlib_node_runtime_t *error_node;
  u32 n_left_from, next_index, *to_next;
  u32 stats_sw_if_index, stats_n_packets, stats_n_bytes;
  u32 thread_index = vm->thread_index;
  u32 cached_sw_if_index = ~0;
  u32 cached_is_l2 = 0;		/* shut up gcc */
  vnet_hw_interface_t *hi = NULL;	/* used for main interface only */
  ethernet_interface_t *ei = NULL;
  vlib_buffer_t *bufs[VLIB_FRAME_SIZE];
  vlib_buffer_t **b = bufs;
```

- 获取各种主结构的指针，如 `vnet_main_t` 和 `ethernet_main_t`。
- `error_node`: 用于记录错误计数。
- `n_left_from`, `from`, `to_next`: 用于 VPP 的标准数据包处理循环。
- `stats_*`: 用于批量更新接口统计计数器。
- `cached_*`: 用于缓存上一个数据包的接口信息，如果连续的数据包来自同一个接口，可以跳过一些重复的查找操作，以提高性能。
- `bufs`, `b`: 用于存储指向 `vlib_buffer_t` 的指针，方便批量处理。





## 3.3 主循环处理

VPP 的节点函数通常使用一个 `while (n_left_from > 0)` 循环来处理一帧（vector）中的所有数据包。内部又会有一个或多个嵌套的 `while` 循环，通常以 2 个或 4 个数据包为一批进行处理，以充分利用 CPU 的流水线和 SIMD 指令。

```c
while (n_left_from > 0)
{
  u32 n_left_to_next;
  vlib_get_next_frame (vm, node, next_index, to_next, n_left_to_next);
  
  while ( n_left_from >= 4 && n_left_to_next >= 2 ) {
      // do something
  }
    
  while ( n_left_from >= 0 && n_left_to_next >= 0 ) {
      // do something
  }
 
  vlib_put_next_frame (vm, node, next_index, n_left_to_next);
}
```



### 3.3.1 预取

```c
	  /* Prefetch next iteration. */
	  {
	    vlib_prefetch_buffer_header (b[2], STORE);
	    vlib_prefetch_buffer_header (b[3], STORE);

	    CLIB_PREFETCH (b[2]->data, sizeof (ethernet_header_t), LOAD);
	    CLIB_PREFETCH (b[3]->data, sizeof (ethernet_header_t), LOAD);
	  }
```

- 在处理当前批次的数据包（`b[0]`, `b[1]`）之前，提前将下一批次的数据包（`b[2]`, `b[3]`）的头部和数据加载到 CPU 缓存中。这可以隐藏内存访问延迟，是 VPP 高性能的关键技术之一。



### 3.3.2 快速路径：处理无 VLAN 的普通以太网帧



```c
if ( PREDICT_TRUE 
     (variant == ETHERNET_INPUT_VARIANT_ETHERNET && !ethernet_frame_is_any_tagged_x2 (type0,type1) )
   )
{ // do something
}
```

- 这是一个高度优化的快速路径。`PREDICT_TRUE` 告诉编译器，这个条件绝大多数情况下为真。
- `ethernet_frame_is_any_tagged_x2` 检查两个数据包的 EtherType 是否是 VLAN 类型（如 0x8100, 0x88a8）。如果都不是，说明它们是普通的、没有 VLAN 标签的以太网帧。



缓存检查

```c
if (PREDICT_FALSE (cached_sw_if_index != sw_if_index0))
	{
	  cached_sw_if_index = sw_if_index0;
	  hi = vnet_get_sup_hw_interface (vnm, sw_if_index0);
	  ei = ethernet_get_interface (em, hi->hw_if_index);
	  intf0 = vec_elt_at_index (em->main_intfs, hi->hw_if_index);
	  subint0 = &intf0->untagged_subint;
	  cached_is_l2 = is_l20 = subint0->flags & SUBINT_CONFIG_L2;
	}
```

- 如果当前数据包的接收接口 (`sw_if_index0`) 与上一个数据包不同，则重新获取该接口的硬件信息、以太网信息和子接口配置，并缓存起来。



### 3.3.3 L2 L3 判断

- 根据接口配置判断是走 L2 路径还是 L3 路径。

```c
if (PREDICT_TRUE (is_l20 != 0))
	{
	  // ... L2 path
	}
      else
	{
	  // ... L3 path
	}
```



#### 3.3.3.1 L2 路径

- 设置 L3 头部偏移量（指向以太网头部之后）。
- 设置 L2 头部长度。
- 将数据包的下一跳设置为 L2 输入节点 (`l2-input`)。

```c
      vnet_buffer (b0)->l3_hdr_offset = ...;
      b0->flags |= VNET_BUFFER_F_L3_HDR_OFFSET_VALID;
      next0 = em->l2_next;
      vnet_buffer (b0)->l2.l2_len = sizeof (ethernet_header_t);

```



#### 3.3.3.2 L3 路径

- **MAC 地址检查**：如果接口不是预先配置为 L3 模式（`STATUS_L3`），则需要检查数据包的目的 MAC 地址是否是接口的 MAC 地址。如果不是，则标记为 `L3_MAC_MISMATCH` 错误。
- `vlib_buffer_advance`: 将数据包的 `current_data` 指针向前移动，跳过以太网头部。
- `determine_next_node`: 根据 EtherType（IPv4, IPv6, MPLS 等）确定下一跳节点

```c
      if (ei && (ei->flags & ETHERNET_INTERFACE_FLAG_STATUS_L3))
	    goto skip_dmac_check01;

      ethernet_input_inline_dmac_check (...);

      if (dmacs_bad[0])
	    error0 = ETHERNET_ERROR_L3_MAC_MISMATCH;

skip_dmac_check01:
      vlib_buffer_advance (b0, sizeof (ethernet_header_t));
      determine_next_node (em, variant, 0, type0, b0,
			       &error0, &next0);
```





determine_next_node：

该函数的的核心作用是：在以太网头部（包括可能的 VLAN 标签）被解析和剥离后，根据数据包的 EtherType（以太网类型）和接口的 L2/L3 模式，决定这个数据包应该被发送到哪个 VPP 图节点（node）进行下一步处理。

```c
static_always_inline void
determine_next_node (ethernet_main_t * em,
		     ethernet_input_variant_t variant,
		     u32 is_l20,
		     u32 type0, vlib_buffer_t * b0, u8 * error0, u8 * next0)
```

- **`static_always_inline`**: 这是一个静态内联函数，编译器会将其代码直接嵌入到调用处，以减少函数调用的开销，提升性能。
- **`ethernet_main_t \* em`**: 指向以太网主结构的指针，其中包含了各种下一跳节点的索引信息。
- **`ethernet_input_variant_t variant`**: 一个枚举，用于区分不同的调用场景。
- **`u32 is_l20`**: 一个标志，如果非零，表示这个数据包被判定为二层（L2）数据包；如果为零，则为三层（L3）数据包。
- **`u32 type0`**: 数据包的 EtherType（以太网类型），如 `0x0800` 代表 IPv4。
- **`vlib_buffer_t \* b0`**: 指向当前正在处理的数据包的指针。
- **`u8 \* error0`**: 指向错误代码的指针。如果在此函数中发现问题，会更新这个值。
- **`u8 \* next0`**: 指向下一跳节点索引的指针。这个函数的主要输出就是修改这个值。













### 3.3.4 慢速路径：处理带 VLAN 标签的帧

- 如果数据包带有 VLAN 标签，则进入此路径。
- **`parse_header`**: 解析以太网头部，剥离一层或多层 VLAN 标签，更新 `current_data` 指针，并返回最内层的 EtherType 和 VLAN ID。
- **`eth_vlan_table_lookups`**: 根据物理接口和解析出的 VLAN ID，在预先构建的哈希表中查找对应的子接口配置。
- **`identify_subint`**: 根据查找结果和匹配标志（如单层标签、双层标签、精确匹配等），最终确定数据包所属的逻辑子接口 (`new_sw_if_index`)。
- **更新 `sw_if_index[VLIB_RX]`**: 将数据包的接收接口更新为新找到的逻辑子接口索引。
- **统计和下一跳确定**: 与快速路径类似，更新子接口的统计计数，并根据 EtherType 确定下一跳节点。

```c
	slowpath:
	  parse_header (variant,
			b0,
			&type0,
			&orig_type0, &outer_id0, &inner_id0, &match_flags0);

	  old_sw_if_index0 = vnet_buffer (b0)->sw_if_index[VLIB_RX];

	  eth_vlan_table_lookups (em,
				  vnm,
				  old_sw_if_index0,
				  orig_type0,
				  outer_id0,
				  inner_id0,
				  &hi0,
				  &main_intf0, &vlan_intf0, &qinq_intf0);

	  identify_subint (em,
			   hi0,
			   b0,
			   match_flags0,
			   main_intf0,
			   vlan_intf0,
			   qinq_intf0, &new_sw_if_index0, &error0, &is_l20);

	  vnet_buffer (b0)->sw_if_index[VLIB_RX] = ...;
```



### 3.3.5 循环和分派

- **`b->error`**: 如果在处理过程中发现任何错误（如 MAC 不匹配、找不到子接口等），则设置 buffer 的错误码。
- **`vlib_validate_buffer_enqueue_x2`**: 这是一个宏，用于将处理完的两个数据包（`b0`, `b1`）放入下一跳节点的帧中。它会检查下一跳节点是否与当前缓存的 `next_index` 相同，如果不同，则需要切换到新的下一跳帧。
- **`vlib_put_next_frame`**: 当一个下一跳节点的帧满了，或者所有数据包都处理完毕时，调用此函数将该帧分派出去。

```c
	ship_it01:
	  b0->error = error_node->errors[error0];
	  b1->error = error_node->errors[error1];

	  vlib_validate_buffer_enqueue_x2 (vm, node, next_index, to_next,
					   n_left_to_next, bi0, bi1, next0,
					   next1);
	}

      vlib_put_next_frame (vm, node, next_index, n_left_to_next);
    }
```























# 4. ip_input



## 4.1. 函数签名



```c
always_inline uword
ip4_input_inline (vlib_main_t * vm,
		  vlib_node_runtime_t * node,
		  vlib_frame_t * frame, int verify_checksum)
```





## 4.2. 初始化和准备



```c
  vnet_main_t *vnm = vnet_get_main ();
  u32 n_left_from, *from;
  u32 thread_index = vm->thread_index;
  vlib_node_runtime_t *error_node =
    vlib_node_get_runtime (vm, ip4_input_node.index);
  vlib_simple_counter_main_t *cm;
  vlib_buffer_t *bufs[VLIB_FRAME_SIZE], **b;
  ip4_header_t *ip[4];
  u16 nexts[VLIB_FRAME_SIZE], *next;
  u32 sw_if_index[4];
  u32 last_sw_if_index = ~0;
  u32 cnt = 0;
  int arc_enabled = 0;

  from = vlib_frame_vector_args (frame);
  n_left_from = frame->n_vectors;

  if (node->flags & VLIB_NODE_FLAG_TRACE)
    vlib_trace_frame_buffers_only (vm, node, from, frame->n_vectors,
				   /* stride */ 1,
				   sizeof (ip4_input_trace_t));

  cm = vec_elt_at_index (vnm->interface_main.sw_if_counters,
			 VNET_INTERFACE_COUNTER_IP4);

  vlib_get_buffers (vm, from, bufs, n_left_from);
  b = bufs;
  next = nexts;
```

- 获取各种 VPP 核心结构的指针，如 `vnet_main_t`。
- 获取指向错误计数器的指针，用于记录处理过程中发生的错误。
- `from` 和 `n_left_from`：从 `frame` 中获取数据包 buffer 索引数组和数据包数量。
- `bufs` 和 `nexts`：用于暂存数据包 buffer 指针和计算出的下一跳节点索引。
- `last_sw_if_index` 和 `cnt`：用于批量更新接口统计计数器的优化变量。
- `arc_enabled`：一个标志，用于判断当前接口是否启用了任何特性（feature）。
- `vlib_get_buffers`: 批量获取 `vlib_buffer_t` 结构体指针。





## 4.3. 主循环处理

在循环内部，代码会尝试一次性处理 4 个或 2 个数据包（Dual Loop 或 Quad Loop），这是为了更好地利用现代 CPU 的指令流水线和超标量特性。

其中会使用到  ip4_input_check_x1  ip4_input_check_x2  ip4_input_check_x4 函数来分别处理不同数量的数据包

```c
#if (CLIB_N_PREFETCHES >= 8)
  while (n_left_from >= 4)
    {
      
      ip4_input_check_x4
	}
#elif (CLIB_N_PREFETCHES >= 4)


```





## 4.4. 核心处理逻辑

我们以处理单个数据包的 `ip4_input_check_x1` 函数为例来分析核心逻辑。`ip4_input_check_x2` 和 `x4` 的逻辑是类似的，只是并行处理多个包。

```c
  while (n_left_from)
    {
      u32 next0;
      vnet_buffer (b[0])->ip.adj_index[VLIB_RX] = ~0;
      sw_if_index[0] = vnet_buffer (b[0])->sw_if_index[VLIB_RX];
      ip4_input_check_sw_if_index (vm, cm, sw_if_index[0], &last_sw_if_index,
				   &cnt, &arc_enabled);
      next0 = ip4_input_set_next (sw_if_index[0], b[0], arc_enabled);
      ip[0] = vlib_buffer_get_current (b[0]);
      ip4_input_check_x1 (vm, error_node, b[0], ip[0], &next0,
			  verify_checksum);
      next[0] = next0;

      /* next */
      b += 1;
      next += 1;
      n_left_from -= 1;
    }
```



### 4.4.1. 确定下一跳和特性弧

























































# 3. ip4-lookup





## 3.1 数据预读取



数据预读取，进行访问优化

```c
      if (n_left >= 8)
	{
	  vlib_prefetch_buffer_header (b[4], LOAD);
	  vlib_prefetch_buffer_header (b[5], LOAD);
	  vlib_prefetch_buffer_header (b[6], LOAD);
	  vlib_prefetch_buffer_header (b[7], LOAD);

	  CLIB_PREFETCH (b[4]->data, sizeof (ip0[0]), LOAD);
	  CLIB_PREFETCH (b[5]->data, sizeof (ip0[0]), LOAD);
	  CLIB_PREFETCH (b[6]->data, sizeof (ip0[0]), LOAD);
	  CLIB_PREFETCH (b[7]->data, sizeof (ip0[0]), LOAD);
	}
```



## 3.2 获取数据指针

获取一个指向 VPP 缓存区 (vlib_buffer_t) 中当前有效数据起始位置的指针

```c
ip0 = vlib_buffer_get_current (b[0]);
      ip1 = vlib_buffer_get_current (b[1]);
      ip2 = vlib_buffer_get_current (b[2]);
      ip3 = vlib_buffer_get_current (b[3]);

always_inline void *
vlib_buffer_get_current (vlib_buffer_t * b)
{
  /* Check bounds. */
  ASSERT ((signed) b->current_data >= (signed) -VLIB_BUFFER_PRE_DATA_SIZE);
  return b->data + b->current_data;
}
```



了理解这一点，我们需要了解 `vlib_buffer_t` 结构体的两个关键成员：

1. **`b->data`**: 这是一个指针，指向缓冲区中用于存放网络报文数据的区域的起始点。
2. **`b->current_data`**: 这是一个**有符号的偏移量**（`i16`）。它表示当前有效数据的起始位置相对于 `b->data` 的偏移。



函数返回 `b->data + b->current_data` 的计算结果。这个地址就是数据包在当前处理阶段的“头部”或起始位置。

- **当数据包刚被网卡接收时**，`current_data` 通常是 `0` 或者一个小的正数，`vlib_buffer_get_current()` 会返回 L2（如以太网）头的起始地址。
- **当数据包在 VPP 的处理图中流转时**，每个节点（node）在处理完自己的那层头部后，通常会调用 `vlib_buffer_advance(b, header_size)`。这个函数会增加 `b->current_data` 的值，相当于将指针“向前”移动，指向下一层协议的头部。例如，`ethernet-input` 节点处理完以太网头后，会调用 `vlib_buffer_advance`，使得 `vlib_buffer_get_current()` 在下一个节点（如 `ip4-input`）返回的就是 IP 头的起始地址。
- **当需要封装（Encapsulation）时**，比如在数据包发出前添加隧道头或以太网头，处理节点会调用 `vlib_buffer_push_uninit()` 或类似函数。这会**减少** `b->current_data` 的值，使其可能变为负数，从而在现有数据**之前**留出空间来写入新的头部。`vlib_buffer_get_current()` 此时会返回一个指向预留头部空间（headroom，即 `pre_data` 区域）的指针。



`vlib_buffer_get_current` 是一个非常基础且关键的函数，它提供了一种动态获取数据包当前处理起点的机制。通过调整 `current_data` 偏移量，VPP 可以在不进行昂贵内存拷贝的情况下，高效地“剥离”（decapsulate）和“添加”（encapsulate）协议头。





















































