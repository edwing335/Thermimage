Thermimage: Thermal Image Analysis
====



[![cran version](https://www.r-pkg.org/badges/version/Thermimage)](https://www.r-pkg.org/badges/version/Thermimage)
[![downloads](https://cranlogs.r-pkg.org/badges/Thermimage)](https://cranlogs.r-pkg.org/badges/Thermimage)
[![total downloads](https://cranlogs.r-pkg.org/badges/grand-total/Thermimage)](https://cranlogs.r-pkg.org/badges/grand-total/Thermimage)
[![Research software impact](http://depsy.org/api/package/cran/Thermimage/badge.svg)](http://depsy.org/package/r/Thermimage)

This is a collection of functions for assisting in converting extracted raw data from infrared thermal images and converting them to estimate temperatures using standard equations in thermography.

## Recent/release notes

* Version 3.0.0 is on Github (development version)
* Changes in this release include functions for importing thermal video files and exporting for ImageJ functionality

* Version 2.2.3 is on CRAN (as of October 2016). 
* Changes in this release include readflirjpg and flirsettings functions for processing flir jpg meta tag info.

## Features

* Functions for importing FLIR image and video files into R.
* Functions for converting thermal image data from FLIR based files, incorporating calibration information stored within each radiometric image file.
* Functions for exporting calibrated thermal image data for analysis in open source platforms, such as ImageJ.
* Functions for steady state estimates of heat exchange from surface temperatures estimated by thermal imaging.
* Functions for modelling heat exchange under various convective, short-wave, and long-wave radiative heat flux, useful in thermal ecology studies.

## Installation


### On current R (>= 3.0.0)

* From CRAN (stable releases 1.0.+):

```
install.packages("Thermimage")
```

* Development version from Github:

```
library("devtools"); install_github("gtatters/Thermimage",dependencies=TRUE)
```

## Package Imports

* Imports: tiff, png

* Suggests: ggplot2, fields, reshape

## OS Requirements

* Exiftool is required for certain functions.  Installation instructions can be found here: http://www.sno.phy.queensu.ca/~phil/exiftool/install.html

## Examples

## A typical thermal image

![Galapagos Night Heron](https://github.com/gtatters/Thermimage/blob/master/inst/extdata/IR_2412.jpg?raw=true)

Normally, these thermal images require access to software that only runs on Windows operating system.  This package will allow you to import certain FLIR jpgs and videos and process the images.

## Import FLIR JPG

To load a FLIR JPG, you first must install Exiftool as per instructions above.
Open sample flir jpg included with Thermimage package:

```
library(Thermimage)
f<-paste0(system.file("extdata/IR_2412.jpg", package="Thermimage"))
img<-readflirJPG(f, exiftoolpath="installed")
dim(img)
```
[1] 480 640

The readflirJPG function has used Exiftool to figure out the resolution and properties of the image file.  Above you can see the dimensions are listed as 480 x 640.  Before plotting or doing any temperature assessments, let's extract the meta-tages from the thermal image file.


# Extract meta-tags from thermal image file

```
cams<-flirsettings(f, exiftoolpath="installed", camvals="")
```

This produes a rather long list of meta-tags.  If you only want to see your camera calibration constants, type:

```
plancks<-flirsettings(f, exiftoolpath="installed", camvals="-*Planck*")
unlist(plancks$Info)
```
 PlanckR1       PlanckB       PlanckF       PlanckO      PlanckR2 
 2.110677e+04  1.501000e+03  1.000000e+00 -7.340000e+03  1.254526e-02 


If you want to check the file data information, type:
```
cbind(unlist(cams$Dates))
```
FileModificationDateTime "2017-03-22 22:15:09"
FileAccessDateTime       "2017-03-22 23:27:31"
FileInodeChangeDateTime  "2017-03-22 22:15:10"
ModifyDate               "2013-05-09 16:22:23"
CreateDate               "2013-05-09 16:22:23"
DateTimeOriginal         "2013-05-09 22:22:23"

or just:
```
cams$Dates$DateTimeOriginal
```
[1] "2013-05-09 22:22:23"

The most relevant variables to extract for calculation of temperature values from raw A/D sensor data are listed here.  These can all be extracted from the cams output as above. I have simplified the output below, since dealing with lists can be awkward.

```
Emissivity<-  cams$Info$Emissivity                    # Image Saved Emissivity - should be ~0.95 or 0.96
dateOriginal<-cams$Dates$DateTimeOriginal             # Original date/time extracted from file
dateModif<-   cams$Dates$FileModificationDateTime     # Modification date/time extracted from file
PlanckR1<-    cams$Info$PlanckR1                      # Planck R1 constant for camera  
PlanckB<-     cams$Info$PlanckB                       # Planck B constant for camera  
PlanckF<-     cams$Info$PlanckF                       # Planck F constant for camera
PlanckO<-     cams$Info$PlanckO                       # Planck O constant for camera
PlanckR2<-    cams$Info$PlanckR2                      # Planck R2 constant for camera
OD<-          cams$Info$ObjectDistance                # object distance in metres
FD<-          cams$Info$FocusDistance                 # focus distance in metres
ReflT<-       cams$Info$ReflectedApparentTemperature  # Reflected apparent temperature
AtmosT<-      cams$Info$AtmosphericTemperature        # Atmospheric temperature
IRWinT<-      cams$Info$IRWindowTemperature           # IR Window Temperature
IRWinTran<-   cams$Info$IRWindowTransmission          # IR Window transparency
RH<-          cams$Info$RelativeHumidity              # Relative Humidity
h<-           cams$Info$RawThermalImageHeight         # sensor height (i.e. image height)
w<-           cams$Info$RawThermalImageWidth          # sensor width (i.e. image width)
```

## Convert raw binary to temperature

Now you have the img loaded, look at the values:
```
str(img)
```
 int [1:480, 1:640] 18090 18074 18064 18061 18081 18057 18092 18079 18071 18071 ...


If stored with a TIFF header, the data load in as a pre-allocated matrix of the same dimensions of the thermal image, but the values are integers values, in this case ~18000.  The data are stored as in binary/raw format at 2^16 bits of resolution = 65535 possible values, starting at 1.  These are not temperature values.  They are, in fact, radiance values or absorbed infrared energy values in arbitrary units.  That is what the calibration constants are for.  The conversion to temperature is a complicated algorithm, incorporating Plank's law and the Stephan Boltzmann relationship, as well as atmospheric absorption, camera IR absorption, emissivity and distance to namea  few.  Each of these raw/binary values can be converted to temperature, using the raw2temp function:

```
temperature<-raw2temp(img, ObjectEmissivity, OD, ReflT, AtmosT, IRWinT, IRWinTran, RH,
                      PlanckR1, PlanckB, PlanckF, PlanckO, PlanckR2)
str(temperature)      
```
num [1:480, 1:640] 23.7 23.6 23.6 23.6 23.7 ...


The raw binary values are now expressed as temperature in degrees Celsius (apologies to Lord Kelvin).  Let's plot the temperature data: 

```
library(fields) # should be loaded imported when installing Thermimage
plotTherm(t(temperature), h, w)
```
![FLIR JPG on import](https://github.com/gtatters/Thermimage/blob/master/READMEimages/FlirJPGdefault.png?raw=true)

The FLIR jpg imports as a matrix, but default plotting parameters leads to it being rotated 270 degrees (counter clockwise) from normal perspective, so you should either rotate the matrix data before plotting, or include the rotate270.matrix transformation in the call to the plotTherm function:

```
plotTherm(temperature, w=w, h=h, minrangeset = 21, maxrangeset = 32, trans="rotate270.matrix")
```

![FLIR JPG rotate 270](https://github.com/gtatters/Thermimage/blob/master/READMEimages/FlirJPGrotate270.png?raw=true)

If you prefer a different palette:
```
plotTherm(temperature, w=w, h=h, minrangeset = 21, maxrangeset = 32, trans="rotate270.matrix", 
          thermal.palette=rainbowpal)
plotTherm(temperature, w=w, h=h, minrangeset = 21, maxrangeset = 32, trans="rotate270.matrix", 
          thermal.palette=glowbowpal)
plotTherm(temperature, w=w, h=h, minrangeset = 21, maxrangeset = 32, trans="rotate270.matrix", 
          thermal.palette=midgreypal)
plotTherm(temperature, w=w, h=h, minrangeset = 21, maxrangeset = 32, trans="rotate270.matrix", 
          thermal.palette=midgreenpal)
```
![FLIR JPG rotate 270 rainbow palette](https://github.com/gtatters/Thermimage/blob/master/READMEimages/FLIRJPGrotate270rainbowpal.png?raw=true)

![FLIR JPG rotate 270 glowbow palette](https://github.com/gtatters/Thermimage/blob/master/READMEimages/FLIRJPGrotate270glowbowpal.png?raw=true)

![FLIR JPG rotate 270 midgrey palette](https://github.com/gtatters/Thermimage/blob/master/READMEimages/FLIRJPGrotate270midgreypal.png?raw=true)


## Export Image or Video

Finding a way to quantitatively analyse thermal images in R is a challenge due to limited interactions with the graphics environment.  Thermimage has a function that allows you to write the image data to a file format that can be imported into ImageJ.  

First, the image matrix needs to be transposed (t) to swap the row vs. column order in which the data are stored, then the temperatures need to be transformed to a vector, a requirement of the writeBin function.  The function writeFlirBin is a wrapper for writeBin, and uses information on image width, height, frame number and image interval (the latter two are included for thermal video saves) but are kept for simplicity to contruct a filename that incorporates image information required when importing to ImageJ:
```
writeFlirBin(as.vector(t(temperature)), templookup=NULL, w=w, h=h, I="", rootname="FLIRjpg")
```
The raw file can be found here: https://github.com/gtatters/Thermimage/blob/master/READMEimages/FLIRjpg_W640_H480_F1_I.raw?raw=true

# Import Raw File into ImageJ
The .raw file is simply the pixel data saved in raw format but with real 32-bit precision.  This means that the temperature data (negative or positive values) are encoded in 4 byte chunks.  ImageJ has a plethora of import functions, and the File-->Import-->Raw option provides great flexibility.  Once opening the .raw file in ImageJ, set the width, height, number of images (i.e. frames or stacks), byte storage order (little endian), and hyperstack (if desired):

![ImageJ Import Settings](https://github.com/gtatters/Thermimage/blob/master/READMEimages/ImageJImport.png?raw=true)

The image imports clearly just as it would in a thermal image program.  Each pixel stores the calculated temperatures as provided from the raw2temp function above. 

![Image Imported into ImageJ](https://github.com/gtatters/Thermimage/blob/master/READMEimages/FLIRjpg_W640_H480_F1_I.raw.png?raw=true)


## Importing Thermal Videos

Importing thermal videos (March 2017: still in development) is a little more involved and less automated, but below are steps that have worked for seq and fcf files tested.

Set file info and extract meta-tags as done above:
```
# set filename as v
v<-paste0(system.file("extdata/SampleSEQ.seq", package="Thermimage"))

# Extract camera values using Exiftool (needs to be installed)
camvals<-flirsettings(v)
w<-camvals$Info$RawThermalImageWidth
h<-camvals$Info$RawThermalImageHeight
```

Create a lookup variable to convert the raw binary to actual temperature estimates, use parameters relevant to the experiment.  You could use the values stored in the FLIR meta-tags, but these are not necessarily correct for the conditions of interest.  suppressWarnings() is used because of NaN values returned for binary values that fall outside the range.

```
suppressWarnings(
templookup<-raw2temp(raw=1:65535, E=camvals$Info$Emissivity, OD=camvals$Info$ObjectDistance, RTemp=camvals$Info$ReflectedApparentTemperature, ATemp=camvals$Info$AtmosphericTemperature, IRWTemp=camvals$Info$IRWindowTemperature, IRT=camvals$Info$IRWindowTransmission, RH=camvals$Info$RelativeHumidity, PR1=camvals$Info$PlanckR1,PB=camvals$Info$PlanckB,PF=camvals$Info$PlanckF,PO=camvals$Info$PlanckO,PR2=camvals$Info$PlanckR2)
)

```

![Binary to Temperature Conversion](https://github.com/gtatters/Thermimage/blob/master/READMEimages/CalibrationCurve.png?raw=true)


Using the width and height information, we use this to find where in the video file these are stored.  This corresponds to reproducible locations in the frame header:
```
fl<-frameLocates(v, w, h)
n.frames<-length(fl$f.start)
n.frames; fl
```
[1] 2
$h.start
[1]    162 308688

$f.start
[1]   1391 309917

The relative positions of the header start (h.start) are 162 and 308688, and the frame start (f.start) positions are 1391 and 309917.  The video file is a short, two frame (n.frames) sequence from a thermal video.

Then pass the fl data to two different functions, one to extract the time information from the header, and the other to extract the actual pixel data from the image frame itself.  The lapply function will have to be used (for efficiency), but to wrap the function across all possible detected image frames.  Note: For large files, the parallel function, mclapply, is advised (?getFrames for an example):

```
extract.times<-do.call("c", lapply(fl$h.start, getTimes, vidfile=v))
data.frame(extract.times)
```
            extract.times
1 2012-06-13 15:52:08.698
2 2012-06-13 15:52:12.665
```
Interval<-signif(mean(as.numeric(diff(extract.times))),3)
Interval
```
[1] 3.97

This particluar sequence was actually captured at 0.03 sec intervals, but the sample file in the package was truncated to only two frames to minimise online size requirements for CRAN.  At present, the getTimes function produces an error on capturing the first frame time.  On the original 100 frame file, it accurately captures the real time stamps, so the error is appears to be how FLIR saves time stamps (save time vs. modification time vs. original time appear highly variable in .seq and .fcf files).  Precise time capture is not crucial but is helpful for verifying data conversion.

After extracting times, then extract the frame data, with the getFrames function:

```
alldata<-unlist(lapply(fl$f.start, getFrames, vidfile=v, w=w, h=h))
class(alldata); length(alldata)/(w*h)
```
[1] "integer"
[1] 2

The raw binary data are stored as an integer vector.  Length(alldata)/(w*h) verifies the total # of frames in the video file is 2.

Before



## Heat Transfer Calculation


