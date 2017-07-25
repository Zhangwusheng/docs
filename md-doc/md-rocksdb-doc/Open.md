##DB::Open

```cpp
    Status DB::Open(const Options& options, const std::string& dbname, DB** dbptr) {
    DBOptions db_options(options);
    ColumnFamilyOptions cf_options(options);
    std::vector<ColumnFamilyDescriptor> column_families;
    column_families.push_back(
        ColumnFamilyDescriptor(kDefaultColumnFamilyName, cf_options));
    std::vector<ColumnFamilyHandle*> handles;
    Status s = DB::Open(db_options, dbname, column_families, &handles, dbptr);
    ...
    return s;
  }
```

首先声明了两个Option，一个是DBOptions，一个是ColumnFamilyOptions. 

DBOptions设置了Env::Default()，这个Env在linux下面是PosixEnv,其他都是一些数值设置。

ColumnFamilyOptions里面设置了memtable_factory为SkipListFactory，table_factory为BlockBasedTableFactory, 

BlockBasedTableFactory 里面设置了flush_block_policy_factory为FlushBlockBySizePolicyFactory。还设置了LRUCache。block_cache = NewLRUCache(8 << 20)

comparator设置成了BytewiseComparator,实际上是个BytewiseComparatorImpl静态对象。write_buffer_size设置成了67108864( 64<<20)。levels设置成了7.很多和leveldb相关的选项其实是在这个ColumnFamilyOptions里面。

这个Open会调用Open的重载函数，这里才是真正的打开数据库的逻辑：

~~~cpp

Status DB::Open(const DBOptions& db_options, const std::string& dbname,
                const std::vector<ColumnFamilyDescriptor>& column_families,
                std::vector<ColumnFamilyHandle*>* handles, DB** dbptr) {
                
  //Sanitize是审查的意思，就是检查这些配置是否符合要求。
  //这里调用的是BlockBasedTableFactory::SanitizeOptions函数，无他不详述。
  Status s = SanitizeOptionsByTable(db_options, column_families);
  if (!s.ok()) {
    return s;
  }

  //检查数据库相关选项，不详述

  s = ValidateOptions(db_options, column_families);
  if (!s.ok()) {
    return s;
  }

  *dbptr = nullptr;
  handles->clear();

 //max_write_buffer_size为列簇的最大缓冲区？
 
  size_t max_write_buffer_size = 0;
  for (auto cf : column_families) {
    max_write_buffer_size =
        std::max(max_write_buffer_size, cf.options.write_buffer_size);
  }

//重点研究DBImpl

 ~~~


##DBImpl
 

继续DB::Open
 
~~~cpp

  DBImpl* impl = new DBImpl(db_options, dbname);
  
  //创建wal_dir
  s = impl->env_->CreateDirIfMissing(impl->immutable_db_options_.wal_dir);
  if (s.ok()) {
    for (auto db_path : impl->immutable_db_options_.db_paths) {
    //创建dbpaths
      s = impl->env_->CreateDirIfMissing(db_path.path);
      if (!s.ok()) {
        break;
      }
    }
  }

  if (!s.ok()) {
    delete impl;
    return s;
  }

//创建ArchivalDirectory
  s = impl->CreateArchivalDirectory();
  if (!s.ok()) {
    delete impl;
    return s;
  }
  impl->mutex_.Lock();
  // Handles create_if_missing, error_if_exists
  s = impl->Recover(column_families);
  if (s.ok()) {
    uint64_t new_log_number = impl->versions_->NewFileNumber();
    unique_ptr<WritableFile> lfile;
    EnvOptions soptions(db_options);
    EnvOptions opt_env_options =
        impl->immutable_db_options_.env->OptimizeForLogWrite(
            soptions, BuildDBOptions(impl->immutable_db_options_,
                                     impl->mutable_db_options_));
    s = NewWritableFile(
        impl->immutable_db_options_.env,
        LogFileName(impl->immutable_db_options_.wal_dir, new_log_number),
        &lfile, opt_env_options);
    if (s.ok()) {
      lfile->SetPreallocationBlockSize(
          impl->GetWalPreallocateBlockSize(max_write_buffer_size));
      impl->logfile_number_ = new_log_number;
      unique_ptr<WritableFileWriter> file_writer(
          new WritableFileWriter(std::move(lfile), opt_env_options));
          
          //这里生成了Writer，注意看这里的数据格式：CRC+Size+Type+LogNumber+数据，并且以block为单位，block的大小是32768，32K
          //上面的Writer只是数据格式的定义，它封装的WritableFileWriter里面包含了缓冲区的管理。回家再看.参见：file_reader_writer.h和.cc
          
      impl->logs_.emplace_back(
          new_log_number,
          new log::Writer(
              std::move(file_writer), new_log_number,
              impl->immutable_db_options_.recycle_log_file_num > 0));

      // set column family handles
      for (auto cf : column_families) {
        auto cfd =
            impl->versions_->GetColumnFamilySet()->GetColumnFamily(cf.name);
        if (cfd != nullptr) {
          handles->push_back(
              new ColumnFamilyHandleImpl(cfd, impl, &impl->mutex_));
          impl->NewThreadStatusCfInfo(cfd);
        } else {
          if (db_options.create_missing_column_families) {
            // missing column family, create it
            ColumnFamilyHandle* handle;
            impl->mutex_.Unlock();
            s = impl->CreateColumnFamily(cf.options, cf.name, &handle);
            impl->mutex_.Lock();
            if (s.ok()) {
              handles->push_back(handle);
            } else {
              break;
            }
          } else {
            s = Status::InvalidArgument("Column family not found: ", cf.name);
            break;
          }
        }
      }
    }
    if (s.ok()) {
      for (auto cfd : *impl->versions_->GetColumnFamilySet()) {
        delete impl->InstallSuperVersionAndScheduleWork(
            cfd, nullptr, *cfd->GetLatestMutableCFOptions());
      }
      impl->alive_log_files_.push_back(
          DBImpl::LogFileNumberSize(impl->logfile_number_));
      impl->DeleteObsoleteFiles();
      s = impl->directories_.GetDbDir()->Fsync();
    }
  }

  if (s.ok()) {
    for (auto cfd : *impl->versions_->GetColumnFamilySet()) {
      if (cfd->ioptions()->compaction_style == kCompactionStyleFIFO) {
        auto* vstorage = cfd->current()->storage_info();
        for (int i = 1; i < vstorage->num_levels(); ++i) {
          int num_files = vstorage->NumLevelFiles(i);
          if (num_files > 0) {
            s = Status::InvalidArgument(
                "Not all files are at level 0. Cannot "
                "open with FIFO compaction style.");
            break;
          }
        }
      }
      if (!cfd->mem()->IsSnapshotSupported()) {
        impl->is_snapshot_supported_ = false;
      }
      if (cfd->ioptions()->merge_operator != nullptr &&
          !cfd->mem()->IsMergeOperatorSupported()) {
        s = Status::InvalidArgument(
            "The memtable of column family %s does not support merge operator "
            "its options.merge_operator is non-null", cfd->GetName().c_str());
      }
      if (!s.ok()) {
        break;
      }
    }
  }
  TEST_SYNC_POINT("DBImpl::Open:Opened");
  Status persist_options_status;
  if (s.ok()) {
    // Persist RocksDB Options before scheduling the compaction.
    // The WriteOptionsFile() will release and lock the mutex internally.
    persist_options_status = impl->WriteOptionsFile();

    *dbptr = impl;
    impl->opened_successfully_ = true;
    impl->MaybeScheduleFlushOrCompaction();
  }
  impl->mutex_.Unlock();

#ifndef ROCKSDB_LITE
  auto sfm = static_cast<SstFileManagerImpl*>(
      impl->immutable_db_options_.sst_file_manager.get());
  if (s.ok() && sfm) {
    // Notify SstFileManager about all sst files that already exist in
    // db_paths[0] when the DB is opened.
    auto& db_path = impl->immutable_db_options_.db_paths[0];
    std::vector<std::string> existing_files;
    impl->immutable_db_options_.env->GetChildren(db_path.path, &existing_files);
    for (auto& file_name : existing_files) {
      uint64_t file_number;
      FileType file_type;
      std::string file_path = db_path.path + "/" + file_name;
      if (ParseFileName(file_name, &file_number, &file_type) &&
          file_type == kTableFile) {
        sfm->OnAddFile(file_path);
      }
    }
  }
#endif  // !ROCKSDB_LITE

  if (s.ok()) {
    Log(InfoLogLevel::INFO_LEVEL, impl->immutable_db_options_.info_log,
        "DB pointer %p", impl);
    LogFlush(impl->immutable_db_options_.info_log);
    if (!persist_options_status.ok()) {
      if (db_options.fail_if_options_file_error) {
        s = Status::IOError(
            "DB::Open() failed --- Unable to persist Options file",
            persist_options_status.ToString());
      }
      Warn(impl->immutable_db_options_.info_log,
           "Unable to persist options in DB::Open() -- %s",
           persist_options_status.ToString().c_str());
    }
  }
  if (!s.ok()) {
    for (auto* h : *handles) {
      delete h;
    }
    handles->clear();
    delete impl;
    *dbptr = nullptr;
  }
  return s;
}
~~~



首先查看 WriteBufferManager，这个是内存跟踪管理类，主要的功能是记录分配了多少内存，是否需要flus.这个类很简单，但是有一句话，需要注意：

~~~cpp
class WriteBufferManager {
 public:
  // _buffer_size = 0 indicates no limit. Memory won't be tracked,
  // memory_usage() won't be valid and ShouldFlush() will always return true.
  explicit WriteBufferManager(size_t _buffer_size)
      : buffer_size_(_buffer_size), memory_used_(0) {}

  ~WriteBufferManager() {}

  bool enabled() const { return buffer_size_ != 0; }

  // Only valid if enabled()
  size_t memory_usage() const {
    return memory_used_.load(std::memory_order_relaxed);
  }
  size_t buffer_size() const { return buffer_size_; }
~~~

* Should only be called from write thread

~~~cpp
  // Should only be called from write thread
  bool ShouldFlush() const {
    return enabled() && memory_usage() >= buffer_size();
  }

  // Should only be called from write thread
  void ReserveMem(size_t mem) {
    if (enabled()) {
      memory_used_.fetch_add(mem, std::memory_order_relaxed);
    }
  }
  void FreeMem(size_t mem) {
    if (enabled()) {
      memory_used_.fetch_sub(mem, std::memory_order_relaxed);
    }
  }

 private:
  const size_t buffer_size_;
  std::atomic<size_t> memory_used_;

  // No copying allowed
  WriteBufferManager(const WriteBufferManager&) = delete;
  WriteBufferManager& operator=(const WriteBufferManager&) = delete;
};
~~~


