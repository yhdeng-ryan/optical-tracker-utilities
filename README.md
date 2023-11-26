# Additional Info

## Atracsys
The Atracsys system is simpler, you need a json file for each tool and another json file that combine all tools. 
1. Use `tool_maker.py` to create the tool json files individually (`example/atracsys/ee.json` & `example/atracsys/rf.json`)
```
python tool_maker.py -t /atracsys/measured_cp -n 5 -o ee.json -p -i 1
python tool_maker.py -t /atracsys/measured_cp -n 5 -o rf.json -i 2
```
2. Create a tools json file (`example/atracsys/calibration.json`)
3. Copy all files and paste them in the atracsys folder `~/catkin_ws/src/cisst-saw/sawAtracsysFusionTrack/share`
4. Run atracsys node with tools json file
```
cd ~/catkin_ws/src/cisst-saw/sswAtracsysFusionTrack/share
rosrun atracsys_ros atracsys_json -j calibration.json
```

## Polaris
Using Polaris is slightly convoluted because the `tool_maker.py` cannot directly generate .rom file desite mensioning it in its README.
1. Use `tool_maker.py` to create the tool json files individually (`example/polaris/ee.json` & `example/polaris/rf.json`)
```
python tool_maker.py -t /atracsys/measured_cp -n 5 -o ee.json -p
python tool_maker.py -t /atracsys/measured_cp -n 5 -o rf.json
```
2. Use `tool_converter.py` to convert them into .rom files. I added a new argument -p so that .rom files would not share the same unique-id and break when use together. Also I set the default tool_main_type in `ndi_tool.py` to 1. It was set to 0 and it's not working.
```
python tool_converter.py -i ee.json -o ee.rom -p EEMarkers
python tool_converter.py -i rf.json -o rf.rom -p RFMarkers
```
3. Create a tools json file (`example/polaris/calibration.json`)
4. Find the unique-id of the .rom files and put them in the tools json file. There are two ways to find it.
    1. You can simply put a wrong number and run the node, it would show the correct ones along with your incorrect ones. Or
    2. The unique-id accually consist of 3 parts as "tool_main_type" - "timestamp_data" - "part_number"
        - "tool_main_type" is set to 1 in `ndi_tool.py`. You can modified it to fit your need.
        - Use hexdump to display the contents in .rom files
        ```
        hexdump ee.rom -C
        ```
        - Locate the timestamp_data encode in the order byte 17, 16, 15, 14. For example if you see
        ```
        00000000  4e 44 49 00 0b 21 00 00  01 00 00 00 02 00 00 01  |NDI..!..........|
        00000010  00 00 00 00 00 c4 bb 3d  5a 00 00 00 05 00 00 00  |.......=Z.......|
        ```
        3DBBC400 will be your timestamp_data. Btw, tool_main_type that you enter in `tool_convert.py` should be starting from byte 250 as
        ```
        00000250  45 45 4d 61 72 6b 65 72  73 00 00 00 00 00 00 00  |EEMarkers.......|
        ```
        - Your combined tool_main_type will be
        ```
        01-3DBBC400-EEMarkers
        ```
5. Polaris supports reference tool, so you can put the tool name under tag "reference".
6. Copy all files and paste them in the Polaris folder `~/catkin_ws/src/cisst-saw/sawNDITracker/share`
7. Run polaris node with tools json file
```
cd ~/catkin_ws/src/cisst-saw/sawNDITracker/share
rosrun ndi_tracker_ros ndi_tracker -j calibration.json
```

Hope these additional instructions helps!


# Optical Tracker Utilities

Scripts used to generate tool definition files (i.e. relative positions of stray markers within a single rigid body) for optical trackers.  These scripts have been developped at the Johns Hopkins University and tested with the following optical tracking systems:
* [NDi](https://www.ndigital.com) (Northern Digital Inc) Polaris and Vicra using [**sawNDITracker**](https://github.com/jhu-saw/sawNDITracker)
* [Atracsys](https://www.atracsys-measurement.com) FusionTrack ft500 using [**sawAtracsysFusionTrack**](https://github.com/jhu-saw/sawAtracsysFusionTrack)

**sawNDITracker** and **sawAtracsysFusionTrack** are two C++ libraries interfacing with NDi and FusionTrack products.  These libraries provide a GUI as well as bridges to ROS 1, ROS 2 and OpenIGTLink.  These are not required to use the scripts in this repository.

# Scripts

## `tool_maker.py`

This script can be used to create a tool definition file from stray marker positions.  The script assumes the marker positions are provided using a ROS topic publishing the poses using a `geometry_msgs/PoseArray` message.  You can create the tool definition "live", i.e. while the optical tracker is running or you can record and replay a bag of stray marker poses.  When collecting the stray marker poses, the tool should be visible and static (i.e. do not move the tool nor the tracking system).  Make sure the stray markers on the tool are the only markers visible. 

The output format can be either the _Atracsys_ compatible `.ini` or our custom _SAW_ `.json` format.  The component **sawAtracsysFusionTrack** can load either. 

## `tool_converter.py`

Tool to convert tool definition files between the following formats:
* `.ini` text file from _Atracsys_.
* `.json` text file from **sawAtracsysFusionTrack**.  This format is based on the `.ini` format and was introduced for older Ubuntu distributions to avoid parsing `.ini` files.
* `.rom` binary files from _NDi_.
