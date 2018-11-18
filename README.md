## Getting Started

### Running etcd3

Ensure you have an etcd3 instance running locally. If not, you can start one with Docker using

```bash
docker run -it --rm -p 2379:2379 quay.io/coreos/etcd
```

### Start a work allocator

Start a work allocator instance using one the 
[various Spring Boot application approaches](https://docs.spring.io/spring-boot/docs/current/reference/html/getting-started-first-application.html#getting-started-first-application-run). 

For example,
```bash
mvn spring-boot:run
```

You will need to locate the HTTP port assigned to the instance by locating the log line like

```text
Tomcat started on port(s): 62930 (http)
```

The examples below will assume you assign a shell variable called `port` with that value, such as
```bash
port=62930
```

### Add some work

Post some new work definitions to the system using a "POST" such as the following with curl. We'll
add four work definitions to make things interesting:

```bash
curl -XPOST -d "testing=one" localhost:$port/work
curl -XPOST -d "testing=two" localhost:$port/work
curl -XPOST -d "testing=three" localhost:$port/work
curl -XPOST -d "testing=four" localhost:$port/work
```

With those you will see the one allocator instance picked up all the work:
```text
m.i.t.services.WorkAllocator             : Handling potential readyWork=9994249a-0ecb-4735-b078-19ce5c4ee20c at transition=NEW
m.i.t.services.WorkAllocator             : I'm the only worker
m.i.t.services.WorkAllocator             : I am least loaded, so I'll try to grab work=9994249a-0ecb-4735-b078-19ce5c4ee20c
m.i.t.services.WorkAllocator             : Successfully grabbed work=9994249a-0ecb-4735-b078-19ce5c4ee20c
m.i.t.services.WorkAllocator             : Stored workLoad=1 update
m.i.t.services.DefaultWorkProcessor      : Starting work on id=9994249a-0ecb-4735-b078-19ce5c4ee20c, content=testing=one
m.i.t.services.WorkAllocator             : Handling potential readyWork=13de78a9-a1e4-4e9f-9d71-49cb368240fe at transition=NEW
m.i.t.services.WorkAllocator             : I'm the only worker
m.i.t.services.WorkAllocator             : I am least loaded, so I'll try to grab work=13de78a9-a1e4-4e9f-9d71-49cb368240fe
m.i.t.services.WorkAllocator             : Successfully grabbed work=13de78a9-a1e4-4e9f-9d71-49cb368240fe
m.i.t.services.WorkAllocator             : Stored workLoad=2 update
m.i.t.services.DefaultWorkProcessor      : Starting work on id=13de78a9-a1e4-4e9f-9d71-49cb368240fe, content=testing=two
m.i.t.services.WorkAllocator             : Handling potential readyWork=c6574ad8-7a3f-48e2-8c33-7fdedef6d20e at transition=NEW
m.i.t.services.WorkAllocator             : I'm the only worker
m.i.t.services.WorkAllocator             : I am least loaded, so I'll try to grab work=c6574ad8-7a3f-48e2-8c33-7fdedef6d20e
m.i.t.services.WorkAllocator             : Successfully grabbed work=c6574ad8-7a3f-48e2-8c33-7fdedef6d20e
m.i.t.services.WorkAllocator             : Stored workLoad=3 update
m.i.t.services.DefaultWorkProcessor      : Starting work on id=c6574ad8-7a3f-48e2-8c33-7fdedef6d20e, content=testing=three
m.i.t.services.WorkAllocator             : Handling potential readyWork=dff30574-fca6-45a0-a0dd-db36142b1e8e at transition=NEW
m.i.t.services.WorkAllocator             : I'm the only worker
m.i.t.services.WorkAllocator             : I am least loaded, so I'll try to grab work=dff30574-fca6-45a0-a0dd-db36142b1e8e
m.i.t.services.WorkAllocator             : Successfully grabbed work=dff30574-fca6-45a0-a0dd-db36142b1e8e
m.i.t.services.WorkAllocator             : Stored workLoad=4 update
m.i.t.services.DefaultWorkProcessor      : Starting work on id=dff30574-fca6-45a0-a0dd-db36142b1e8e, content=testing=four
```

Using `etcdctl get --endpoints http://localhost:2479 --prefix /work/` we can also confirm the 
state of etcd after the work allocations (line space added for clarity):
```text
/work/active/13de78a9-a1e4-4e9f-9d71-49cb368240fe
81ffc456-79e4-4273-a684-3d3dc473f139
/work/active/9994249a-0ecb-4735-b078-19ce5c4ee20c
81ffc456-79e4-4273-a684-3d3dc473f139
/work/active/c6574ad8-7a3f-48e2-8c33-7fdedef6d20e
81ffc456-79e4-4273-a684-3d3dc473f139
/work/active/dff30574-fca6-45a0-a0dd-db36142b1e8e
81ffc456-79e4-4273-a684-3d3dc473f139

/work/registry/13de78a9-a1e4-4e9f-9d71-49cb368240fe
testing=two
/work/registry/9994249a-0ecb-4735-b078-19ce5c4ee20c
testing=one
/work/registry/c6574ad8-7a3f-48e2-8c33-7fdedef6d20e
testing=three
/work/registry/dff30574-fca6-45a0-a0dd-db36142b1e8e
testing=four

/work/workers/81ffc456-79e4-4273-a684-3d3dc473f139
0000000004
```

Notice under the default prefix of `/work/` there are three keysets that the allocator uses for
tracking and coordinating amongst the allocator instances:

- **workers**
  - One for each allocator/worker
  - Value contains the current work load of that worker
  - Each key is tied to the worker's lease and will be auto-removed when the worker leaves the system
- **registry**
  - One for each work item that needs to be worked
  - Value contains the content given when created/updated
- **active**
  - One for each work item that is actively assigned
  - Value contains the ID of the worker assigned
  - Each key is tied to the worker's lease and will be auto-removed when the worker leaves the system

### Start some more work allocators

If you start two more work allocator instances, you can see that the first one sheds some of its
work load to ensure the second and then third allocators/workers have their fair share.

Looking at the first allocator's logs:
```text
m.i.t.services.WorkAllocator             : Saw new worker=8b1da48a-7088-498a-9ae6-6245cdc870b1
m.i.t.services.WorkAllocator             : Rebalancing work load
m.i.t.services.WorkAllocator             : Shedding work to rebalance count=2
m.i.t.services.WorkAllocator             : Releasing work=dff30574-fca6-45a0-a0dd-db36142b1e8e
m.i.t.services.WorkAllocator             : Releasing work=c6574ad8-7a3f-48e2-8c33-7fdedef6d20e
m.i.t.services.DefaultWorkProcessor      : Stopping work on id=dff30574-fca6-45a0-a0dd-db36142b1e8e, content=testing=four
m.i.t.services.DefaultWorkProcessor      : Stopping work on id=c6574ad8-7a3f-48e2-8c33-7fdedef6d20e, content=testing=three
m.i.t.services.WorkAllocator             : Handling potential readyWork=dff30574-fca6-45a0-a0dd-db36142b1e8e at transition=RELEASED
m.i.t.services.WorkAllocator             : Handling potential readyWork=c6574ad8-7a3f-48e2-8c33-7fdedef6d20e at transition=RELEASED
m.i.t.services.WorkAllocator             : Stored workLoad=2 update
m.i.t.services.WorkAllocator             : Stored workLoad=3 update
m.i.t.services.WorkAllocator             : I am leastLoaded=false out of workerCount=2
m.i.t.services.WorkAllocator             : I am leastLoaded=false out of workerCount=2
m.i.t.services.WorkAllocator             : Saw new worker=fa69632a-023b-44db-93a8-173994fe936a
m.i.t.services.WorkAllocator             : Rebalancing work load
m.i.t.services.WorkAllocator             : Shedding work to rebalance count=1
m.i.t.services.WorkAllocator             : Releasing work=13de78a9-a1e4-4e9f-9d71-49cb368240fe
m.i.t.services.DefaultWorkProcessor      : Stopping work on id=13de78a9-a1e4-4e9f-9d71-49cb368240fe, content=testing=two
m.i.t.services.WorkAllocator             : Handling potential readyWork=13de78a9-a1e4-4e9f-9d71-49cb368240fe at transition=RELEASED
m.i.t.services.WorkAllocator             : Stored workLoad=1 update
m.i.t.services.WorkAllocator             : I am leastLoaded=false out of workerCount=3
m.i.t.services.WorkAllocator             : Handling potential readyWork=c6574ad8-7a3f-48e2-8c33-7fdedef6d20e at transition=RELEASED
m.i.t.services.WorkAllocator             : I am leastLoaded=false out of workerCount=3
```

Keep in mind there was some churn as the third allocator entered the system.

Using the same `etcdctl` command, we can see the work load is even balanced across the three workers:
```text
/work/active/13de78a9-a1e4-4e9f-9d71-49cb368240fe
fa69632a-023b-44db-93a8-173994fe936a
/work/active/9994249a-0ecb-4735-b078-19ce5c4ee20c
81ffc456-79e4-4273-a684-3d3dc473f139
/work/active/c6574ad8-7a3f-48e2-8c33-7fdedef6d20e
fa69632a-023b-44db-93a8-173994fe936a
/work/active/dff30574-fca6-45a0-a0dd-db36142b1e8e
8b1da48a-7088-498a-9ae6-6245cdc870b1

/work/registry/13de78a9-a1e4-4e9f-9d71-49cb368240fe
testing=two
/work/registry/9994249a-0ecb-4735-b078-19ce5c4ee20c
testing=one
/work/registry/c6574ad8-7a3f-48e2-8c33-7fdedef6d20e
testing=three
/work/registry/dff30574-fca6-45a0-a0dd-db36142b1e8e
testing=four

/work/workers/81ffc456-79e4-4273-a684-3d3dc473f139
0000000001
/work/workers/8b1da48a-7088-498a-9ae6-6245cdc870b1
0000000001
/work/workers/fa69632a-023b-44db-93a8-173994fe936a
0000000002
```

### Stop an allocator

Stop one of the allocators, ideally one with only one work item to keep the example interesting.
You can locate the ID of an allocator from the start of its logs, such as

```text
We are worker=81ffc456-79e4-4273-a684-3d3dc473f139
```

Looking at the other allocator with one work item, you'll see it correctly picked up the released
work item since it is the least loaded allocator:

```text
m.i.t.services.WorkAllocator             : Handling potential readyWork=9994249a-0ecb-4735-b078-19ce5c4ee20c at transition=RELEASED
m.i.t.services.WorkAllocator             : I am leastLoaded=true out of workerCount=2
m.i.t.services.WorkAllocator             : I am least loaded, so I'll try to grab work=9994249a-0ecb-4735-b078-19ce5c4ee20c
m.i.t.services.WorkAllocator             : Successfully grabbed work=9994249a-0ecb-4735-b078-19ce5c4ee20c
m.i.t.services.WorkAllocator             : Stored workLoad=2 update
m.i.t.services.DefaultWorkProcessor      : Starting work on id=9994249a-0ecb-4735-b078-19ce5c4ee20c, content=testing=one
```

Looking again with `etcdctl` we can see the work is spread across the now two workers:

```text
/work/active/13de78a9-a1e4-4e9f-9d71-49cb368240fe
fa69632a-023b-44db-93a8-173994fe936a
/work/active/9994249a-0ecb-4735-b078-19ce5c4ee20c
8b1da48a-7088-498a-9ae6-6245cdc870b1
/work/active/c6574ad8-7a3f-48e2-8c33-7fdedef6d20e
fa69632a-023b-44db-93a8-173994fe936a
/work/active/dff30574-fca6-45a0-a0dd-db36142b1e8e
8b1da48a-7088-498a-9ae6-6245cdc870b1

/work/registry/13de78a9-a1e4-4e9f-9d71-49cb368240fe
testing=two
/work/registry/9994249a-0ecb-4735-b078-19ce5c4ee20c
testing=one
/work/registry/c6574ad8-7a3f-48e2-8c33-7fdedef6d20e
testing=three
/work/registry/dff30574-fca6-45a0-a0dd-db36142b1e8e
testing=four

/work/workers/8b1da48a-7088-498a-9ae6-6245cdc870b1
0000000002
/work/workers/fa69632a-023b-44db-93a8-173994fe936a
0000000002
```

### Update a work item

Not so interesting, but still important in a real system, is the ability to update an existing 
work item.

Assuming the shell variable `id` has been set to the UUID of a work item in the registry
the following `PUT` will update that work item's content. **NOTE** you might need to
update `port`, if the original instance was stopped.

```bash
curl -H "Content-Type: text/plain" -X PUT -d "testing=twotwo" localhost:$port/work/$id
```

The previously assigned worker shows the update was observed and applied:
```text
m.i.t.services.WorkAllocator             : Updated our work=13de78a9-a1e4-4e9f-9d71-49cb368240fe
m.i.t.services.DefaultWorkProcessor      : Updating work on id=13de78a9-a1e4-4e9f-9d71-49cb368240fe, content=testing=twotwo
```

### Delete some work

With the shell variable `id` set to one of the work items in the registry, we can start deleting
off work using:

```bash
curl -X DELETE localhost:$port/work/$id
```

The one assigned that work item processes the deletion, but also coordinates indirectly with
the collective workers to rebalanace:

```text
m.i.t.services.WorkAllocator             : Stopping our work=13de78a9-a1e4-4e9f-9d71-49cb368240fe
m.i.t.services.DefaultWorkProcessor      : Stopping work on id=13de78a9-a1e4-4e9f-9d71-49cb368240fe, content=testing=twotwo
m.i.t.services.WorkAllocator             : Handling potential readyWork=9994249a-0ecb-4735-b078-19ce5c4ee20c at transition=RELEASED
m.i.t.services.WorkAllocator             : I am leastLoaded=false out of workerCount=2
m.i.t.services.WorkAllocator             : Handling potential readyWork=13de78a9-a1e4-4e9f-9d71-49cb368240fe at transition=RELEASED
m.i.t.services.WorkAllocator             : Removed active work=13de78a9-a1e4-4e9f-9d71-49cb368240fe key
m.i.t.services.WorkAllocator             : I am leastLoaded=false out of workerCount=2
m.i.t.services.WorkAllocator             : Stored workLoad=1 update
```

We can confirm the deletion with `etcdctl`:
```text
/work/active/9994249a-0ecb-4735-b078-19ce5c4ee20c
8b1da48a-7088-498a-9ae6-6245cdc870b1
/work/active/c6574ad8-7a3f-48e2-8c33-7fdedef6d20e
fa69632a-023b-44db-93a8-173994fe936a
/work/active/dff30574-fca6-45a0-a0dd-db36142b1e8e
8b1da48a-7088-498a-9ae6-6245cdc870b1

/work/registry/9994249a-0ecb-4735-b078-19ce5c4ee20c
testing=one
/work/registry/c6574ad8-7a3f-48e2-8c33-7fdedef6d20e
testing=three
/work/registry/dff30574-fca6-45a0-a0dd-db36142b1e8e
testing=four

/work/workers/8b1da48a-7088-498a-9ae6-6245cdc870b1
0000000002
/work/workers/fa69632a-023b-44db-93a8-173994fe936a
0000000001
```

For fun, let's delete off that one work item from worker `fa69632a-023b-44db-93a8-173994fe936a`,
which is work item `c6574ad8-7a3f-48e2-8c33-7fdedef6d20e`, looking at the `active` keys.

```bash
id=c6574ad8-7a3f-48e2-8c33-7fdedef6d20e
curl -X DELETE localhost:$port/work/$id
```

That allocator released the deleted work, but because the other allocator initiated a
rebalance we picked up one of the remaining two items to keep the allocations in balance:

```text
m.i.t.services.WorkAllocator             : Stopping our work=c6574ad8-7a3f-48e2-8c33-7fdedef6d20e
m.i.t.services.DefaultWorkProcessor      : Stopping work on id=c6574ad8-7a3f-48e2-8c33-7fdedef6d20e, content=testing=three
m.i.t.services.WorkAllocator             : Removed active work=c6574ad8-7a3f-48e2-8c33-7fdedef6d20e key
m.i.t.services.WorkAllocator             : Handling potential readyWork=c6574ad8-7a3f-48e2-8c33-7fdedef6d20e at transition=RELEASED
m.i.t.services.WorkAllocator             : I am leastLoaded=true out of workerCount=2
m.i.t.services.WorkAllocator             : I am least loaded, so I'll try to grab work=c6574ad8-7a3f-48e2-8c33-7fdedef6d20e
m.i.t.services.WorkAllocator             : Stored workLoad=0 update
m.i.t.services.WorkAllocator             : Handling potential readyWork=9994249a-0ecb-4735-b078-19ce5c4ee20c at transition=RELEASED
m.i.t.services.WorkAllocator             : I am leastLoaded=true out of workerCount=2
m.i.t.services.WorkAllocator             : I am least loaded, so I'll try to grab work=9994249a-0ecb-4735-b078-19ce5c4ee20c
m.i.t.services.WorkAllocator             : Successfully grabbed work=9994249a-0ecb-4735-b078-19ce5c4ee20c
m.i.t.services.WorkAllocator             : Stored workLoad=1 update
m.i.t.services.DefaultWorkProcessor      : Starting work on id=9994249a-0ecb-4735-b078-19ce5c4ee20c, content=testing=one
```

Finally, with `etcdctl` we can confirm each allocator has each of the two remaining work items:
```text
/work/active/9994249a-0ecb-4735-b078-19ce5c4ee20c
fa69632a-023b-44db-93a8-173994fe936a
/work/active/dff30574-fca6-45a0-a0dd-db36142b1e8e
8b1da48a-7088-498a-9ae6-6245cdc870b1

/work/registry/9994249a-0ecb-4735-b078-19ce5c4ee20c
testing=one
/work/registry/dff30574-fca6-45a0-a0dd-db36142b1e8e
testing=four

/work/workers/8b1da48a-7088-498a-9ae6-6245cdc870b1
0000000001
/work/workers/fa69632a-023b-44db-93a8-173994fe936a
0000000001
```