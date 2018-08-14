# DriverClass

net.opentsdb.tools.OpenTSDBMain



opentsdb2.4提供了很多相关的工具类，统一由此类进行功能分发。（2.3是独立的类，2.4移到了这个类进行统一管理）。

这个类包含的工具如下：



```java
static {
  Map<String, Class<?>> tmp = new HashMap<String, Class<?>>();
  tmp.put("fsck", Fsck.class);
  tmp.put("import", TextImporter.class);
  tmp.put("mkmetric", UidManager.class); // -> shift --> set uid assign metrics "$@"
  tmp.put("query", CliQuery.class);
  tmp.put("tsd", OpenTSDBMain.class);
  tmp.put("scan", DumpSeries.class);
  tmp.put("uid", UidManager.class);
  tmp.put("exportui", UIContentExporter.class);
  tmp.put("help", HelpProcessor.class);
  COMMANDS = Collections.unmodifiableMap(tmp);
}
```



```
private static void process(String targetTool, String[] args) {
  if("mkmetric".equals(targetTool)) {
    shift(args);
  } 
  if(!"tsd".equals(targetTool)) {     
    try {
      COMMANDS.get(targetTool).getDeclaredMethod("main", String[].class).invoke(null, new Object[] {args});
    } catch(Exception x) {
        log.error("Failed to call [" + targetTool + "].", x);
        System.exit(-1);
    }     
  } else {
    launchTSD(args);
  }
}
```

# mkmetric

metrics相关的功能由类UidManager进行管理

```java
/**
 * Command line tool to manipulate UIDs.
 * Can be used to find or assign UIDs.
 */
```
具体使用说明如下：

```
/** Prints usage. */
static void usage(final ArgP argp, final String errmsg) {
  System.err.println(errmsg);
  System.err.println("Usage: uid <subcommand> args\n"
      + "Sub commands:\n"
      + "  grep [kind] <RE>: Finds matching IDs.\n"
      + "  assign <kind> <name> [names]:"
      + " Assign an ID for the given name(s).\n"
      + "  rename <kind> <name> <newname>: Renames this UID.\n"
      + "  delete <kind> <name>: Deletes this UID.\n"
      + "  fsck: [fix] [delete_unknown] Checks the consistency of UIDs.\n"
      + "        fix            - Fix errors. By default errors are logged.\n"
      + "        delete_unknown - Remove columns with unknown qualifiers.\n"
      + "                         The \"fix\" flag must be supplied as well.\n"
      + "\n"
      + "  [kind] <name>: Lookup the ID of this name.\n"
      + "  [kind] <ID>: Lookup the name of this ID.\n"
      + "  metasync: Generates missing TSUID and UID meta entries, updates\n"
      + "            created timestamps\n"
      + "  metapurge: Removes meta data entries from the UID table\n"
      + "  treesync: Process all timeseries meta objects through tree rules\n"
      + "  treepurge <id> [definition]: Purge a tree and/or the branches\n"
      + "            from storage. Provide an integer Tree ID and optionally\n"
      + "            add \"true\" to delete the tree definition\n\n"
      + "Example values for [kind]:"
      + " metrics, tagk (tag name), tagv (tag value).");
  if (argp != null) {
    System.err.print(argp.usage());
  }
}


Usage: uid <subcommand> args
Sub commands:
  grep [kind] <RE>: Finds matching IDs.
  assign <kind> <name> [names]: Assign an ID for the given name(s).
  rename <kind> <name> <newname>: Renames this UID.
  delete <kind> <name>: Deletes this UID.
  fsck: [fix] [delete_unknown] Checks the consistency of UIDs.
        fix            - Fix errors. By default errors are logged.
        delete_unknown - Remove columns with unknown qualifiers.
                         The "fix" flag must be supplied as well.

  [kind] <name>: Lookup the ID of this name.
  [kind] <ID>: Lookup the name of this ID.
  metasync: Generates missing TSUID and UID meta entries, updates
            created timestamps
  metapurge: Removes meta data entries from the UID table
  treesync: Process all timeseries meta objects through tree rules
  treepurge <id> [definition]: Purge a tree and/or the branches
            from storage. Provide an integer Tree ID and optionally
            add "true" to delete the tree definition

Example values for [kind]: metrics, tagk (tag name), tagv (tag value).
```