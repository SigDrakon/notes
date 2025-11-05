

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















