# Kredis

Kredis (Keyed Redis) encapsulates higher-level types and data structures around a single key, so you can interact with them as coherent objects rather than isolated procedural commands. These higher-level structures can be configured as attributes within Active Models and Active Records using a declarative DSL.

Kredis is configured using env-aware YAML files, using `Rails.application.config_for`, so you can locate the data structures on separate Redis instances, if you've reached a scale where a single shared instance is no longer sufficient.

Kredis provides namespacing support for keys such that you can safely run parallel testing against the data structures without different tests trampling each others data.


## Examples

Kredis provides typed scalars for strings, integers, decimals, floats, booleans, datetimes, and JSON hashes:

```ruby
string = Kredis.string "mystring"
string.value = "hello world!"  # => SET mystring "hello world"
"hello world!" == string.value # => GET mystring

integer = Kredis.integer "myinteger"
integer.value = 5  # => SET myinteger "5"
5 == integer.value # => GET myinteger

decimal = Kredis.decimal "mydecimal" # accuracy!
decimal.value = "%.47f" % (1.0 / 10) # => SET mydecimal "0.10000000000000000555111512312578270211815834045"
BigDecimal("0.10000000000000000555111512312578270211815834045e0") == decimal.value # => GET mydecimal

float = Kredis.float "myfloat" # speed!
float.value = 1.0 / 10 # => SET myfloat "0.1"
0.1 == float.value # => GET myfloat

boolean = Kredis.boolean "myboolean"
boolean.value = true # => SET myboolean "t"
true == boolean.value # => GET myboolean

datetime = Kredis.datetime "mydatetime"
memoized_midnight = Time.zone.now.midnight
datetime.value = memoized_midnight # SET mydatetime "2021-07-27T00:00:00.000000000Z"
memoized_midnight == datetime.value # => GET mydatetime

json = Kredis.json "myjson"
json.value = { "one" => 1, "two" => "2" }  # => SET myjson "{\"one\":1,\"two\":\"2\"}"
{ "one" => 1, "two" => "2" } == json.value  # => GET myjson
```

There are data structures for counters, enums, flags, lists, unique lists, sets, and slots:

```ruby
list = Kredis.list "mylist"
list << "hello world!"               # => RPUSH mylist "hello world!"
[ "hello world!" ] == list.elements  # => LRANGE mylist 0, -1
list.clear                           # => DEL mylist

integer_list = Kredis.list "myintegerlist", typed: :integer, default: [ 1, 2, 3 ] # => EXISTS? myintegerlist, RPUSH myintegerlist "1" "2" "3"
integer_list.append([ 4, 5, 6 ])                                                  # => RPUSH myintegerlist "4" "5" "6"
integer_list << 7                                                                 # => RPUSH myintegerlist "7"
[ 1, 2, 3, 4, 5, 6, 7 ] == integer_list.elements                                  # => LRANGE myintegerlist 0 -1
integer_list.clear                                                                # => DEL myintegerlist

unique_list = Kredis.unique_list "myuniquelist"
unique_list.append(%w[ 2 3 4 ])                # => LREM myuniquelist 0, "2" + LREM myuniquelist 0, "3" + LREM myuniquelist 0, "4"  + RPUSH myuniquelist "2", "3", "4"
unique_list.prepend(%w[ 1 2 3 4 ])             # => LREM myuniquelist 0, "1"  + LREM myuniquelist 0, "2" + LREM myuniquelist 0, "3" + LREM myuniquelist 0, "4"  + LPUSH myuniquelist "1", "2", "3", "4"
unique_list.append([])
unique_list << "5"                             # => LREM myuniquelist 0, "5" + RPUSH myuniquelist "5"
unique_list.remove(3)                          # => LREM myuniquelist 0, "3"
[ "4", "2", "1", "5" ] == unique_list.elements # => LRANGE myuniquelist 0, -1
unique_list.clear                              # => DEL myuniquelist

ordered_set = Kredis.ordered_set "myorderedset"
ordered_set.append(%w[ 2 3 4 ])                # => ZADD myorderedset 1646131025.4953232 2 1646131025.495326 3 1646131025.4953272 4
ordered_set.prepend(%w[ 1 2 3 4 ])             # => ZADD myorderedset -1646131025.4957051 1 -1646131025.495707 2 -1646131025.4957082 3 -1646131025.4957092 4
ordered_set.append([])
ordered_set << "5"                             # => ZADD myorderedset 1646131025.4960442 5
ordered_set.remove(3)                          # => ZREM myorderedset 3
[ "4", "2", "1", "5" ] == ordered_set.elements # => ZRANGE myorderedset 0 -1

set = Kredis.set "myset", typed: :datetime
set.add(DateTime.tomorrow, DateTime.yesterday)           # => SADD myset "2021-02-03 00:00:00 +0100" "2021-02-01 00:00:00 +0100"
set << DateTime.tomorrow                                 # => SADD myset "2021-02-03 00:00:00 +0100"
2 == set.size                                            # => SCARD myset
[ DateTime.tomorrow, DateTime.yesterday ] == set.members # => SMEMBERS myset
set.clear                                                # => DEL myset

hash = Kredis.hash "myhash"
hash.update("key" => "value", "key2" => "value2")     # => HSET myhash "key", "value", "key2", "value2"
{ "key" => "value", "key2" => "value2" } == hash.to_h # => HGETALL myhash
"value2" == hash["key2"]                              # => HMGET myhash "key2"
%w[ key key2 ] == hash.keys                           # => HKEYS myhash
%w[ value value2 ] == hash.values                     # => HVALS myhash
hash.remove                                           # => DEL myhash

high_scores = Kredis.hash "high_scores", typed: :integer
high_scores.update(space_invaders: 100, pong: 42)             # HSET high_scores "space_invaders", "100", "pong", "42"
%w[ space_invaders pong ] == high_scores.keys                 # HKEYS high_scores
[ 100, 42 ] == high_scores.values                             # HVALS high_scores
{ "space_invaders" => 100, "pong" => 42 } == high_scores.to_h # HGETALL high_scores

head_count = Kredis.counter "headcount"
0 == head_count.value              # => GET "headcount"
head_count.increment               # => SET headcount 0 NX + INCRBY headcount 1
head_count.increment               # => SET headcount 0 NX + INCRBY headcount 1
head_count.decrement               # => SET headcount 0 NX + DECRBY headcount 1
1 == head_count.value              # => GET "headcount"

counter = Kredis.counter "mycounter", expires_in: 5.seconds
counter.increment by: 2         # => SET mycounter 0 EX 5 NX + INCRBY "mycounter" 2
2 == counter.value              # => GET "mycounter"
sleep 6.seconds
0 == counter.value              # => GET "mycounter"
counter.reset                   # => DEL mycounter

cycle = Kredis.cycle "mycycle", values: %i[ one two three ]
:one == cycle.value             # => GET mycycle
cycle.next                      # => GET mycycle + SET mycycle 1
:two == cycle.value             # => GET mycycle
cycle.next                      # => GET mycycle + SET mycycle 2
:three == cycle.value           # => GET mycycle
cycle.next                      # => GET mycycle + SET mycycle 0
:one == cycle.value             # => GET mycycle
cycle.reset                     # => DEL mycycle

enum = Kredis.enum "myenum", values: %w[ one two three ], default: "one"
"one" == enum.value             # => GET myenum
true == enum.one?               # => GET myenum
enum.value = "two"              # => SET myenum "two"
"two" == enum.value             # => GET myenum
enum.three!                     # => SET myenum "three"
"three" == enum.value           # => GET myenum
enum.value = "four"
"three" == enum.value           # => GET myenum
enum.reset                      # => DEL myenum
"one" == enum.value             # => GET myenum

slots = Kredis.slots "myslots", available: 3
true == slots.available?        # => GET myslots
slots.reserve                   # => INCR myslots
true == slots.available?        # => GET myslots
slots.reserve                   # => INCR myslots
true == slots.available?        # => GET myslots
slots.reserve                   # => INCR myslots
false == slots.available?       # => GET myslots
slots.reserve                   # => INCR myslots + DECR myslots
false == slots.available?       # => GET myslots
slots.release                   # => DECR myslots
true == slots.available?        # => GET myslots
slots.reset                     # => DEL myslots


slot = Kredis.slot "myslot"
true == slot.available?        # => GET myslot
slot.reserve                   # => INCR myslot
false == slot.available?       # => GET myslot
slot.release                   # => DECR myslot
true == slot.available?        # => GET myslot
slot.reset                     # => DEL myslot

flag = Kredis.flag "myflag"
false == flag.marked?           # => EXISTS myflag
flag.mark                       # => SET myflag 1
true == flag.marked?            # => EXISTS myflag
flag.remove                     # => DEL myflag
false == flag.marked?           # => EXISTS myflag

true == flag.mark(expires_in: 1.second, force: false)    #=> SET myflag 1 EX 1 NX
false == flag.mark(expires_in: 10.seconds, force: false) #=> SET myflag 10 EX 1 NX
true == flag.marked?            #=> EXISTS myflag
sleep 0.5.seconds
true == flag.marked?            #=> EXISTS myflag
sleep 0.6.seconds
false == flag.marked?           #=> EXISTS myflag

limiter = Kredis.limiter "mylimit", limit: 3, expires_in: 5.seconds
0 == limiter.value              # => GET "limiter"
limiter.poke                    # => SET limiter 0 NX + INCRBY limiter 1
limiter.poke                    # => SET limiter 0 NX + INCRBY limiter 1
false == limiter.exceeded?      # => GET "limiter"
limiter.poke                    # => SET limiter 0 NX + INCRBY limiter 1
true == limiter.exceeded?       # => GET "limiter"
sleep 6
limiter.poke                    # => SET limiter 0 NX + INCRBY limiter 1
limiter.poke                    # => SET limiter 0 NX + INCRBY limiter 1
false == limiter.exceeded?      # => GET "limiter"
```

### Models

You can use all these structures in models:

```ruby
class Person < ApplicationRecord
  kredis_list :names
  kredis_list :names_with_custom_key_via_lambda, key: ->(p) { "person:#{p.id}:names_customized" }
  kredis_list :names_with_custom_key_via_method, key: :generate_names_key
  kredis_unique_list :skills, limit: 2
  kredis_enum :morning, values: %w[ bright blue black ], default: "bright"
  kredis_counter :steps, expires_in: 1.hour

  private
    def generate_names_key
      "key-generated-from-private-method"
    end
end

person = Person.find(5)
person.names.append "David", "Heinemeier", "Hansson" # => RPUSH people:5:names "David" "Heinemeier" "Hansson"
true == person.morning.bright?                       # => GET people:5:morning
person.morning.value = "blue"                        # => SET people:5:morning
true == person.morning.blue?                         # => GET people:5:morning
```

### Default values

You can set a default value for all types. For example:

```ruby
list = Kredis.list "favorite_colors", default: [ "red", "green", "blue" ]

# or, in a model
class Person < ApplicationRecord
  kredis_string :name, default: "Unknown"
  kredis_list :favorite_colors, default: [ "red", "green", "blue" ]
end
```

There's a performance overhead to consider though. When you first read or write an attribute in a model, Kredis will
check if the underlying Redis key exists, while watching for concurrent changes, and if it does not,
write the specified default value.

This means that using default values in a typical Rails app additional Redis calls (WATCH, EXISTS, UNWATCH) will be
executed for each Kredis attribute with a default value read or written during a request.

### Callbacks

You can also define `after_change` callbacks that trigger on mutations:

```ruby
class Person < ApplicationRecord
  kredis_list :names, after_change: ->(p) {  }
  kredis_unique_list :skills, limit: 2, after_change: :skillset_changed

  def skillset_changed
  end
end
```

### Multiple Redis servers

And using structures on a different than the default `shared` redis instance, relying on `config/redis/secondary.yml`:

```ruby
one_string = Kredis.string "mystring"
two_string = Kredis.string "mystring", config: :secondary

one_string.value = "just on shared"
two_string.value != one_string.value
```

## Installation

1. Run `./bin/bundle add kredis`
2. Run `./bin/rails kredis:install` to add a default configuration at [`config/redis/shared.yml`](lib/install/shared.yml)

Additional configurations can be added under `config/redis/*.yml` and referenced when a type is created. For example, `Kredis.string("mystring", config: :strings)` would lookup `config/redis/strings.yml`.

Kredis passes the configuration to `Redis.new` to establish the connection. See the [Redis documentation](https://github.com/redis/redis-rb) for other configuration options.

If you don't have `config/redis/shared.yml` (or use another named configuration), Kredis will default to look in env for `REDIS_URL`, then fallback to a default URL of `redis://127.0.0.1:6379/0`.

### Redis support

Kredis works with Redis server 4.0+, with the [Redis Ruby](https://github.com/redis/redis-rb) client version 4.2+. Redis Cluster is not supported.

### Setting SSL options on Redis Connections

If you need to connect to Redis with SSL, the recommended approach is to set your Redis instance manually by adding an entry to the `Kredis::Connections.connections` hash. Below an example showing how to connect to Redis using Client Authentication:

```ruby
Kredis::Connections.connections[:shared] = Redis.new(
  url: ENV["REDIS_URL"],
  ssl_params: {
    cert_store: OpenSSL::X509::Store.new.tap { |store|
      store.add_file(Rails.root.join("config", "ca_cert.pem").to_s)
    },

    cert: OpenSSL::X509::Certificate.new(File.read(
      Rails.root.join("config", "client.crt")
    )),

    key: OpenSSL::PKey::RSA.new(
      Rails.application.credentials.redis[:client_key]
    ),

    verify_mode: OpenSSL::SSL::VERIFY_PEER
  }
)
```

The above code could be added to either `config/environments/production.rb` or an initializer. Please ensure that your client private key, if used, is stored your credentials file or another secure location.

### Configure how the redis client is created

You can configure how the redis client is created by setting `config.kredis.connector` in your `application.rb`:

```ruby
config.kredis.connector = ->(config) { SomeRedisProxy.new(config) }
```

By default Kredis will use `Redis.new(config)`.

## Development

A development console is available by running `bin/console`.

From there, you can experiment with Kredis. e.g.

```erb
>> str = Kredis.string "mystring"
  Kredis  (0.1ms)  Connected to shared
=>
#<Kredis::Types::Scalar:0x0000000134c7d938
...
>> str.value = "hello, world"
  Kredis Proxy (2.4ms)  SET mystring ["hello, world"]
=> "hello, world"
>> str.value
```

Run tests with `bin/test`.

[`debug`](https://github.com/ruby/debug) can be used in the development console and in the test suite by inserting a
breakpoint, e.g. `debugger`.

## License

Kredis is released under the [MIT License](https://opensource.org/licenses/MIT).
