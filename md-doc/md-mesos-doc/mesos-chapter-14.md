##Resource(s)
关于资源的理解和操作


~~~cpp
/**
 * Describes a resource on a machine. The `name` field is a string
 * like "cpus" or "mem" that indicates which kind of resource this is;
 * the rest of the fields describe the properties of the resource. A
 * resource can take on one of three types: scalar (double), a list of
 * finite and discrete ranges (e.g., [1-10, 20-30]), or a set of
 * items. A resource is described using the standard protocol buffer
 * "union" trick.
 *
 * Note that "disk" and "mem" resources are scalar values expressed in
 * megabytes. Fractional "cpus" values are allowed (e.g., "0.5"),
 * which correspond to partial shares of a CPU.
 */
message Resource {
  required string name = 1;
  required Value.Type type = 2;
  optional Value.Scalar scalar = 3;
  optional Value.Ranges ranges = 4;
  optional Value.Set set = 5;

  // The role that this resource is reserved for. If "*", this indicates
  // that the resource is unreserved. Otherwise, the resource will only
  // be offered to frameworks that belong to this role.
  optional string role = 6 [default = "*"];

  message ReservationInfo {
    // Describes a dynamic reservation. A dynamic reservation is
    // acquired by an operator via the '/reserve' HTTP endpoint or by
    // a framework via the offer cycle by sending back an
    // 'Offer::Operation::Reserve' message.
    // NOTE: We currently do not allow frameworks with role "*" to
    // make dynamic reservations.

    // Indicates the principal, if any, of the framework or operator
    // that reserved this resource. If reserved by a framework, the
    // field should match the `FrameworkInfo.principal`. It is used in
    // conjunction with the `UnreserveResources` ACL to determine
    // whether the entity attempting to unreserve this resource is
    // permitted to do so.
    optional string principal = 1;

    // Labels are free-form key value pairs that can be used to
    // associate arbitrary metadata with a reserved resource.  For
    // example, frameworks can use labels to identify the intended
    // purpose for a portion of the resources the framework has
    // reserved at a given slave. Labels should not contain duplicate
    // key-value pairs.
    optional Labels labels = 2;
  }

  // If this is set, this resource was dynamically reserved by an
  // operator or a framework. Otherwise, this resource is either unreserved
  // or statically reserved by an operator via the --resources flag.
  optional ReservationInfo reservation = 8;

  message DiskInfo {
    // Describes a persistent disk volume.
    // A persistent disk volume will not be automatically garbage
    // collected if the task/executor/slave terminates, but is
    // re-offered to the framework(s) belonging to the 'role'.
    // A framework can set the ID (if it is not set yet) to express
    // the intention to create a new persistent disk volume from a
    // regular disk resource. To reuse a previously created volume, a
    // framework can launch a task/executor when it receives an offer
    // with a persistent volume, i.e., ID is set.
    // NOTE: Currently, we do not allow a persistent disk volume
    // without a reservation (i.e., 'role' should not be '*').
    message Persistence {
      // A unique ID for the persistent disk volume.
      // NOTE: The ID needs to be unique per role on each slave.
      required string id = 1;

      // This field indicates the principal of the operator or
      // framework that created this volume. It is used in conjunction
      // with the "destroy" ACL to determine whether an entity
      // attempting to destroy the volume is permitted to do so.
      //
      // NOTE: This field is optional, while the `principal` found in
      // `ReservationInfo` is required. This field is optional to
      // allow for the possibility of volume creation without a
      // principal, though currently it must be provided.
      //
      // NOTE: This field should match the FrameworkInfo.principal of
      // the framework that created the volume.
      optional string principal = 2;
    }

    optional Persistence persistence = 1;

    // Describes how this disk resource will be mounted in the
    // container. If not set, the disk resource will be used as the
    // sandbox. Otherwise, it will be mounted according to the
    // 'container_path' inside 'volume'. The 'host_path' inside
    // 'volume' is ignored.
    // NOTE: If 'volume' is set but 'persistence' is not set, the
    // volume will be automatically garbage collected after
    // task/executor terminates. Currently, if 'persistence' is set,
    // 'volume' must be set.
    optional Volume volume = 2;

    // Describes where a disk originates from.
    // TODO(jmlvanre): Add support for BLOCK devices.
    message Source {
      enum Type {
        PATH = 1;
        MOUNT = 2;
      }

      // A folder that can be located on a separate disk device. This
      // can be shared and carved up as necessary between frameworks.
      message Path {
        // Path to the folder (e.g., /mnt/raid/disk0).
        required string root = 1;
      }

      // A mounted file-system set up by the Agent administrator. This
      // can only be used exclusively: a framework can not accept a
      // partial amount of this disk.
      message Mount {
        // Path to mount point (e.g., /mnt/raid/disk0).
        required string root = 1;
      }

      required Type type = 1;
      optional Path path = 2;
      optional Mount mount = 3;
    }

    optional Source source = 3;
  }

  optional DiskInfo disk = 7;

  message RevocableInfo {}

  // If this is set, the resources are revocable, i.e., any tasks or
  // executors launched using these resources could get preempted or
  // throttled at any time. This could be used by frameworks to run
  // best effort tasks that do not need strict uptime or performance
  // guarantees. Note that if this is set, 'disk' or 'reservation'
  // cannot be set.
  optional RevocableInfo revocable = 9;
}


~~~


下面是两个辅助函数，解析JSON格式或者KV格式的资源字符串。

~~~cpp
Resources类可以理解为自己实现了PB的一个Resource的数组。
无他。记住这句话：
Resource objects stored in the class are always valid and
kept combined if possible. It is the caller's responsibility to
validate any Resource object or repeated Resource protobufs before constructing a Resources object


/////////////////////////////////////////////////
// Public static functions.
/////////////////////////////////////////////////

Try<Resource> Resources::parse(
    const string& name,
    const string& value,
    const string& role)
{
  Try<Value> result = internal::values::parse(value);
  if (result.isError()) {
    return Error(
        "Failed to parse resource " + name +
        " value " + value + " error " + result.error());
  }

  Resource resource;

  Value _value = result.get();
  resource.set_name(name);
  resource.set_role(role);

  if (_value.type() == Value::SCALAR) {
    resource.set_type(Value::SCALAR);
    resource.mutable_scalar()->CopyFrom(_value.scalar());
  } else if (_value.type() == Value::RANGES) {
    resource.set_type(Value::RANGES);
    resource.mutable_ranges()->CopyFrom(_value.ranges());
  } else if (_value.type() == Value::SET) {
    resource.set_type(Value::SET);
    resource.mutable_set()->CopyFrom(_value.set());
  } else {
    return Error(
        "Bad type for resource " + name + " value " + value +
        " type " + Value::Type_Name(_value.type()));
  }

  return resource;
}


// TODO(wickman) It is possible for Resources::ostream<< to produce
// unparseable resources, i.e.  those with
// ReservationInfo/DiskInfo/RevocableInfo.
Try<Resources> Resources::parse(
    const string& text,
    const string& defaultRole)
{
  Resources result;

  // Try to parse as a JSON Array.
  Try<JSON::Array> resourcesJSON = JSON::parse<JSON::Array>(text);
  if (resourcesJSON.isSome()) {
    Try<Resources> resources =
      internal::convertJSON(resourcesJSON.get(), defaultRole);
    if (resources.isError()) {
      return resources;
    }

    result = resources.get();
  } else {
    VLOG(1) << "Parsing resources as JSON failed: " << text << "\n"
            << "Trying semicolon-delimited string format instead";

    foreach (const string& token, strings::tokenize(text, ";")) {
      vector<string> pair = strings::tokenize(token, ":");
      if (pair.size() != 2) {
        return Error(
            "Bad value for resources, missing or extra ':' in " + token);
      }

      string name;
      string role;
      size_t openParen = pair[0].find("(");
      if (openParen == string::npos) {
        name = strings::trim(pair[0]);
        role = defaultRole;
      } else {
        size_t closeParen = pair[0].find(")");
        if (closeParen == string::npos || closeParen < openParen) {
          return Error(
              "Bad value for resources, mismatched parentheses in " + token);
        }

        name = strings::trim(pair[0].substr(0, openParen));

        role = strings::trim(pair[0].substr(
            openParen + 1,
            closeParen - openParen - 1));
      }

      Try<Resource> resource = Resources::parse(name, pair[1], role);
      if (resource.isError()) {
        return Error(resource.error());
      }

      result += resource.get();
    }
  }

  // TODO(jmlvanre): Move this up into `Containerizer::resources`.
  Option<Error> error = internal::validateCommandLineResources(result);
  if (error.isSome()) {
    return error.get();
  }

  return result;
}
~~~


~~~
判断两个Resource是不是相等

bool operator==(
    const Resource::ReservationInfo& left,
    const Resource::ReservationInfo& right)
{
  if (left.has_principal() != right.has_principal()) {
    return false;
  }

  if (left.has_principal() && left.principal() != right.principal()) {
    return false;
  }

  if (left.has_labels() != right.has_labels()) {
    return false;
  }

  if (left.has_labels() && left.labels() != right.labels()) {
    return false;
  }

  return true;
}

看看他是怎么判断两个资源是不是相互包含的。

// Tests if "right" is contained in "left".
static bool contains(const Resource& left, const Resource& right)
{
  // NOTE: This is a necessary condition for 'contains'.
  // 'subtractable' will verify name, role, type, ReservationInfo,
  // DiskInfo and RevocableInfo compatibility.
  if (!subtractable(left, right)) {
    return false;
  }

  if (left.type() == Value::SCALAR) {
    return right.scalar() <= left.scalar();
  } else if (left.type() == Value::RANGES) {
    return right.ranges() <= left.ranges();
  } else if (left.type() == Value::SET) {
    return right.set() <= left.set();
  } else {
    return false;
  }
}


这两个函数可以看一下，他对于保留资源的定义

bool Resources::isReserved(
    const Resource& resource,
    const Option<string>& role)
{
  if (role.isSome()) {
    return !isUnreserved(resource) && role.get() == resource.role();
  } else {
    return !isUnreserved(resource);
  }
}


bool Resources::isUnreserved(const Resource& resource)
{
  return resource.role() == "*" && !resource.has_reservation();
}

bool Resources::isDynamicallyReserved(const Resource& resource)
{
  return resource.has_reservation();
}

这几个函数比较好玩，有助于理解资源的定义

bool Resources::contains(const Resources& that) const
{
  Resources remaining = *this;

  foreach (const Resource& resource, that.resources) {
    // NOTE: We use _contains because Resources only contain valid
    // Resource objects, and we don't want the performance hit of the
    // validity check.
    if (!remaining._contains(resource)) {
      return false;
    }

    remaining -= resource;
  }

  return true;
}


bool Resources::contains(const Resource& that) const
{
  // NOTE: We must validate 'that' because invalid resources can lead
  // to false positives here (e.g., "cpus:-1" will return true). This
  // is because 'contains' assumes resources are valid.
  return validate(that).isNone() && _contains(that);
}


Resources Resources::filter(
    const lambda::function<bool(const Resource&)>& predicate) const
{
  Resources result;
  foreach (const Resource& resource, resources) {
    if (predicate(resource)) {
      result += resource;
    }
  }
  return result;
}


hashmap<string, Resources> Resources::reserved() const
{
  hashmap<string, Resources> result;

  foreach (const Resource& resource, resources) {
    if (isReserved(resource)) {
      result[resource.role()] += resource;
    }
  }

  return result;
}


Resources Resources::reserved(const string& role) const
{
  return filter(lambda::bind(isReserved, lambda::_1, role));
}


Resources Resources::unreserved() const
{
  return filter(isUnreserved);
}


Resources Resources::persistentVolumes() const
{
  return filter(isPersistentVolume);
}


Resources Resources::revocable() const
{
  return filter(isRevocable);
}


Resources Resources::nonRevocable() const
{
  return filter(
      [](const Resource& resource) { return !isRevocable(resource); });
}

~~~


重点看看这个函数，这个在自己写的Scheduler里面会用得到的。另外还实现了关于资源的 +-算法，不在详述。

~~~cpp

Resources Resources::flatten(
    const string& role,
    const Option<Resource::ReservationInfo>& reservation) const
{
  Resources flattened;

  foreach (Resource resource, resources) {
    resource.set_role(role);
    if (reservation.isNone()) {
      resource.clear_reservation();
    } else {
      resource.mutable_reservation()->CopyFrom(reservation.get());
    }
    flattened += resource;
  }

  return flattened;
}

~~~


~~~cpp
一些操作符重载

/////////////////////////////////////////////////
// Overloaded operators.
/////////////////////////////////////////////////

Resources::operator const RepeatedPtrField<Resource>&() const
{
  return resources;
}


bool Resources::operator==(const Resources& that) const
{
  return this->contains(that) && that.contains(*this);
}


bool Resources::operator!=(const Resources& that) const
{
  return !(*this == that);
}


Resources Resources::operator+(const Resource& that) const
{
  Resources result = *this;
  result += that;
  return result;
}


Resources Resources::operator+(const Resources& that) const
{
  Resources result = *this;
  result += that;
  return result;
}


Resources& Resources::operator+=(const Resource& that)
{
  if (validate(that).isNone() && !isEmpty(that)) {
    bool found = false;
    foreach (Resource& resource, resources) {
      if (internal::addable(resource, that)) {
        resource += that;
        found = true;
        break;
      }
    }

    // Cannot be combined with any existing Resource object.
    if (!found) {
      resources.Add()->CopyFrom(that);
    }
  }

  return *this;
}


Resources& Resources::operator+=(const Resources& that)
{
  foreach (const Resource& resource, that.resources) {
    *this += resource;
  }

  return *this;
}


Resources Resources::operator-(const Resource& that) const
{
  Resources result = *this;
  result -= that;
  return result;
}


Resources Resources::operator-(const Resources& that) const
{
  Resources result = *this;
  result -= that;
  return result;
}


Resources& Resources::operator-=(const Resource& that)
{
  if (validate(that).isNone() && !isEmpty(that)) {
    for (int i = 0; i < resources.size(); i++) {
      Resource* resource = resources.Mutable(i);

      if (internal::subtractable(*resource, that)) {
        *resource -= that;

        // Remove the resource if it becomes invalid or zero. We need
        // to do the validation because we want to strip negative
        // scalar Resource object.
        if (validate(*resource).isSome() || isEmpty(*resource)) {
          // As `resources` is not ordered, and erasing an element
          // from the middle using `DeleteSubrange` is expensive, we
          // swap with the last element and then shrink the
          // 'RepeatedPtrField' by one.
          resources.Mutable(i)->Swap(resources.Mutable(resources.size() - 1));
          resources.RemoveLast();
        }

        break;
      }
    }
  }

  return *this;
}


Resources& Resources::operator-=(const Resources& that)
{
  foreach (const Resource& resource, that.resources) {
    *this -= resource;
  }

  return *this;
}

~~~
