# FUSION360-KOSY-WABECO
This is a modified version of the KOSY postprocessor for Autodesk Fusion 360 (https://cam.autodesk.com/posts/post.php?name=kosy) for Wabeco CNC Mills working with nccad75


## Remarks
* Since the basic version of nccad75 only supports 2 1/2d movements, this postprocessor has helical movements turned off (it will linearize helical movements).
* If you are using a mill with dovetail guides the max speed for linear movements is 100mm/s. This needs to be considered in Fusion.
* This postprocessor writes out a G77 command at the end to return to a home position. You might want to remove it, I'm not sure if it is supported by nccad75 and Wabeco.

## Why modify?
This Postprocessor uses a different format for G2 and G3 (Circlular, clockwise and counter-clockwise) movements then the kosy postprocessor by Autodesk. 

The Wabeco version of nccad supports two formats for circular movements. The first one is the more common version, that is also used in the original kosy postprocessor:

**G01 X20 Y60            ;go to starting point** 

**G02 X20 Y40 I0 J-10    ;180 degree-arc clockwise**

On the G02 line X and Y stand for the end point, I stands for the the distance between the starting point and center center of the circle in x direction and J stands for the distance between the starting point of the circle and the center in y direction.

The problem here is, that for the starting point, both values X and Y are needed in order to make nccad draw the circle. But Fusion360 removes all redundant lines and parameters from the code. This means that if your starting point is on the same axis (X or Y) than your previous point, Fusion will delete the redundant parameter to reduce file size. It is a nice feature, but unfortunately makes nccad crash.

Fortunately there is a second way to format G2 and G3 g-code commands: 

**G02 D50 I40 J-5        ;50 degree-arc**

This format comes without the end point (no X and Y), but has an additional D parameter. This D parameter indicates the angle in degrees.

So all we have to do is to calculate the angle of the arc and change the output for circlular movements in the cps file.

## Modifications

On line 33, `allowHelicalMoves` is set `false` by default

On line 82 we introduce a new reference variable for the D parameter: 

```javascript
var dOutput = createReferenceVariable({prefix:"D", force:true}, xyzFormat); // WABECO MOD: new output format for angle of circle
```

In the function "onCircular" on line 678 we add the following lines to calculate the angle using default javascript Math functions:

```javascript
var middle = {"x": (x-start.x)/2, "y": (y-start.y)/2}; //stores the distance between the middle between the start and and point
var lengthOpposite = Math.sqrt(Math.pow(middle.x,2)+Math.pow(middle.y,2)); //Calculate the distance between middle and end point using Pythagoras
var lengthAdjacent = Math.sqrt(Math.pow(cx-(start.x+middle.x),2)+Math.pow(cy-(start.y+middle.y),2)); //calculate the distance between middle and center using Pythagoras
var angleRadians = Math.atan(lengthOpposite/lengthAdjacent) * 2; //calculate angle in radians using tangens
var angleDegrees = angleRadians*180/Math.PI; //convert angle from radians to degrees
```

and then change the  output lines on line 705 and 713 to:

```javascript
writeBlock(gMotionModal.format(clockwise ? 2:3), dOutput.format(angleDegrees, 0), iOutput.format(cx - start.x, 0), jOutput.format(cy - start.y, 0), feedOutput.format(feed));
```
