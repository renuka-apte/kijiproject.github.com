---
layout: post
title : Music Recommendation Producer
categories: [tutorials, music-recommendation, 1.0.0-rc4]
tags: [music]
order : 7
description: Read and write to the same row of a table.
---

<div id="accordion-container">
  <h2 class="accordion-header"> NextSongRecommender.java </h2>
    <div class="accordion-content">
    <script src="http://gist-it.appspot.com/github/kijiproject/kiji-music/raw/master/src/main/java/org/kiji/examples/music/produce/NextSongRecommender.java"> </script>
  </div>
</div>

<h3 style="margin-top:0px;padding-top:10px;"> NextSongRecommender.java </h3>
The NextSongRecommender is an example of a [KijiProducer]({{site.userguide_mapreduce_rc4}}/producers). A producer operates on
a single row of input data and generates new outputs that are written to the same row. For every row
this producer processes, it will:

* Read the most recent value from the "info:track_plays" column of the users table. This is the song
  id of the most recently played song by the user
* Lookup a list of the songs most frequently played next from the songs table.
* Generate a recommendation from the list of songs most frequently played next.
* Write that recommendation to the "info:next_song_rec" column of the users table.

#### Get The Most Recent Song Played
Like Gatherers, you specify the required columns for your producer in the getDataRequest method. We
only want the most recent value from this column, so we can use the create convenience method.
{% highlight java %}
  public KijiDataRequest getDataRequest() {
    // Only request the most recent version from the "info:track_plays" column.
    return KijiDataRequest.create("info", "track_plays");
  }
{% endhighlight %}

In our produce method, we then access our requested data through the [`KijiRowData`]({{site.api_schema_rc4}}/KijiRowData.html):

{% highlight java %}
  String mostRecentSong = input.<CharSequence>getMostRecentValue("info", "track_plays")
      .toString();// Avro strings get deserialized to CharSequences, so .toString() the result.
{% endhighlight %}

#### Join External Data Sources
[KeyValueStores]({{site.userguide_mapreduce_rc4}}/key-value-stores) allow you to access external data sources in a MapReduce job.
In this case, we will use the "top_next_songs" column of our songs table as a KeyValueStore. In
order to access KeyValueStores in a KijiMR Job, the class that needs the external data must
implement KeyValueStoreClient. This interface requires that you implement getRequiredStores().
The value that you must return from getRequiredStores is a map from the name of a KeyValueStore to
the default implementation.

For reasons pertaining to [KijiMR-91](https://jira.kiji.org/browse/KIJIMR-91) we leave our default
implementation unconfigured.

{% highlight java %}
  public Map<String, KeyValueStore<?, ?>> getRequiredStores() {
    return RequiredStores.just("nextPlayed", UnconfiguredKeyValueStore.builder().build());
  }
{% endhighlight %}

When we run this producer in a test, we will override the default implementation programmatically
using a job builder. When you run this producer from the command line, you will override the
default implementation using the KVStoreConfig.xml file.

#### Generate a Recommendation
To generate a recommendation from the list of songs that are most likely to be played next, we do
the simplest thing possible; choose the first element of the list.

{% highlight java %}
  private CharSequence recommend(List<SongCount> topNextSongs) {
    return topNextSongs.get(0).getSongId(); // Do the simplest possible thing.
  }
{% endhighlight %}

#### Write the Output to a Column
To write our recommendation to the table, we need to declare what column we are writing to.

{% highlight java %}
  public String getOutputColumn() {
    return "info:next_song_rec";
  }
{% endhighlight %}

Since the column is already declared, to write a value to it, we simply call context.put() with
the value we want to write as the parameter.

{% highlight java %}
  context.put(recommend(popularNextSongs));
{% endhighlight %}

<div id="accordion-container">
  <h2 class="accordion-header"> TestNextSongRecommender.java </h2>
    <div class="accordion-content">
    <script src="http://gist-it.appspot.com/github/kijiproject/kiji-music/raw/master/src/test/java/org/kiji/examples/music/TestNextSongRecommender.java"> </script>
  </div>
</div>

<h3 style="margin-top:0px;padding-top:10px;"> TestNextSongRecommender.java </h3>
To test NextSongRecommender, we need specify which KijiTable we want to use to back our
KeyValueStore. We do this by constructing the KeyValueStore we want to use, via the KeyValueStore's
builder method. We then override the KeyValueStore binding in this job configuration by using the
withStore() method of JobBuilders.

{% highlight java %}
  KijiTableKeyValueStore.Builder kvStoreBuilder = KijiTableKeyValueStore.builder();
  kvStoreBuilder.withColumn("info", "top_next_songs").withTable(mSongTableURI);

  // Configure first job.
  final MapReduceJob mrjob = KijiProduceJobBuilder.create()
      .withStore("nextPlayed", kvStoreBuilder.build())

  // ...
{% endhighlight %}

### Running the Example
When we run this example, we again need to need specify which [`KijiTable`]({{site.api_schema_rc4}}/KijiTable.html) we want to use to back our
KeyValueStore. This time, we will override the KeyValueStore binding from
the command line using an XML configuration file. The content of the file is displayed below.
If you are not using KijiBento, you may need to modify this XML file so that the URI points to the
songs table you would like to use.


{% highlight xml %}
<?xml version="1.0" encoding="UTF-8"?>
<stores>
  <store name="nextPlayed" class="org.kiji.mapreduce.kvstore.lib.KijiTableKeyValueStore">
    <configuration>
      <property>
        <name>table.uri</name>
        <!-- This URI can be replace with the URI of a different 'songs' table to use. -->
        <value>kiji://.env/kiji_music/songs</value>
      </property>
      <property>
        <name>column</name>
        <value>info:top_next_songs</value>
      </property>
    </configuration>
  </store>
</stores>
{% endhighlight %}

Now, run the command.

<div class="userinput">
{% highlight bash %}
kiji produce \
      --producer=org.kiji.examples.music.produce.NextSongRecommender \
      --input="format=kiji table=$KIJI/users" \
      --output="format=kiji table=$KIJI/users nsplits=2" \
      --lib=${LIBS_DIR} \
      --kvstores=$KIJI_HOME/examples/music/KVStoreConfig.xml
{% endhighlight %}
</div>

#### Verify

<div class="userinput">
{% highlight bash %}
kiji ls --kiji=$KIJI/users --columns=info:next_song_rec --max-rows=3
{% endhighlight %}

These are our recommendations for the next song to play for each user!
</div>

    entity-id='user-41' [1361564713968] info:next_song_rec
                                 song-41

    entity-id='user-3' [1361564713980] info:next_song_rec
                                 song-2

    entity-id='user-13' [1361564713990] info:next_song_rec
                                 song-27

