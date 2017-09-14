#sql/repl_master.cc

353：

bool com_binlog_dump_gtid(THD *thd, char *packet, size_t packet_length)
{
  DBUG_ENTER("com_binlog_dump_gtid");
  /*
    Before going GA, we need to make this protocol extensible without
    breaking compatitibilty. /Alfranio.
  */
  ushort flags= 0;
  uint32 data_size= 0;
  uint64 pos= 0;
  char name[FN_REFLEN + 1];
  uint32 name_size= 0;
  char* gtid_string= NULL;
  const uchar* packet_position= (uchar *) packet;
  size_t packet_bytes_todo= packet_length;
  Sid_map sid_map(NULL/*no sid_lock because this is a completely local object*/);
  Gtid_set slave_gtid_executed(&sid_map);

  thd->status_var.com_other++;
  thd->enable_slow_log= opt_log_slow_admin_statements;
  if (check_global_access(thd, REPL_SLAVE_ACL))
    DBUG_RETURN(false);

  READ_INT(flags,2);
  READ_INT(thd->server_id, 4);
  READ_INT(name_size, 4);
  READ_STRING(name, name_size, sizeof(name));
  READ_INT(pos, 8);
  DBUG_PRINT("info", ("pos=%llu flags=%d server_id=%d", pos, flags, thd->server_id));
  READ_INT(data_size, 4);
  CHECK_PACKET_SIZE(data_size);
  if (slave_gtid_executed.add_gtid_encoding(packet_position, data_size) !=
      RETURN_STATUS_OK)
    DBUG_RETURN(true);
  slave_gtid_executed.to_string(&gtid_string);
  DBUG_PRINT("info", ("Slave %d requested to read %s at position %llu gtid set "
                      "'%s'.", thd->server_id, name, pos, gtid_string));

  kill_zombie_dump_threads(thd);
  query_logger.general_log_print(thd, thd->get_command(),
                                 "Log: '%s' Pos: %llu GTIDs: '%s'",
                                 name, pos, gtid_string);
  my_free(gtid_string);
  mysql_binlog_send(thd, name, (my_off_t) pos, &slave_gtid_executed, flags);

  unregister_slave(thd, true, true/*need_lock_slave_list=true*/);
  /*  fake COM_QUIT -- if we get here, the thread needs to terminate */
  DBUG_RETURN(true);

error_malformed_packet:
  my_error(ER_MALFORMED_PACKET, MYF(0));
  DBUG_RETURN(true);
}

#sql/repl_slave.cc
4297 line

static int request_dump(THD *thd, MYSQL* mysql, Master_info* mi,
                        bool *suppress_warnings)
{
  DBUG_ENTER("request_dump");

  const size_t BINLOG_NAME_INFO_SIZE= strlen(mi->get_master_log_name());
  int error= 1;
  size_t command_size= 0;
  enum_server_command command= mi->is_auto_position() ?
    COM_BINLOG_DUMP_GTID : COM_BINLOG_DUMP;
  uchar* command_buffer= NULL;
  ushort binlog_flags= 0;

  if (RUN_HOOK(binlog_relay_io,
               before_request_transmit,
               (thd, mi, binlog_flags)))
    goto err;

  *suppress_warnings= false;
  if (command == COM_BINLOG_DUMP_GTID)
  {
    // get set of GTIDs
    Sid_map sid_map(NULL/*no lock needed*/);
    Gtid_set gtid_executed(&sid_map);
    global_sid_lock->wrlock();
    gtid_state->dbug_print();

    if (gtid_executed.add_gtid_set(mi->rli->get_gtid_set()) != RETURN_STATUS_OK ||
        gtid_executed.add_gtid_set(gtid_state->get_executed_gtids()) !=
        RETURN_STATUS_OK)
    {
      global_sid_lock->unlock();
      goto err;
    }
    global_sid_lock->unlock();
     
    // allocate buffer
    size_t encoded_data_size= gtid_executed.get_encoded_length();
    size_t allocation_size= 
      ::BINLOG_FLAGS_INFO_SIZE + ::BINLOG_SERVER_ID_INFO_SIZE +
      ::BINLOG_NAME_SIZE_INFO_SIZE + BINLOG_NAME_INFO_SIZE +
      ::BINLOG_POS_INFO_SIZE + ::BINLOG_DATA_SIZE_INFO_SIZE +
      encoded_data_size + 1;
    if (!(command_buffer= (uchar *) my_malloc(key_memory_rpl_slave_command_buffer,
                                              allocation_size, MYF(MY_WME))))
      goto err;
    uchar* ptr_buffer= command_buffer;

    DBUG_PRINT("info", ("Do I know something about the master? (binary log's name %s - auto position %d).",
               mi->get_master_log_name(), mi->is_auto_position()));
    /*
      Note: binlog_flags is always 0.  However, in versions up to 5.6
      RC, the master would check the lowest bit and do something
      unexpected if it was set; in early versions of 5.6 it would also
      use the two next bits.  Therefore, for backward compatibility,
      if we ever start to use the flags, we should leave the three
      lowest bits unused.
    */
    int2store(ptr_buffer, binlog_flags);
    ptr_buffer+= ::BINLOG_FLAGS_INFO_SIZE;
    int4store(ptr_buffer, server_id);
    ptr_buffer+= ::BINLOG_SERVER_ID_INFO_SIZE;
    int4store(ptr_buffer, static_cast<uint32>(BINLOG_NAME_INFO_SIZE));
    ptr_buffer+= ::BINLOG_NAME_SIZE_INFO_SIZE;
    memset(ptr_buffer, 0, BINLOG_NAME_INFO_SIZE);
    ptr_buffer+= BINLOG_NAME_INFO_SIZE;
    int8store(ptr_buffer, 4LL);
    ptr_buffer+= ::BINLOG_POS_INFO_SIZE;

    int4store(ptr_buffer, static_cast<uint32>(encoded_data_size));
    ptr_buffer+= ::BINLOG_DATA_SIZE_INFO_SIZE;
    gtid_executed.encode(ptr_buffer);
    ptr_buffer+= encoded_data_size;

    command_size= ptr_buffer - command_buffer;
    DBUG_ASSERT(command_size == (allocation_size - 1));
  }
  else
  {
    size_t allocation_size= ::BINLOG_POS_OLD_INFO_SIZE +
      BINLOG_NAME_INFO_SIZE + ::BINLOG_FLAGS_INFO_SIZE +
      ::BINLOG_SERVER_ID_INFO_SIZE + 1;
    if (!(command_buffer= (uchar *) my_malloc(key_memory_rpl_slave_command_buffer,
                                              allocation_size, MYF(MY_WME))))
      goto err;
    uchar* ptr_buffer= command_buffer;
  
    int4store(ptr_buffer, DBUG_EVALUATE_IF("request_master_log_pos_3", 3,
                                           static_cast<uint32>(mi->get_master_log_pos())));
    ptr_buffer+= ::BINLOG_POS_OLD_INFO_SIZE;
    // See comment regarding binlog_flags above.
    int2store(ptr_buffer, binlog_flags);
    ptr_buffer+= ::BINLOG_FLAGS_INFO_SIZE;
    int4store(ptr_buffer, server_id);
    ptr_buffer+= ::BINLOG_SERVER_ID_INFO_SIZE;
    memcpy(ptr_buffer, mi->get_master_log_name(), BINLOG_NAME_INFO_SIZE);
    ptr_buffer+= BINLOG_NAME_INFO_SIZE;

    command_size= ptr_buffer - command_buffer;
    DBUG_ASSERT(command_size == (allocation_size - 1));
  }

  if (simple_command(mysql, command, command_buffer, command_size, 1))
  {
    /*
      Something went wrong, so we will just reconnect and retry later
      in the future, we should do a better error analysis, but for
      now we just fill up the error log :-)
    */
    if (mysql_errno(mysql) == ER_NET_READ_INTERRUPTED)
      *suppress_warnings= true;                 // Suppress reconnect warning
    else
      sql_print_error("Error on %s: %d  %s, will retry in %d secs",
                      command_name[command].str,
                      mysql_errno(mysql), mysql_error(mysql),
                      mi->connect_retry);
    goto err;
  }
  error= 0;

err:
  my_free(command_buffer);
  DBUG_RETURN(error);
}


#sql/rpl_gtid_set.cc
gtidset管理
