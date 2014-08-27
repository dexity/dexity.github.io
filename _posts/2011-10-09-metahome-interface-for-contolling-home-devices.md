---
layout: post
title: MetaHome&#58; Interface for Contolling Home Devices
tags: android, web service
image: metahome-interface/index.png
blogfeed: true
---

Controlling home devices from a single hand held device is a dream that will soon become real. Home devices are getting smarter and include more functionality than devices we have now. I developed a simple prototype of Android application that can manage virtual devices. The idea can be useful both for *Home Automation* to control devices as well as for *Smart Maps* to display detailed information about the objects on the map such as airports or shopping centers.

{{ more }}

> Code: [https://bitbucket.org/dexity/metahome][source-metahome]

## Application Description

The application manages predefined items (`Restroom`, `Telephone`, `Elevator`, `ATM`) by adding or removing them from display. The items can represent any object on the map: controllable home device or non-controllable object on the map. The application uses 3 independent data sources: `Memory`, `Local Database` and `Web Service`. I implemented different data sources to explore user experience. User can select any of the data sources from the Menu. For Web Service I wrote a simple PHP script and put it on remote web server which handles incoming requests from the application. Adding item to the map is as simple as clicking on the dashed square and selecting the item.

## Layout and Drawing

The layout consists of three main views: `MapView`, `TextView` and `ListView`.

{% highlight java linenos %}
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
   android:orientation="vertical"
   android:layout_width="match_parent"
   android:layout_height="match_parent">

    <com.surfingbits.metahome.MapView
       android:id="@+id/map_view"
       android:layout_width="wrap_content"
       android:layout_height="240dip"
       android:layout_margin="10dip"/>

    <TextView android:id="@+id/status"
       android:layout_width="wrap_content"
       android:layout_height="23dip"
       android:textSize="14dip"/>

    <ListView android:id="@android:id/list"
       android:layout_width="match_parent"
       android:layout_height="wrap_content"
       android:drawSelectorOnTop="false"/>

</LinearLayout>
{% endhighlight %}

Both `MapView` and `ListView` are different views of the same data model described in the next section. `TextView` displays the status of selected data source. When the screen is rotated the application will use a different layout: `layout-land/main.xml`

`MapView` performs custom drawing of items depending if an item is located in the square or not.

{% highlight java linenos %}
@Override
protected void onDraw(Canvas canvas)
{
    super.onDraw(canvas);

    float[] outerR = new float[] { 10, 10, 10, 10, 10, 10, 10, 10 };

    mShapeDrawable = new ShapeDrawable(new RoundRectShape(outerR, null, null));
    mShapeDrawable.getPaint().setColor(0xcccccccc);
    mShapeDrawable.setBounds(0, 0, MAPSIZE, MAPSIZE);
    mShapeDrawable.draw(canvas);

    for (int i = 0; i < ldata.size(); i++)
    {
        if (ldata.isEmpty(i))
        {
            drawPlace(canvas, i);
        }
        else {
            drawIcon(canvas, i);
        }
    }
}
{% endhighlight %}

## Data Model and Interface

Data model of the application is very simple. It is described by a list of size `4`. The index of the list represents cell index of flattened table: from left to right, top to bottom. The value of the list represents index of item which is placed in the cell. Empty cell has value `-1`.

Items indices:

    0 - Restroom
    1 - Telephone
    2 - Elevator
    3 - ATM

For example: `["0","-1","-1","2"]`

means that item `Restroom` (item index `0`) is in cell `(0, 0)` (cell index `0`) and `Elevator` (item index `2`) is in cell `(1, 1)` (cell index `3`).

To make operations on the cells independent on the data source I implemented a general interface. All manipulation with the data storage is performed through this interface. If you want to add a new data storage you need to add also implementation of the interface.

{% highlight java linenos %}
public interface IMapData {
    abstract void set(int index, int value);
    abstract int get(int index);
    abstract int[] getAll();
    abstract void remove(int index);
    abstract boolean isEmpty(int index);
    abstract int size();
}
{% endhighlight %}

Cells list is then translated to `ImageText` class which is used by views to display image and corresponding label.

{% highlight java linenos %}
class ImageText
{
    String text;
    Bitmap icon;
    public ImageText(Bitmap icon, String text)
    {
        this.icon   = icon;
        this.text   = text;
    }
}
{% endhighlight %}

## Data Sources

As mentioned before, I implemented 3 independent data sources each of which can be selected from the application:

* Memory
* Local Database
* Web Service

Local Database is a simple SQLite database which resides on Android file system and has the following database schema:

{% highlight bash %}
$ sqlite3 cells.db
{% endhighlight %}

{% highlight sql %}
sqlite> .schema
CREATE TABLE android_metadata (locale TEXT);
CREATE TABLE cells (_id INTEGER PRIMARY KEY,cid INTEGER,cell INTEGER);
sqlite> select * from cells;
1|0|0
2|1|2
3|2|1
4|3|-1
{% endhighlight %}

I will show here the examples of implementation `set()` method for each of these data sources:

### Memory

{% highlight java linenos %}
class MemoryMapData implements IMapData
{
    private int[] cells     = new int[MetaHome.NUM_ITEMS];

    public void set(int index, int value) {
        cells[index]    = value;
    }
    ...
}
{% endhighlight %}

### Local Database

{% highlight java linenos %}
class DatabaseMapData implements IMapData
{
    private DatabaseHelper mOpenHelper;

    private static final String DBNAME = "cells.db";
    private static final int DBVERSION = 3;

    public void set(int index, int value) {

        SQLiteDatabase db = mOpenHelper.getWritableDatabase();
        ContentValues values    = new ContentValues();
        values.put(Cells.COLUMN_CELL, value);
        db.update(Cells.TABLE_NAME, values, Cells.COLUMN_ID+"=?", new String[] {Long.toString(index)});
    }
    ...
}
{% endhighlight %}

### Web Service

{% highlight java linenos %}
class WebMapData implements IMapData
{

    private static final String ENDPOINT    = "http://surfingbits.com/metahome/index.php";
    private static final String KEY         = "access_key";

    public void set(int index, int value) {
        HashMap<String, String> params  = new HashMap<String, String>();
        params.put("action", "set");
        params.put("cell", Long.toString(index));
        params.put("value", Long.toString(value));
        JSONObject js   = getContentObject(params);

        boolean status;
        try {
            status  = js.getBoolean("status");
            Log.i("Web", "Set value cell["+index+"] = "+value+": "+status);
        } catch (Exception e) {
            Log.e("WebError", e.toString());
        }
    }
    ...
}
{% endhighlight %}

## Web Service Interface and Implementation

For Web Service I implemented a simple PHP script which handles incoming requests from the application. In the backend the script also uses SQLite database with the same database schema as Android is using for the `Local Database` source.

{% highlight php linenos %}
<?php

// Copyright 2011 Alex Dementsov

$key        = "access_key";
$actions    = array("get", "getall", "remove", "set", "size", "empty");

class Handler
{
    private $db;

    public function __construct(){
        $this->open_db();
    }

    function run()
    {
        global $key, $actions;

        if (!(isset($_GET["key"]) && $_GET["key"] == $key) ||
            !(isset($_GET["action"]) && in_array($_GET["action"], $actions)))
        {
            echo "Wrong key or action";
            return;
        }

        switch($_GET["action"])
        {
            case "get":
                return $this->get();
            case "getall":
                return $this->get_all();
            case "remove":
                return $this->remove();
            case "set":
                return $this->set();
            case "size":
                return $this->size();
            case "empty":
                return $this->isempty();
        }
    }
    function get()
    {
        if (!isset($_GET["cell"]))
            return $this->status_fail();

        $row    = $this->getValue();
        if (!$row)
            return $this->status_fail();

        return json_encode(array("cell" => $row["cid"], "value" => $row["cell"]));
    }


    function get_all()
    {
        $array  = array(-1, -1, -1, -1);
        $rows   = $this->getValues();
        if (!$rows)
            return $this->status_fail();

        foreach ($rows as $i => $value)
        {
            $idx = $value["cid"];
            if ($idx >= 0 && $idx < 4)
                $array[$idx] = $value["cell"];
        }
        return json_encode(array("cells" => $array));
    }

    function remove()
    {
        if (!isset($_GET["cell"]))
            return $this->status_fail();

        return $this->setValue(-1);
    }

    function set()
    {
        if (!isset($_GET["cell"]) or !isset($_GET["value"]))
            return $this->status_fail();

        return $this->setValue($_GET["value"]);
    }

    private function update($arr)
    {
        // Executes query and returns status
        $query  = "update cells set cell=? where cid=?";
        $q      = $this->db->prepare($query);
        $q->execute($arr) or die(print_r($this->db->errorInfo(), true));
        return $q;
    }

    private function getValue()
    {
        // Returns value of a cell
        $result = $this->db->query("select * from cells where cid=".$_GET["cell"]) or die($this->status_fail());
        return $result->fetch();
    }

    private function getValues()
    {
        // Returns value of a cell
        $result = $this->db->query("select * from cells limit 4") or die($this->status_fail());
        return $result->fetchAll();
    }

    private function setValue($value)
    {
        $arr    = array($value, $_GET["cell"]);
        $q      = $this->update($arr);
        if (!$q)
            return $this->status_fail();

        return $this->status_ok();
    }

    function size()
    {
        return json_encode(array("size" => 4));
    }

    function isempty()
    {
        $row    = $this->getValue();
        if (!$row)
            return $this->status_fail();

        if ($row["cell"] == -1)
            return json_encode(array("empty" => True));

        return json_encode(array("empty" => False));
    }

    private function status_fail()
    {
        return $this->status("fail");
    }

    private function status_ok()
    {
        return $this->status("ok");
    }

    private function status($status)
    {
        return json_encode(array("status" => $status));
    }

    private function open_db()
    {
        try {
            $this->db = new PDO('sqlite:db/cells.db');
        }
        catch(Exception $e) {
            die($this->status_fail());
        }
    }
}

$handler = new Handler();
echo $handler->run();
?>
{% endhighlight %}

Here is the interface for the Web Service:

### Get All Cells

    Example Request:
    http://HOSTNAME/metaservice/index.php?action=getall&key=access_key

    Example Response:
    {
        "cells": ["2", "3", "-1", "-1"]
    }

### Get Cell

    Example Request:
    http://HOSTNAME/metaservice/index.php?action=get&cell=3&key=access_key

    Example Response:
    {
        "cell": 3,
        "value": 2
    }

### Remove Cell

    Example Request:
    http://HOSTNAME/metaservice/index.php?action=remove&cell=3&key=access_key

    Example Response:
    {
        "status": "ok"
    }

### Set Cell

    Example Request:
    http://HOSTNAME/metaservice/index.php?action=set&cell=3&value=2&key=access_key

    Example Response:
    {
        "status": "ok"
    }

### Get Cells Size

    Example Request:
    http://HOSTNAME/metaservice/index.php?action=size&key=access_key

    Example Response:
    {
        "size": 4
    }

### Is Empty

    Example Request:
    http://HOSTNAME/metaservice/index.php?action=empty&cell=3&key=access_key

    Example Response:
    {
        "empty": true
    }

If something goes wrong the response returns error:

    {
        "status": "fail"
    }

## Caching Mechanism and Other Trick

Using Web Service data source without caching makes user experience not very pleasant. I implemented some caching mechanism to make experience better. Remote calls are made to Web Service only when it is absolutely necessary, for example to change the state of the cells. Other requests are made locally from the stored data attribute `ldata`.

{% highlight java linenos %}
public class MapView extends View {

    private IMapData ldata;         // Local storage for the data (cache)
    ...
}
{% endhighlight %}

This attribute is used for all three data sources even when it somewhat duplicates storage (as in Memory data source). This simple caching works surprisingly well for Web Service data source.

Other issue I had with the application is the screen rotation. When you rotate the mobile phone all the local objects are gone because Android restarts the running Activity unless you take some measures to keep the state. Here is the reference to Android documentation which discusses this issue: [http://developer.android.com/guide/topics/resources/runtime-changes.html](http://developer.android.com/guide/topics/resources/runtime-changes.html)

To do the trick I implemented `onSaveInstanceState()` and `onRestoreInstanceState(Parcelable state)`.

{% highlight java linenos %}
public class MapView extends View {

    @Override
    protected Parcelable onSaveInstanceState() {
        Bundle b = new Bundle();

        Parcelable s = super.onSaveInstanceState();
        b.putParcelable("map_super_state", s);
        b.putIntArray("map_cells", dumpCells(ldata));
        b.putIntArray("memory_cells", dumpCells(memory));
        return b;
    }

    @Override
    protected void onRestoreInstanceState(Parcelable state)
    {
        Bundle b = (Bundle) state;
        Parcelable superState = b.getParcelable("map_super_state");
        loadCells(ldata, b, "map_cells");
        loadCells(memory, b, "memory_cells");

        super.onRestoreInstanceState(superState);
    }
    ...
}
{% endhighlight %}

## Real Example

To show the possible application of the idea I present here the map of the Terminal 4 at Los Angeles International Airport (LAX). It looks a bit more complicated than what I showed you before :) .

[source-metahome]: https://bitbucket.org/dexity/metahome
