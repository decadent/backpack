# backpack

Ultimately fast storage for billions of files with http interface.

Backpack not a database, it's an append-only mostly-reads storage. It's a good photo storage if you want to build your own Facebook, Imgur or Tumblr.

## How it works

This project is inspired by Haystack from Facebook, which is not currently open-source.
There's a whitepaper about Haystack: [Finding a needle in Haystack: Facebook’s photo storage](http://static.usenix.org/event/osdi10/tech/full_papers/Beaver.pdf)

When you need to save billions of small files on the filesystem and access them fast, you will need
to make many seeks on physical disk to get files. If you don't care about permissions
for files and other metadata you'll have a huge overhead.

Backpack stores all metadata in memory and only needs one read per file. It groups small
files into big ones and always keeps them open to return data much faster than a usual filesystem.
You also get much better space utilisation for free, because there's no need to store
useless metadata. Note that backpack does not overwrite files, it's not a replacement
for a file system. Store only data that won't change if you don't want to waste disk space.

Backpack also has metadata for every data file so you can restore your data if
redis fails and loses some parts of data. However, disks can fail so we recommend
to save every piece of you data on different machines or even in different data centers.

## Production usage

We use it in [Topface](http://topface.com/) as storage for more than 100 million photos.
Backpack instances are managed by [Coordinators](https://github.com/Topface/backpack-coordinator)
and organized into shards to provide high availability and better performance.

## Benchmarks

Let's take 1 750 000 real photos from Topface and compare backpack and nginx. We'll use
two identical servers with 1Tb SATA disks (WDC WD1003FBYX-01Y7B1). Nginx is high-performance
web server so it should be fair competition. We'll save files in nginx with scheme
`xx/yy/u123_abc.jpg` to keep directory size relatively small.

### Writing

Backpack: 175 240 megabytes on disk, 1h41m to write.

Nginx: 177 483 megabytes on disk, 2h13m to write.

Result: 23% faster, just a bit smaller size on disk. But that's not the case, actually.

### Reading

In real world you cannot read files sequentially. People do random things on internet and
request random files. We cannot put files in order people will read them, so we'll just
pick 100 000 random files to read (same for nginx and backpack). To be fair, we'll drop
all page cache on linux with `echo 3 > /proc/sys/vm/drop_caches`.

It's better to see on graphs how it looks like (backpack is red, nginx is green).

* Requests per second.

![requests per second](http://i.imgur.com/1R0Kvld.png)

Backpack finished after 792 seconds, nginx after 1200 seconds. 33% faster!
Here you may see that nginx is getting faster but it has it's limits.

* Reads per second (from `iostat -x -d 10`).

![reads per second](http://i.imgur.com/kKUzCWy.png)

Here you may see the reason why nginx is slower: there are too many seeks.
Nginx needs to open directories and fetch file metadata that isn't in cache.
Note that you can't cache everything you need if you store more data
that in this test. Real servers hold way more data.

Backpack only reads the data and does not seek too much.

* Disk io utilization (from `iostat -x -d 10`).

Both disks are used by 100% while reads are active.

![disk io utilization](http://i.imgur.com/aePjesO.png)

Now imagine that you have not 1 750 000 files, but 22 000 000 files on a single server.
Extra seek for no reason will choke your system under load. That is the main reason
why we came up with backpack.

## Dependencies

* [Redis](http://redis.io/) — saves metadata about stored files

## Running the server

1. Install and run a Redis server. In this example, Redis listens on `127.0.0.1:6379`.

2. Decide where you want to save the files. Let's assume you want to store the files in `/var/backpack`.

3. Add some storage capacity to your Backpack:

    ```
    # each run of this command will add
    # approximately 3.5Gb of storage capacity
    ./bin/backpack-add /var/backpack 127.0.0.1 6379
    ```

4. Run the Backpack server. Let's say you want it to listen on `127.0.0.1:8080`:

    ```
    ./bin/backpack /var/backpack 127.0.0.1 8080 127.0.0.1 6379
    ```

Now try uploading a file to the storage:

```
./bin/backpack-upload 127.0.0.1 8080 /etc/hosts my/hosts
```

And download the file you just uploaded:

```
wget http://127.0.0.1:8080/my/hosts
```

## API

* Make a PUT request to save a file at specified location (the query string is taken into account).
* Make a GET request to retrieve a saved file.
* GET /stats to see some stats about what is happening.

## Stats

You may ask Backpack for stats by hitting the `/stats` url. The response is like this:

```javascript
{
    "gets": {
        "count" : 273852, // the total number of GET requests
        "bytes" : 66603359793, // the total size of responses in bytes
        "avg"   : 17.317889781518243 // the average response time in ms for the last 1000 GET requests
    },
    "puts":{
        "count" : 5604, // the total  of PUT requests
        "bytes" : 1616002993, // the size of written data in bytes
        "avg"   : 4.842664467849093 // the average response time in ms for last 1000 PUT requests
    }
}
```

## Utilities

* `./bin/backpack-add` - Add data files to the storage to extend capacity.

* `./bin/backpack-upload` - Upload a file to the storage.

* `./bin/backpack-dump-log` - Dump lengths and offsets of a specified storage file.

* `./bin/backpack-list` - List data files with their sizes and read-only flags.

* `./bin/backpack-get-file-info` - Get data file number, offset and length by the file name.

* `./bin/backpack-restore-from-log` - Restore the Redis index from the data file log.

## Authors

* [Ian Babrou](https://github.com/bobrik)
