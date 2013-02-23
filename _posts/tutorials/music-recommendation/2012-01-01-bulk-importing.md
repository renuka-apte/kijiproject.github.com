---
layout: post
title : Bulk Importing
categories: [tutorials, music-recommendation, 1.0.0-rc4]
tags: [music]
order : 3
description: Bulk importing data into a Kiji table.
---

In cases where there is a significant amount of existing data to load into a Kiji
table, it hardly makes sense to do it a row at a time. We will show you how to use MapReduce to efficiently import
such large amounts of data into Kiji.

<div id="accordion-container">
  <h2 class="accordion-header"> SongMetadataBulkImporter.java </h2>
    <div class="accordion-content">
    <script src="http://gist-it.appspot.com/github/kijiproject/kiji-music/raw/master/src/main/java/org/kiji/examples/music/bulkimport/SongMetadataBulkImporter.java"> </script>
  </div>
  <h2 class="accordion-header"> JSONBulkImporter.java </h2>
    <div class="accordion-content">
    <script src="http://gist-it.appspot.com/github/kijiproject/kiji-mapreduce-lib/raw/master/kiji-mapreduce-lib/src/main/java/org/kiji/mapreduce/lib/bulkimport/JSONBulkImporter.java"> </script>
  </div>
</div>

<h3 style="margin-top:0px;padding-top:10px;">Custom Bulk Importers</h3>

One of the ways to bulk import your data is to extend `KijiBulkImporter` and override its `produce()` method
to insert rows in a distributed manner into the Kiji table. In the example below, we use this method to populate the song
metadata.

Input files contain JSON data representing song metadata, with one song per line. Below is the whitespace-augmented
example of a single row in our input file `song-metadata.json`.

{% highlight js %}
{
    "song_id" : "0",
    "song_name" : "song0",
    "artist_name" : "artist1",
    "album_name" : "album1",
    "genre" : "awesome",
    "tempo" : "140",
    "duration" : "180"
}
{% endhighlight %}

The `SongMetadataBulkImporter` class extends `KijiBulkImporter`. It expects a
[text input format]({{site.userguide_mapreduce_rc4}}/command-line-tools/#input) where the
input keys are the byte offsets of each line in the input file and the input values are the lines
of text described above.

In the `produce()` method of the class, extract the JSON as follows:
{% highlight java %}
// Parse JSON:
final JSONObject json = (JSONObject) parser.parse(line.toString());

// Extract JSON fields:
final String songId = json.get("song_id").toString();
{% endhighlight %}

Use an Avro record called SongMetaData described below:
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

Then build an Avro metadata record from the parsed JSON.

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

We create an [`EntityId`]({{site.api_schema_rc4}}/EntityId.html) object in order to use the song ID as the row key.
{% highlight java %}
final EntityId eid = context.getEntityId(songId);
{% endhighlight %}

Finally, write this Avro record to a cell in our Kiji table with the song ID as the row key.
{% highlight java %}
context.put(eid, "info", "metadata", song);
{% endhighlight %}

*As an aside, take care while using explicit timestamps when writing to Kiji. You can read about
[common pitfalls of timestamps in HBase](http://www.kiji.org/2013/02/13/common-pitfalls-of-timestamps-in-hbase/) on the Kiji blog
for more details.*

### Running the Example

Run the bulk import tool by specifying `SongMetadataBulkImporter` as the importer, the Kiji table `songs` as the output, and `song-metadata.json` as the input with the following command:

<div class="userinput">
{% highlight bash %}
$ kiji bulk-import \
    --importer=org.kiji.examples.music.bulkimport.SongMetadataBulkImporter \
    --lib=${LIBS_DIR} \
    --output="format=kiji table=${KIJI}/songs nsplits=1" \
    --input="format=text file=kiji-mr-tutorial/song-metadata.json"
{% endhighlight %}
</div>

#### Verify

Verify that the `user` table records were added properly by executing:

<div class="userinput">
{% highlight bash %}
$ kiji ls --kiji=${KIJI}/songs --max-rows=3
{% endhighlight %}
</div>

Here's what the first three entries should look like (assuming you're using the pregenerated song data).

    entity-id='song-32' [1361561116668] info:metadata
                                 {"song_name": "song name-32", "artist_name": "artist-2", "album_name": "album-0", "genre": "genre4.0", "tempo": 130, "duration": 120}

    entity-id='song-49' [1361561116737] info:metadata
                                 {"song_name": "song name-49", "artist_name": "artist-3", "album_name": "album-1", "genre": "genre7.0", "tempo": 80, "duration": 240}

    entity-id='song-36' [1361561116684] info:metadata
                                 {"song_name": "song name-36", "artist_name": "artist-2", "album_name": "album-0", "genre": "genre4.0", "tempo": 170, "duration": 120}

### Bulk importing using table import descriptors

In the example below, we use an import descriptor to bulk import our history of song plays from the `song-plays.json` into the
`user` table. This method of bulk import requires a table import descriptor, which is a JSON file containing:

<ul>
<li>The table that is the destination of the import.</li>
<li>The table column families.</li>
<ul>
<li>The name of the destination column.</li>
<li>The name of the source field to import from.</li>
</ul>
<li>The source for the entity ID.</li>
<li>An optional timestamp to use instead of system timestamp.</li>
<li>The format version of the import descriptor.</li>
</ul>

The import descriptor used for the `user` table is shown below:

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

We then use the pre-written `JSONBulkImporter` which expects a JSON file. Each line in this file
represents a separate JSON object to be imported into a row. The JSON object is described by an
import descriptor such as the one above. Target columns whose sources are not present in the JSON
object are skipped.

This descriptor parametrizes a special MapReduce job, where every row of the input file is parsed,
and inserted into the `users` Kiji table. The value of `user_id` will
be used as the row key in the Kiji table, the timestamp will be retrieved from the `play_time`
field. The value of `song_id` will be extracted and inserted into the `info:track_plays` column.

### Running the Example

Copy the descriptor file into HDFS.

<div class="userinput">
{% highlight bash %}
$ hadoop fs -copyFromLocal \
    $KIJI_HOME/examples/music/import/song-plays-import-descriptor.json \
    kiji-mr-tutorial/
{% endhighlight %}
</div>

Run the bulk import tool by specifying `JSONBulkImporter` as the importer, the Kiji table `users` as the output, and `song-plays.json` as the input with the following command:

<div class="userinput">
{% highlight bash %}
$ kiji bulk-import \
    -Dkiji.import.text.input.descriptor.path=kiji-mr-tutorial/song-plays-import-descriptor.json \
    --importer=org.kiji.mapreduce.lib.bulkimport.JSONBulkImporter \
    --output="format=kiji table=${KIJI}/users nsplits=1" \
    --input="format=text file=kiji-mr-tutorial/song-plays.json" \
    --lib=${LIBS_DIR}
{% endhighlight %}
</div>

#### Verify

Verify that the `user` table records were added properly by executing:

<div class="userinput">
{% highlight bash %}
$ kiji ls --kiji=${KIJI}/users --max-rows=3
{% endhighlight %}
</div>

Here’s what the first three entries should look like:

    entity-id='user-41' [1325750820000] info:track_plays
                                 song-43

    entity-id='user-3' [1325756880000] info:track_plays
                                 song-1

    entity-id='user-13' [1325752080000] info:track_plays
                                 song-23

