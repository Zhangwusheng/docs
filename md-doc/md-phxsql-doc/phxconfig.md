cat /root/phxsql/etc/phxsqlproxy.conf
[Server]
SlaveForkProcCnt = 1
LogLevel = 3
IP = 10.199.201.143
MasterWorkerThread = 30
SlaveIORoutineCnt = 100
ProxyProtocol = 2
MasterForkProcCnt = 1
MasterEnableReadPort = 1
LogFilePath = /data/phxsql/log
Port = 54321
MasterIORoutineCnt = 100
SlaveWorkerThread = 30
LogMaxSize = 1600



~~~cpp

void PHXSqlProxyConfig::ReadConfig() {
    svr_port_ = GetInteger("Server", "Port", 0);
    svr_ip_ = Get("Server", "IP", "");
    if (svr_ip_ == "$InnerIP") {
        svr_ip_ = GetInnerIP();
    }
    phxsql_plugin_config_ = Get("Server", "PluginConfigFile", "");
    is_open_debug_mode_ = GetInteger("Server", "OpenDebugMode", 0);
    is_only_proxy_ = GetInteger("Server", "OnlyProxy", 0);
    is_master_enable_read_port_ = GetInteger("Server", "MasterEnableReadPort", 0);
    is_enable_try_best_ = GetInteger("Server", "TryBestIfBinlogsvrDead", 0);
    freqctrl_config_ = Get("Server", "FreqCtrlConfig", "");
    log_level_ = GetInteger("Server", "LogLevel", 3);
    log_file_max_size_ = GetInteger("Server", "LogFileMaxSize", 1600);
    log_path_ = Get("Server", "LogFilePath", "/tmp/");
    sleep_ = GetInteger("Server", "Sleep", 0);
    connect_timeout_ms_ = GetInteger("Server", "ConnectTimeoutMs", 200);
    write_timeout_ms_ = GetInteger("Server", "WriteTimeoutMs", 1000);
    proxy_protocol_ = GetInteger("Server", "ProxyProtocol", 0);
    proxy_protocol_timeout_ms_ = GetInteger("Server", "ProxyProtocolTimeoutMs", 1000);

    ReadMasterWorkerConfig(&master_worker_config_);
    ReadSlaveWorkerConfig(&slave_worker_config_);
    LogWarning("read plugin config [%s]", phxsql_plugin_config_.c_str());
}



void PHXSqlProxyConfig::ReadMasterWorkerConfig(WorkerConfig_t * worker_config) {
    worker_config->listen_ip_ = svr_ip_.c_str();
    worker_config->port_ = svr_port_;
    worker_config->proxy_port_ = GetInteger("Server", "MasterProxyPort", 0);
    worker_config->fork_proc_count_ = GetInteger("Server", "MasterForkProcCnt", 1);
    worker_config->worker_thread_count_ = GetInteger("Server", "MasterWorkerThread", 3);
    worker_config->io_routine_count_ = GetInteger("Server", "MasterIORoutineCnt", 1000);
    if (!worker_config->proxy_port_) {
        worker_config->proxy_port_ = worker_config->port_ + 2;
    }
    worker_config->is_master_port_ = true;
}

void PHXSqlProxyConfig::ReadSlaveWorkerConfig(WorkerConfig_t * worker_config) {
    worker_config->listen_ip_ = svr_ip_.c_str();
    worker_config->port_ = GetInteger("Server", "SlavePort", 0);
    worker_config->proxy_port_ = GetInteger("Server", "SlaveProxyPort", 0);
    worker_config->fork_proc_count_ = GetInteger("Server", "SlaveForkProcCnt", 1);
    worker_config->worker_thread_count_ = GetInteger("Server", "SlaveWorkerThread", 3);
    worker_config->io_routine_count_ = GetInteger("Server", "SlaveIORoutineCnt", 1000);
    if (!worker_config->port_) {
        worker_config->port_ = svr_port_ + 1;
    }
    if (!worker_config->proxy_port_) {
        worker_config->proxy_port_ = worker_config->port_ + 2;
    }
    worker_config->is_master_port_ = false;
}
~~~
