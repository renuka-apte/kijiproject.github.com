---
layout: post
title : Bulk Importing
categories: [tutorials, music-recommendation, 1.0.0-rc4]
tags: [music]
order : 3
description: Bulk importing historic data into a Kiji table.
---

In cases where we have a significant amount of existing data that we'd like to load into a Kiji
table, it hardly makes sense to do it a row at a time. We show you how to import the song metadata
into the Kiji table all at once.

### Bulk Importing using Direct Writes

One of the ways to bulk import your data is to extend `KijiBulkImporter` and use its producer method
to insert rows into the kiji table. In the example below, we use this method to populate the song
metadata.

Input files contain JSON data representing song metadata, with one song per line (as shown below).

    {
        "song_id" : "0",
        "song_name" : "song0",
        "artist_name" : "artist1",
        "album_name" : "album1",
        "genre" : "awesome",
        "tempo" : "140",
        "duration" : "180"
    }

The `SongMetadataBulkImporter` class extends the `KijiBulkImporter`. It expects a text input format where
input keys are the positions (in bytes) of each line in input file and input values are the lines of text
describde above.

In the producer method of the class, we extract the json as follows:
{% highlight java %}
// Parse JSON:
final JSONObject json = (JSONObject) parser.parse(line.toString());

// Extract JSON fields:
final String songId = json.get("song_id").toString();
{% endhighlight %}

We use an avro record called SongMetaData described below:
{% highlight java %}
    record SongMetadata {
        string song_name;
        string artist_name;
        string album_name;
        string genre;
        long tempo;
        long duration;
    }
{% endhighlight %}

We then build the Avro metadata record from the parsed json.

{% highlight java %}
final SongMetadata song = SongMetadata.newBuilder()
      .setSongName(songName)
      .setAlbumName(albumName)
      .setArtistName(artistName)
      .setGenre(genre)
      .setTempo(tempo)
      .setDuration(duration)
      .build();
{% endhighlight %}

We create an EntityId object in order to use the song ID as the row key.
{% highlight java %}
final EntityId eid = context.getEntityId(songId);
{% endhighlight %}

Finally, we write this avro record to a cell in our kiji table with the song ID as the row key.
{% highlight java %}
context.put(eid, "info", "metadata", song);
{% endhighlight %}

*As an aside, care must be taken while using explicit timestamps while writing to Kiji. You can read 
[this post](http://www.kiji.org/2013/02/13/common-pitfalls-of-timestamps-in-hbase/) on the Kiji blog
for more details about this.*

### Running the Example

<div class="userinput">
{% highlight bash %}
$ kiji bulk-import \
    --importer=org.kiji.examples.music.bulkimport.SongMetadataBulkImporter \
    --lib=${LIBS_DIR} \
    --output="format=kiji table=${KIJI}/songs nsplits=1" \
    --input="format=text file=${HDFS_ROOT}/kiji-mr-tutorial/song-metadata.json"
{% endhighlight %}
</div>

#### Verify
<div class="userinput">
{% highlight bash %}
$ kiji ls --kiji=${KIJI}
{% endhighlight %}
</div>

    Listing tables in kiji instance: kiji://localhost:2181/kiji_music/
    songs
    users

### Bulk importing using table import descriptors

Another way to bulk import your data is by using import descriptors.
At the top-level, a table import descriptor contains:

 *   the table that is the destination of the import.
 *   the table column families.
 *   the source for the entity id.
 *   (optional) timestamp to use instead of system timestamp.
 *   format version of the import descriptor.

Each column family has:

 *   the name of the destination column.
 *   the name of the source field to import from.

In the example below, we use import descriptors to bulk import the song plays into the user
table. The import descriptor used for the user table is shown below:

{% highlight bash %}
{
  name : "users",
  families : [ {
    name : "info",
    columns : [ {
      name : "track_plays",
      source : "song_id"
    } ]
  } ],
  entityIdSource : "user_id",
  overrideTimestampSource : "play_time",
  version : "import-1.0"
}
{% endhighlight %}

We then use the pre-written `JSONBulkImporter` which expects a JSON file. Each line in this file represents a 
separate JSON object to be imported into a row. The JSON object is described by an import descriptor such as the one above.
Target columns whose sources are not present in the JSON object are skipped.

The above descriptor asks every row of the input file to be parsed and inserted into the `users` Kiji table. 
The value of `user_id` will be used as the row key in the
Kiji table. The timestamp will be retrieved from the `play_time` field. The value of `song_id` will be extracted and inserted
into the `info:track_plays` column.

### Running the Example

Copy the descriptor file into HDFS.

<div class="userinput">
{% highlight bash %}
$ hadoop fs -copyFromLocal \
    $KIJI_HOME/examples/music/import/song-plays-import-descriptor.json \
    ${HDFS_ROOT}/kiji-mr-tutorial/
{% endhighlight %}
</div>

Run the JSON bulk importer.

<div class="userinput">
{% highlight bash %}
$ kiji bulk-import \
    -Dkiji.import.text.input.descriptor.path=${HDFS_ROOT}/kiji-mr-tutorial/song-plays-import-descriptor.json \
    --importer=org.kiji.mapreduce.lib.bulkimport.JSONBulkImporter \
    --output="format=kiji table=${KIJI}/users nsplits=1" \
    --input="format=text file=$HDFS_ROOT/kiji-mr-tutorial/song-plays.json" \
    --lib=${LIBS_DIR}
{% endhighlight %}
</div>

