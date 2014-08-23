---
layout: post
title: Sea World Tiles
tags: android
---

After hanging out you with my friends from Oregon to the SeaWorld in San Diego, CA I decided that it is might be cool to develop a simple Android application which keeps our memory about the adventure parks.

{{ more }}

So here is the app called Sea World Tiles. Feel free to download it from Android Market and play around. In this app you need to compose the picture of a sea creature from left and right sides by clicking the appropriate buttons.

The source code is available in the repository surfsnippets in directory shamu. Here is the short description of source code to show how simple it is to write Android applications.

The layout in Android applications is described in .xml file, main.xml and includes TableLayout with two TableRow. The upper TableRow displays two tiles of ImageView and the bottom TableRow keeps left and right buttons:

