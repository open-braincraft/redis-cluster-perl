# NAME

    Redis::Cluster - Redis Cluster client for Perl

# SYNOPSYS

    use Redis::Cluster;

    my $cluster = Redis::Cluster->new(
      server         => [qw(
        127.0.0.1:7000 127.0.0.1:7001 127.0.0.1:7002
        127.0.0.1:7003 127.0.0.1:7004 127.0.0.1:7005
      )],
      refresh        => 0,
      max_redirects  => 5,
      max_queue_size => 10,
      allow_slave    => 0,
      ... # See Redis.pm for other arguments
    );

    $cluster->set('key', 1);
    my $res = $cluster->get('key');

    # See Redis.pm for API details

# DESCRIPTION

Redis Cluster is HA solution for Redis. This module deals with:

- Cluster state
- Connection pool
- Node selection (hash tags are supported)
- Key slot redirects
- Execution of read-only commands on slave nodes (optional)

Transactions (not recommended), Lua scripts (recommended) and 'wait' command
are supported with some limitations described in BUGS AND LIMITATIONS.

# MIGRATION

This module provides the same API as Redis.pm. So, migration should be quite
simple. There are two main differences:

- 'server' property should contain at least three nodes. It may be an ArrayRef
or comma separated string.
- You should use hash tags in your keys to avoid issues.

# PUBLIC METHODS

## new

    my $cluster = Redis::Cluster->new(
      server         => [qw(
        127.0.0.1:7000 127.0.0.1:7001 127.0.0.1:7002
        127.0.0.1:7003 127.0.0.1:7004 127.0.0.1:7005
      )],
      # Default values below
      refresh        => 0,
      max_redirects  => 5,
      max_queue_size => 10,
      allow_slave    => 0,
      ...
    );

Constructor.
Returns Redis::Cluster object.

### server

Cluster nodes (mandatory). Should be an ArrayRef or comma separated string.
Every node should be described as ip:port. You should specify at least
three (prefferably master) nodes. This nodes will be used to get cluster
state, other nodes will be found automatically.

You can also specify nodes in REDIS\_CLUSTER envitronment variable.

### refresh

Cluster state refresh interval (in seconds).
If set to zero, cluster state will be updated only on MOVED redirect.

### max\_redirects

Maximum number of key slot redirects (to avoid infinite redirects)

### max\_queue\_size

Maximum internal queue size (used in 'multi' mode)

### allow\_slave

Allow execution of read-only commands on slave nodes

### default\_slot

This key slot is used for execution of commands without keys in arguments.
If not specified, random key slot will be used, and command will be executed
on a random master node.

### Other arguments are identical to Redis.pm

## get\_master\_by\_key

    my $redis = $cluster->get_master_by_key($key);

Get master node by key. Returns Redis object.

## get\_slave\_by\_key

    $redis = $cluster->get_slave_by_key($key, $num);

Get slave node by key. Returns Redis object.

$num is a slave number (1 - first slave, ...).
If not specified, random slave node will be returned.

## get\_node\_by\_key

    $redis = $cluster->get_node_by_key($key, $num);

Get any node (master or slave) by key. Returns Redis object.

$num is a node number (1 - master, 2 - first slave, ...).
If not specified, random node will be returned.

## get\_node

    $redis = $cluster->get_node($node);

Get node by ip:port. Returns Redis object.
Node should be a member of cluster.

## get\_random\_master

    my $redis = $cluster->get_random_master();

Get random master node.

## get\_random\_slave

    my $redis = $cluster->get_random_slave();

Get random slave node.

# PRIVATE METHODS

## \_exec\_cmd

    my $res = $cluster->_exec_cmd($cmd, @args);

Execute Redis command. Returns command execution result.

## \_enqueue

    $cluster->_enqueue($cmd, @args);

Enqueue command into internal queue (used in 'multi' mode).

## \_set\_mode\_by\_cmd

    $cluster->_set_mode_by_cmd($cmd);

Set mode by command (multi, exec, discard, watch, unwatch).

## \_on\_error

    my $res = $cluster->_on_error($redis, $cmd, \@args, $err_msg);

Error handler. Retries command on another node on redirect.
Returns command execution result.

## \_get\_slot\_by\_key

    my $slot = $cluster->_get_slot_by_key($key);

Get slot by key. Returns slot number.

## \_get\_master\_by\_slot

    $redis = $cluster->_get_master_by_slot($slot);

Get master node by slot number. Returns Redis object.

## \_get\_slave\_by\_slot

    $redis = $cluster->_get_slave_by_slot($slot, $num);

Get slave node by slot number. Returns Redis object.

$num is a slave number (1 - first slave, ...).
If not specified, random slave node will be returned.

## \_get\_node\_by\_slot

    $redis = $cluster->_get_node_by_slot($slot, $num, $offset);

Get any node (master or slave) by slot number. Return Redis object.

$num is a node number (1 - master, 2 - first slave, ...).
If not specified, random node will be returned.

$offset is a node offset (0 - any node, 1 - any slave, ...).

## \_get\_range\_by\_slot

    my $range = $cluster->_get_range_by_slot($slot);

Get slot range by slot number.
Returns an item of 'cluster slots' command excution result (ArrayRef).

## \_get\_slots

    my $slots = $cluster->_get_slots($force);

Get slots (cluster state).
Returns 'cluster slots' command execution result (ArrayRef).

If $force flag is set, cluster state will be obtained immediately,
ignoring 'refresh' property.

## \_get\_node

    $redis = $cluster->_get_node($node);

Get node by ip:port. Returns Redis object.

## \_is\_member

    my $is_member = $cluster->_is_member($node);

Check if node is a member of cluster by ip:port. Returns boolean.

# BUGS AND LIMITATIONS

- Commands with keys in arguments will be executed on node, selected by
first key.  In 'watch' and/or 'multi' mode node will be selected by
first key of first command with key. All commands prior to first command
with key will be enqueued into internal queue. It means that the same
master node will be selected from 'watch' or 'multi' command till 'exec',
'discard' or 'unwatch'.

    All multi-key commands should be in a single key slot. It is a Redis
    Cluster limitation. You should use hash tags to avoid issues.

- 'Wait' command will be executed on node, selected by last known key.
- Redirects are not allowed inside 'multi' and/or 'watch'.
Using Lua scripts instead of transactions is highly recommended.
- Unlike Redis.pm, there is a fixed list of supported commands with keys
in arguments. It means there may be some issues with new commands until
they are supported by authors of this module.

# SEE ALSO

- Redis
- [http://redis.io/topics/cluster-tutorial](http://redis.io/topics/cluster-tutorial)
- [http://redis.io/topics/cluster-spec](http://redis.io/topics/cluster-spec)

# AUTHORS

- SMS Online <dev.opensource@sms-online.com>

# COPYRIGHT AND LICENSE

Copyright (C) 2015 SMS Online

This is free software, licensed under:
  The Artistic License 2.0 (GPL Compatible)
