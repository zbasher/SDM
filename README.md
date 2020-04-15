SDM
================

This R script accompanies the following research article:

Frida Ben Rais Lasram, Tarek Hattab, Quentin Nogues, Grégory Beaugrand,
Jean Claude Dauvin, Ghassen Halouani, François Le Loc’h, Nathalie
Niquil, Boris Leroy, (2020). An open-source framework to project
potential future distributions of marine species at local scale
(currently under review in Ecological informatics).

It includes (i) a procedure for homogenizing occurrence data to
eliminate the influence of sampling bias, (ii) a procedure for
generating pseudo-absences, (iii) a hierarchical-filter approach
(i.e. global Bioclimatic Envelope Models combined with local Habitat
Models), (iv) full incorporation of the third dimension by considering
climatic variables at multiple depths and (v) building of maps that
predict current and future ranges of marine species.

### Running the script inside a docker container

If you want to launch RStudio inside of a *Docker* container, first of
all you have to install Docker.

Then you will have to run an image of RStudio Server containing all the
R packages and their system dependencies.

``` r
docker run -d -p 8787:8787 -e PASSWORD="secret" tarek1984/r-enm:0.0.1
```

The command will print the ID of the new container and exit. Now you can
open your browser at <http://localhost:8787>, enter “rstudio” as
unsername, and “secret” as password. If you are running a Mac or Windows machine
open a browser and enter http://, followed by your ip address, followed by :8787.

### Load required libraries

``` r
library(spocc)
library(robis)
library(sp)
library(rgdal)
library(maptools)
library(raster)
library(SDMTools)
library(gridExtra)
library(biomod2)
library(PresenceAbsence)
library(rgeos)
library(plyr)
library(ecospat)
library(ade4)
library(rworldmap)
require(stringr)
library(colorRamps)
library(biogeonetworks)
```

### Set working directory

``` r
setwd("/home/tarek/Bureau/SDM")
```

### Global parameters

Four parameters must be set by users: (i) the species scientific name
(ii) the species’ vertical habitat (iii) the list of algorithms to be
used and (iv) the choice of K in K-fold cross-validation.

``` r
Species<-"Mullus surmuletus" # The scientific name of the species
Vertical_habitat<-"Demersal" #  it can take one of the following applelations: Benthopelagic","Pelagic","Benthic","Demersal
models <- c('GLM', 'GBM', 'GAM', 'CTA', 'ANN', 'FDA', 'MARS', 'RF')
k <- 3 # Numbers of k in the  k-fold validation
```

### Import and preprocess environmental data

For model calibration, the script consider temperature and salinity
climatologies from the global database WOD 2013 V2
(<https://www.nodc.noaa.gov/OC5/woa13/>). See *step 2* in model
framework section in Ben Rais Lasram et al (2020).

Note that the script uses average temperature and salinity on the first
50m depth for the calibration of the pelagic species models, on the
first 200m depth for the benthopelagic species and on the last 50m depth
for benthic and demersal species.

``` r
#Download climatic data for each time period
download.file("https://pcsbox.univ-littoral.fr/d/b75dc393597748659c0f/files/?p=%2FEnvironmental%20data%2FClimatic_data%2FWOD%2FTemperature_1955_1964.RData&dl=1", destfile = "Temperature_1955_1964.RData", mode = "wb")
download.file("https://pcsbox.univ-littoral.fr/d/b75dc393597748659c0f/files/?p=%2FEnvironmental%20data%2FClimatic_data%2FWOD%2FTemperature_1965_1974.RData&dl=1", destfile = "Temperature_1965_1974.RData", mode = "wb")
download.file("https://pcsbox.univ-littoral.fr/d/b75dc393597748659c0f/files/?p=%2FEnvironmental%20data%2FClimatic_data%2FWOD%2FTemperature_1975_1984.RData&dl=1", destfile = "Temperature_1975_1984.RData", mode = "wb")
download.file("https://pcsbox.univ-littoral.fr/d/b75dc393597748659c0f/files/?p=%2FEnvironmental%20data%2FClimatic_data%2FWOD%2FTemperature_1985_1994.RData&dl=1", destfile = "Temperature_1985_1994.RData", mode = "wb")
download.file("https://pcsbox.univ-littoral.fr/d/b75dc393597748659c0f/files/?p=%2FEnvironmental%20data%2FClimatic_data%2FWOD%2FTemperature_1995_2004.RData&dl=1", destfile = "Temperature_1995_2004.RData", mode = "wb")
download.file("https://pcsbox.univ-littoral.fr/d/b75dc393597748659c0f/files/?p=%2FEnvironmental%20data%2FClimatic_data%2FWOD%2FTemperature_2005_2012.RData&dl=1", destfile = "Temperature_2005_2012.RData", mode = "wb")
download.file("https://pcsbox.univ-littoral.fr/d/b75dc393597748659c0f/files/?p=%2FEnvironmental%20data%2FClimatic_data%2FWOD%2FSalinity_1955_1964.RData&dl=1", destfile = "Salinity_1955_1964.RData", mode = "wb")
download.file("https://pcsbox.univ-littoral.fr/d/b75dc393597748659c0f/files/?p=%2FEnvironmental%20data%2FClimatic_data%2FWOD%2FSalinity_1965_1974.RData&dl=1", destfile = "Salinity_1965_1974.RData", mode = "wb")
download.file("https://pcsbox.univ-littoral.fr/d/b75dc393597748659c0f/files/?p=%2FEnvironmental%20data%2FClimatic_data%2FWOD%2FSalinity_1975_1984.RData&dl=1", destfile = "Salinity_1975_1984.RData", mode = "wb")
download.file("https://pcsbox.univ-littoral.fr/d/b75dc393597748659c0f/files/?p=%2FEnvironmental%20data%2FClimatic_data%2FWOD%2FSalinity_1985_1994.RData&dl=1", destfile = "Salinity_1985_1994.RData", mode = "wb")
download.file("https://pcsbox.univ-littoral.fr/d/b75dc393597748659c0f/files/?p=%2FEnvironmental%20data%2FClimatic_data%2FWOD%2FSalinity_1995_2004.RData&dl=1", destfile = "Salinity_1995_2004.RData", mode = "wb")
download.file("https://pcsbox.univ-littoral.fr/d/b75dc393597748659c0f/files/?p=%2FEnvironmental%20data%2FClimatic_data%2FWOD%2FSalinity_2005_2012.RData&dl=1", destfile = "Salinity_2005_2012.RData", mode = "wb")
```

``` r
#Importing climatic data for each time period
Temperature1 <- brick(get(load("Temperature_1955_1964.RData")))
Temperature2 <- brick(get(load("Temperature_1965_1974.RData")))
Temperature3 <- brick(get(load("Temperature_1975_1984.RData")))
Temperature4 <- brick(get(load("Temperature_1985_1994.RData")))
Temperature5 <- brick(get(load("Temperature_1995_2004.RData")))
Temperature6 <- brick(get(load("Temperature_2005_2012.RData")))
spplot(Temperature1,names.attr=c("Bottom","0-50m","0-200m"),main="Temperature 1955-1964")
```

<img src="SDM_files/figure-gfm/unnamed-chunk-6-1.png" style="display: block; margin: auto;" />

``` r
Salinity1 <- brick(get(load("Salinity_1955_1964.RData")))
Salinity2 <- brick(get(load("Salinity_1965_1974.RData")))
Salinity3 <- brick(get(load("Salinity_1975_1984.RData")))
Salinity4 <- brick(get(load("Salinity_1985_1994.RData")))
Salinity5 <- brick(get(load("Salinity_1995_2004.RData")))
Salinity6 <- brick(get(load("Salinity_2005_2012.RData")))
spplot(Salinity1,names.attr=c("Bottom","0-50m","0-200m"),main="Salinity 1955-1964")
```

<img src="SDM_files/figure-gfm/unnamed-chunk-6-2.png" style="display: block; margin: auto;" />

``` r
# Download local scale climate maps (current)

download.file("https://pcsbox.univ-littoral.fr/d/b75dc393597748659c0f/files/?p=%2FEnvironmental%20data%2FClimatic_data%2FWOD%2FLocal%2FTemperature_2005_2012_local.RData&dl=1", destfile = "Temperature_2005_2012_local.RData", mode = "wb")
download.file("https://pcsbox.univ-littoral.fr/d/b75dc393597748659c0f/files/?p=%2FEnvironmental%20data%2FClimatic_data%2FWOD%2FLocal%2FSalinity_2005_2012_local.RData&dl=1", destfile = "Salinity_2005_2012_local.RData", mode = "wb")
```

``` r
# Import local scale climate maps (current)

TemperatureL <- brick(get(load("Temperature_2005_2012_local.RData")))
SalinityL <- brick(get(load("Salinity_2005_2012_local.RData")))
```

For projections, the script allows to use climate projections for
2041-2050 and 2091-2100 under the RCP 2.6 (strong mitigation) and 8.5
(high emission) scenarios of the IPCC (Intergovernmental Panel on
Climate Change). See *step 2* in model framework section in Ben Rais
Lasram et al (2020).

``` r
# Download local scale climate maps (GFDL-ESM2G_RCP2.6)

download.file("https://pcsbox.univ-littoral.fr/d/b75dc393597748659c0f/files/?p=%2FEnvironmental%20data%2FClimatic_data%2FCMIP5%2FDelta_Temperature_GFDL-ESM2G_RCP2.6.RData&dl=1", destfile = "Delta_Temperature_GFDL-ESM2G_RCP2.6.RData", mode = "wb")
download.file("https://pcsbox.univ-littoral.fr/d/b75dc393597748659c0f/files/?p=%2FEnvironmental%20data%2FClimatic_data%2FCMIP5%2FDelta_Salinity_GFDL-ESM2G_RCP2.6.RData&dl=1", destfile = "Delta_Salinity_GFDL-ESM2G_RCP2.6.RData", mode = "wb")
```

``` r
# Import local scale climate maps (GFDL-ESM2G_RCP2.6)

load("Delta_Temperature_GFDL-ESM2G_RCP2.6.RData")
Temperature_GFDL_ESM2G_RCP2.6 <- TemperatureL + Delta ;rm(Delta)
load("Delta_Salinity_GFDL-ESM2G_RCP2.6.RData")
Salinity_GFDL_ESM2G_RCP2.6 <- SalinityL + Delta ;rm(Delta)
```

Raw habitat data are available from EMODnet-bathymetry
(<http://www.emodnet-bathymetry.eu/>) and EMODnet- seabedhabitats
(<http://www.emodnet-seabedhabitats.eu/>) and has a 250m spatial
resolution. See *step 2* in model framework section in Ben Rais Lasram
et al (2020).

``` r
# Download habitat data

download.file("https://pcsbox.univ-littoral.fr/d/b75dc393597748659c0f/files/?p=%2FEnvironmental%20data%2FHabitat%20data%2FHabitat_data.RData&dl=1", destfile = "Habitat_data.RData", mode = "wb")
```

``` r
# Import habitat data 

habitat <- get(load("Habitat_data.RData")) 
spplot(habitat[,"Bathy"],names.attr="Depth (m)",main="Example of habitat data: Depth (m)")
```

<img src="SDM_files/figure-gfm/unnamed-chunk-12-1.png" style="display: block; margin: auto;" />

``` r
spplot(habitat[,"Seafloor"],main="Example of habitat data (Seafloor)")
```

<img src="SDM_files/figure-gfm/unnamed-chunk-12-2.png" style="display: block; margin: auto;" />

For habitat models ordination axes will be used and not the parameters
themself in order to reduce the multi colinearity between habitat
variables.

``` r
# Ordination of the habitat parmeters

ordi <- dudi.mix(habitat[, c(1,2,3,6)], scannf = FALSE, nf = 9) # Nine axes are kept
ordi <- ordi$li 
ordi <- data.frame(coordinates(habitat), ordi) # Spatializing ordination axes 
coordinates(ordi) <-~ x + y;ordi <- as(ordi, "SpatialPixelsDataFrame")
ordi <- raster::stack(ordi)
spplot(ordi[[1:4]],main="Ordination axes")
```

<img src="SDM_files/figure-gfm/unnamed-chunk-13-1.png" style="display: block; margin: auto;" />

### Environnemental backgound

The environnemental backgound is a grid encompassing all the
combinations of temperature and salinity occurring at the global scale.
It will be used to make the environmental filtering of occurrences data
and pseudo-absences generation. See *step 3* in model framework section
in Ben Rais Lasram et al (2020).

``` r
# Realised climat (Historical)

Bottom.comb=NULL
Surface.comb=NULL
Vertical.comb=NULL

# Extract all combinaison of climatic parameter for each layer of the vertical habitat for all time period chosen

for (time in c("1955_1964", "1965_1974", "1975_1984", "1985_1994", "1995_2004", "2005_2012")){
  load(paste("Temperature_", time, ".RData", sep = ""))
  load(paste("Salinity_", time, ".RData", sep = ""))
  Bottom.comb = rbind(Bottom.comb,data.frame(Temperature = Temperature@data[, 1],Salinity = Salinity@data[, 1]))
  Surface.comb = rbind(Surface.comb,data.frame(Temperature = Temperature@data[, 2],Salinity = Salinity@data[, 2]))
  Vertical.comb = rbind(Vertical.comb,data.frame(Temperature = Temperature@data[, 3],Salinity = Salinity@data[, 3]))
  rm(Temperature);rm(Salinity)
}

# Create a raster for each vertical habitat layer with temperature on the x axis and salinity on the y axis

Bottom.comb <- na.omit(Bottom.comb)
Bottom.r <- raster(nrows = 100,
                   ncols = 100,
                   xmn = min(Bottom.comb[, 1]),
                   xmx = max(Bottom.comb[, 1]),
                   ymn = min(Bottom.comb[, 2]),
                   ymx = max(Bottom.comb[, 2]))
Bottom.r <- rasterize(Bottom.comb, Bottom.r, fun="count")
Bottom.r = Bottom.r > 0
spplot(Bottom.r,main="Environnemental backgound (Bottom)",xlab="Temperature",ylab="Salinity")
```

<img src="SDM_files/figure-gfm/unnamed-chunk-14-1.png" style="display: block; margin: auto;" />

``` r
Surface.comb <- na.omit(Surface.comb)
Surface.r <- raster(nrows = 100, 
                    ncols = 100, 
                    xmn = min(Surface.comb[, 1]), 
                    xmx = max(Surface.comb[, 1]), 
                    ymn = min(Surface.comb[, 2]), 
                    ymx = max(Surface.comb[, 2]))
Surface.r <- rasterize(Surface.comb, Surface.r, fun="count")
Surface.r = Surface.r > 0
spplot(Surface.r,main="Environnemental backgound (0-50m)",xlab="Temperature",ylab="Salinity")
```

<img src="SDM_files/figure-gfm/unnamed-chunk-14-2.png" style="display: block; margin: auto;" />

``` r
Vertical.comb <- na.omit(Vertical.comb)
Vertical.r <- raster(nrows = 100, 
                     ncols = 100, 
                     xmn = min(Vertical.comb[, 1]), 
                     xmx = max(Vertical.comb[, 1]), 
                     ymn = min(Vertical.comb[ ,2]), 
                     ymx = max(Vertical.comb[, 2]))
Vertical.r <- rasterize(Vertical.comb, Vertical.r, fun = "count")
Vertical.r = Vertical.r > 0
spplot(Vertical.r,main="Environnemental backgound (0-200m)",xlab="Temperature",ylab="Salinity")
```

<img src="SDM_files/figure-gfm/unnamed-chunk-14-3.png" style="display: block; margin: auto;" />

### Download occurrences data

Species occurrences data can be obtained from 5 global biogeographic
database (see *step 2* in model framework section in Ben Rais Lasram et
al (2020)): 
 * OBIS (Ocean Biogeographic Information System) <http://www.iobis.org/> 
 * GBIF (Global Biodiversity Information Facility) <http://www.gbif.org/> 
 * iNaturalist (A Community for Naturalists) <http://www.inaturalist.org> 
 * VertNet (vertebrate biodiversity networks) <http://vertnet.org/> 
 * Ecoengine (UC Berkeley’s Natural History Data)‘<https://ecoengine.berkeley.edu/>

``` r
data <- data.frame(Species=Species,Vertical_habitat)
  
# Download occurrences from OBIS  

occOBIS <- robis::occurrence(scientificname = data[1, "Species"]) # Download, be carefull of the firewall
```

    ## Retrieved 5000 records of approximately 8317 (60%) Retrieved 8317 records of approximately 8317 (100%)

``` r
if (dim(occOBIS)[1] > 0) {
occOBIS <- cbind(occOBIS[, c("decimalLongitude", "decimalLatitude", "species")],
                 rep("obis", nrow(occOBIS)),
                 occOBIS[, "eventDate"],
                 occOBIS[, c("phylum", "class", "order", "family", "genus")])
names(occOBIS) <- c("longitude", "latitude", "name", "prov", "date", "phylum", "class", "order", "family", "genus")
occOBIS$prov <- as.character(occOBIS$prov)
occOBIS$year <- as.numeric(substr(as.character(occOBIS$date), 1, 4))
}
  
# Download occurrences from GBIF and other databases
  
occOther <- occ(query=as.character(data[1, 1]),
                from=c("gbif", "bison", "inat", "ebird", "ecoengine", "vertnet"),
                has_coords = T,limit = 100000) # Download

occOther <- occ2df(occOther, "all")
occOther <- data.frame(longitude = as.numeric(occOther$data[, 2]),
                       latitude = as.numeric(occOther$data[, 3]),
                       occOther$data[, c(1, 4)],
                       occOther$data[, 5], 
                       as.numeric(substr(as.character(occOther$data[, 5]), 1, 4)))
names(occOther) <- c("longitude", "latitude", "name", "prov", "date", "year") # Keep the same name as the one of OCCOBIS but with less parameters

# Merge all occurrences data
  
if(nrow(occOBIS) == 0){OCC <- occOther}
if(nrow(occOther) == 0){OCC <- occOBIS}
if (nrow(occOBIS) > 0 & nrow(occOther) > 0) {
occOther[,c("phylum", "class", "order", "family", "genus")] <- occOBIS[1, c("phylum", "class", "order", "family", "genus")]
occOther <- occOther[, c("longitude", "latitude", "name", "prov", "date", "phylum", "class", "order", "family", "genus", "year")]
OCC <- rbind(occOBIS[, -5],occOther[, -5])} # Merge
OCC <- OCC[which(OCC$name == as.character(data[1, 1])), ]
  
# Create spatial object

idx<-which(is.na(OCC$longitude) | is.na(OCC$latitude))
if(length(idx)>0){OCC<-OCC[-idx,]}
coordinates(OCC) <-~ longitude + latitude
OCC@data[, 1] <- as.character(OCC@data[, 1])
  
# Remove duplicated records 
  
OCC <- remove.duplicates(OCC)
  
# Remove records with zero in Lat and Long
  
zero.coord <- which(coordinates(OCC)[, 1] == 0 & coordinates(OCC)[, 2] == 0)
if(length(zero.coord) > 0){
   OCC <- OCC[ - zero.coord, ]
 }
  
# Delete occurences located on the continent

land <- extract(Temperature1, OCC) # Extract the land of the global studied area 
OCC <- OCC[ - which(is.na(land)), ] # Delete the land occurences

# write shapefile 
  
writeOGR(obj = OCC, layer = as.character(data[1, 1]),
         dsn = "Shapefiles", driver = "ESRI Shapefile",
         overwrite_layer = T) # Write a shapefile of the occurences for each species 

# Remove NA from the occurences

OCC@data <- na.omit(OCC@data)

#Plot occurrences

plot(getMap(resolution = "coarse"),col="gray")
plot(OCC,add=T,col=2,cex=0.3,pch=15)
```

<img src="SDM_files/figure-gfm/unnamed-chunk-15-1.png" style="display: block; margin: auto;" />

### Match up

A spatiotemporal match-up between climatic climatologies and species
occurrences is realised taking into account, the geographic coordinates
of occurrences as well as their corresponding decade and the right
vertical layer that corresponds to the vertical habitat of the focal
species (see *step 3* in model framework section in Ben Rais Lasram et
al (2020)).

``` r
# Load occurences from shapefile
    
OCC = readOGR(paste("Shapefiles/", data[1, 1], ".shp", sep = ""), verbose = F) 

# Aggregating each occurences depending on the climatic time periods choosen
    
OCC@data$decade <- NA
OCC@data$decade <- ifelse(OCC@data$year >= 1955 & OCC@data$year <= 1964, 1, OCC@data$decade)
OCC@data$decade <- ifelse(OCC@data$year >= 1965 & OCC@data$year <= 1974, 2, OCC@data$decade)
OCC@data$decade <- ifelse(OCC@data$year >= 1975 & OCC@data$year <= 1984, 3, OCC@data$decade)
OCC@data$decade <- ifelse(OCC@data$year >= 1985 & OCC@data$year <= 1994, 4, OCC@data$decade)
OCC@data$decade <- ifelse(OCC@data$year >= 1995 & OCC@data$year <= 2004, 5, OCC@data$decade)
OCC@data$decade <- ifelse(OCC@data$year >= 2004 & OCC@data$year <= 2012, 6, OCC@data$decade)
if(length(which(is.na(OCC@data$decade))) > 0){OCC = OCC[-which(is.na(OCC@data$decade)), ]}
OCC@data$Temperature <- NA
OCC@data$Salinity <- NA
    
# Set the Vlayer object depending on the vertical habitat of the focal species 
    
Vlayer <- NULL
if (data[1, "Vertical_habitat"] == "Benthic"){Vlayer <- 1}
if (data[1, "Vertical_habitat"] == "Demersal"){Vlayer <- 1}
if (data[1, "Vertical_habitat"] == "Pelagic"){Vlayer <- 2}
if (data[1, "Vertical_habitat"] == "Benthopelagic"){Vlayer <- 3}
    
# Find Temperature and salinity for each occurences depending on it's decade and vertical positionning
    
for (j in unique(OCC@data$decade)){
  idx = which(OCC@data$decad == j)
      OCC@data[idx, "Temperature"] <- do.call(what = "extract",
                                              args = list(y = OCC[which(OCC@data$decad == j), ],
                                              x    = eval(parse(text = paste("Temperature", j, "[[", Vlayer, "]]", sep = "")))))
      OCC@data[idx, "Salinity"] <- do.call(what = "extract", 
                                           args = list(y = OCC[which(OCC@data$decad == j), ],
                                           x    = eval(parse(text = paste("Salinity", j, "[[", Vlayer, "]]", sep = "")))))
}
    
# Update the shapefile of the occurences for each species   
writeOGR(obj = OCC, layer = as.character(data[1, 1]),
           dsn = "Shapefiles", driver = "ESRI Shapefile",
           overwrite_layer = T) 
```

### Bioclimatic Envelope Modelling

The bioclimatic envelope modelling procedure is descibed in *step 4* in
model framework section in Ben Rais Lasram et al (2020).

``` r
# Create empty object to save validation results

Validation <- NULL

#Select the species's environmental background acoording to its vertical habitat 
 
Vlayer  <- NULL
if (data[1, "Vertical_habitat"] == "Benthic"){Vlayer <- 1       ;background <- Bottom.r}
if (data[1, "Vertical_habitat"] == "Demersal"){Vlayer <- 1      ;background <- Bottom.r}
if (data[1, "Vertical_habitat"] == "Pelagic"){Vlayer <- 2       ;background <- Surface.r}
if (data[1, "Vertical_habitat"] == "Benthopelagic"){Vlayer <- 3 ;background <- Vertical.r}
  
# Load the occurrences with matchups from shapefile
  
OCC = readOGR(paste("Shapefiles/", Species, ".shp", sep = ""), verbose = F)
names(OCC)[which(names(OCC) %in% c("Temprtr" , "Salinty"))] <- c("Temperature","Salinity")
  
# Create a map of all the combinaison of temperature and salinity encounter by the speces and put all the values in a table
  
XY <- OCC@data[, c("Temperature", "Salinity")]
coordinates(XY) <-~ Temperature + Salinity
XY.r <- rasterize(XY, background, fun = "count");XY.r <- XY.r > 0
Nb.presence <- length(which(values(XY.r) > 0))
Presence <- coordinates(XY.r)[which(values(XY.r) > 0), ]
colnames(Presence) <- c("Temperature", "Salinity")


# Pseudo-absence simulation
  
XY.quantile <- OCC@data[,c("Temperature","Salinity")]
XY.quantile <- XY.quantile[which(XY.quantile[, 1] > quantile(XY.quantile[, 1], 0.001) &
                                 XY.quantile[, 1] < quantile(XY.quantile[, 1], 0.999) & 
                                 XY.quantile[, 2] > quantile(XY.quantile[, 2], 0.001) & 
                                 XY.quantile[, 2] < quantile(XY.quantile[, 2], 0.999)), ]
coordinates(XY.quantile) <-~ Temperature + Salinity
hull <- rasterize(gConvexHull(XY.quantile), background)
values(hull) <- ifelse(is.na(values(hull)), 1, NA)
hull <- background * hull
XY.hull <- coordinates(hull)[which(is.na(values(hull)) == FALSE), ]

if(Nb.presence < 1000){Nb.absence <- 1000} else{Nb.absence  <- Nb.presence} # Number of pseudo-absence

sampledCells <- sample(1 : nrow(XY.hull), Nb.absence)
pseudoAbs <- XY.hull[sampledCells, ]
colnames(pseudoAbs) <- c("Temperature", "Salinity")

plot(background, col = 0.5, main = "Presences (red) and Pseudo-absences (green)", xlab = "Temperature", ylab = "Salinity")
plot(hull, add = T, col = 1, colNA = "transparent")
points(Presence, pch = 15, cex = 0.3, col = 2)
points(pseudoAbs, pch = 15, cex = 0.3, col = 3)
```

<img src="SDM_files/figure-gfm/unnamed-chunk-17-1.png" style="display: block; margin: auto;" />

``` r
# k-fold cross validation using Boyce index, TSS and ROC
  
GlobalEnv <- stack(Temperature6[[Vlayer]], Salinity6[[Vlayer]])
names(GlobalEnv) <- c("Temperature", "Salinity")
GlobalEnv <- stack(aggregate(GlobalEnv,fact = 3))
                            
PA <- as.data.frame(rbind(cbind(Presence, resp = rep(1, Nb.presence)), cbind(pseudoAbs, resp = rep(0, Nb.absence))))
PA$x <- 1:nrow(PA)
PA$y <- 1:nrow(PA)
  
# Formating data for biomod

CalibData <- BIOMOD_FormatingData(resp.var    = PA$resp, 
                                    expl.var  = PA[, c("Temperature", "Salinity")],
                                    resp.xy   = PA[, c("x","y")],
                                    resp.name = as.character(Species))

# Creating the SplitTable, to split the data set in k part
                                
DataSplitTable <- BIOMOD_cv(CalibData, k = k, rep = 1, balance = "presences")

# Models calibration
  
Biomod_cal = BIOMOD_Modeling(data             = CalibData,
                             models           = models, 
                             models.eval.meth = c("TSS", "ROC"),
                             DataSplitTable   = DataSplitTable,
                             do.full.models   = FALSE,
                             SaveObj          = FALSE) 

eval <- get_evaluations(Biomod_cal, as.data.frame = T)
eval <- cbind(eval[, -4], Species = rep(Species, nrow(eval))) # Exctract the eval values (TSS and ROC)
  
# Update the validation file
  
Validation <- rbind(Validation, eval)
  
# Projection of the models on the current global climatic data

gc() # Warning : This step requires a lot of  memory, gc() to "clean" memory usage

Biomod_current_global <- BIOMOD_Projection(modeling.output = Biomod_cal,
                                           new.env         = GlobalEnv, 
                                           selected.models = Biomod_cal@models.computed[1:(k * length(models))], 
                                           proj.name       = paste(Species, "_Current_Global", sep = ""),
                                           silent          = TRUE); gc()

Biomod_current_global <- Biomod_current_global@proj@val

# Response curves
  
new.env <- coordinates(background)[which(values(background) > 0),]
colnames(new.env) <- c("Temperature", "Salinity")

gc()

CurveFull <- BIOMOD_Projection(Biomod_cal, 
                               new.env = new.env, 
                               selected.models = Biomod_cal@models.computed[grep(Biomod_cal@models.computed,pattern = "4")], 
                               proj.name = "Full_Current_Global",
                               silent=TRUE);gc()

CurveFull <- as.data.frame(cbind(new.env,CurveFull@proj@val[, , 1, 1]/1000))
coordinates(CurveFull) <-~ Temperature+Salinity
CurveFull <- as(CurveFull,"SpatialPixelsDataFrame") # Use to plot the Full responce curve
gc()  

# Calculate CBI (Boyce index) for each repetition

for (j in models){
    for (kfold in 1 : k){
        idModel <- grep(names(Biomod_current_global), pattern = paste("RUN", kfold,"_", j, sep=""))
        val.data <- PA[DataSplitTable[, kfold] == FALSE, c("Temperature", "Salinity", "resp")]
        val.data <- val.data[val.data$resp == 1, -3]
        val.data <- BIOMOD_Projection(modeling.output = Biomod_cal, # Biomod projection on the climatic map
                                      new.env         = val.data, 
                                      selected.models = Biomod_cal@models.computed[idModel], 
                                      proj.name       = paste(Biomod_cal@models.computed[idModel], "_Current_Global", sep = ""),
                                      silent          = TRUE) ;gc()
        CBI <- ecospat.boyce (fit      = na.omit(values(Biomod_current_global[[idModel]])/1000), 
                              obs      = val.data@proj@val[, 1, 1, 1]/1000, 
                              nclass   = 0, 
                              window.w ="default", 
                              res      = 100, 
                              PEplot   = FALSE)
        Validation <- rbind( Validation, 
                             data.frame(Model.name   = paste("RUN", kfold, "_", j, sep = ""),  
                                        Eval.metric  = "CBI",  
                                        Testing.data = CBI$Spearman.cor, 
                                        Cutoff       = NA, 
                                        Sensitivity  = NA,
                                        Specificity  = NA,  
                                        Species      = Species))
    }
}

gc()

# Current Predictions (local scale)
  
CurrentEnv <- stack(TemperatureL[[Vlayer]],SalinityL[[Vlayer]])
names(CurrentEnv) <- c("Temperature", "Salinity")
gc()  

Biomod_current <- BIOMOD_Projection(Biomod_cal, 
                                    new.env         = CurrentEnv, 
                                    proj.name       = paste(Species, "_Current_local", sep = ""),
                                    selected.models = Biomod_cal@models.computed[grep(Biomod_cal@models.computed, pattern = "4")],
                                    silent          = TRUE) 

gc()

current_distribution <- as(Biomod_current@proj@val, "SpatialPixelsDataFrame")

# Projection of GFDL-ESM2G_RCP2.6

FutureEnv <- stack(Temperature_GFDL_ESM2G_RCP2.6[[Vlayer]], Salinity_GFDL_ESM2G_RCP2.6[[Vlayer]])
names(FutureEnv) <- c("Temperature", "Salinity")
gc()

Biomod_future <- BIOMOD_Projection(Biomod_cal, 
                                   new.env         = FutureEnv, 
                                   proj.name       = paste(Species,"_GFDL-ESM2G_RCP2.6", sep = ""), 
                                   selected.models = Biomod_cal@models.computed[grep(Biomod_cal@models.computed, pattern = "4")],
                                   silent          = TRUE) ;gc()

future_distribution <- as(Biomod_future@proj@val, "SpatialPixelsDataFrame")
```

### Bioclimatic Envelope Model averaging

Ensemble suitability maps are obtained by calculated the average
suitability among all predictions weighted by the CBI to account for
model-based uncertainty (see *step 4* in model framework section in Ben
Rais Lasram et al (2020)).These maps can then be transformed into binary
maps using the probability threshold optimizing the TSS. In addition,
uncertainty maps are computed by calculating the standard deviation
among
predictions.

``` r
for (i in 1:4){Validation[grep(Validation$Model.name,pattern = as.character(i)), "Rep"] <- i}
for (i in models){Validation[grep(Validation$Model.name,pattern = as.character(i)), "Algorithm"] <- i} 
Validation <- Validation[which(Validation$Rep != 4), c(2:9)]
Validation <- Validation[, c("Eval.metric", "Testing.data", "Cutoff", "Rep", "Species", "Algorithm")]
Validation$Eval.metric <- as.factor(Validation$Eval.metric)
Validation$Species <- as.factor(Validation$Species)
Validation$Algorithm <- as.factor(Validation$Algorithm)
bem.tmp <- Validation[which(Validation$Species == Species), ] 
tmp.cbi <-  bem.tmp[which(bem.tmp$Eval.metric == "CBI"), ]
Mean.tmpcbi <- aggregate(tmp.cbi[, 2], list(tmp.cbi$Algorithm), mean)

# Asking for the mean of the CBI of all the 3 cross validation to be higher than 0.5

sel.models <- names(which(table(Mean.tmpcbi[which(Mean.tmpcbi$x > 0.5), "Group.1"]) == 1)) 
sel.cbi <-  tmp.cbi[which(tmp.cbi$Algorithm %in% sel.models), c("Algorithm", "Testing.data")]
    
boxplot(Testing.data~Algorithm, # Show the selected algorithm(s)
        data = sel.cbi,
        ylab = "CBI",
        xlab = "Algorithms")
```

<img src="SDM_files/figure-gfm/unnamed-chunk-18-1.png" style="display: block; margin: auto;" />

``` r
cbi <- aggregate(sel.cbi$Testing.data, by = list(sel.cbi$Algorithm), FUN = mean)    
cbi <- cbi$x; names(cbi) <- sel.models
    
# Cutoff setting from TSS 

sel.cutoff <- na.omit(Validation[which(Validation$Species == Species & 
                                         Validation$Algorithm %in% sel.models &  
                                         Validation$Eval.metric %in% c("TSS", "CBI")), ])
tss <- aggregate(sel.cutoff$Testing.data, by = list(sel.cutoff$Algorithm), FUN = mean)
tss <- tss$x; names(tss) <- sel.models
cutoff <- aggregate(sel.cutoff$Cutoff, by = list(sel.cutoff$Algorithm), FUN = mean)
cutoff <- cutoff$x; names(cutoff) <- sel.models
cutoff <- weighted.mean(cutoff, w = tss)/1000 # Cutoff for thr binary map from the TSS

# Find the selected model(s)    
sel.models.grep = NULL
for (z in 1 : NROW(sel.models)){
  grep1 <- grep(sel.models[z], names(current_distribution@data), value = TRUE)
  sel.models.grep <- rbind(sel.models.grep, data.frame(grep1))
}

    
current_distribution@data <- as.data.frame(current_distribution@data[, as.character(unlist(sel.models.grep))])
future_distribution@data <- as.data.frame(future_distribution@data[, as.character(unlist(sel.models.grep))])
response <- CurveFull
response@data <- as.data.frame(CurveFull@data[, sel.models])

# Maps of mean and standard.deviation of probability and binary result (current)
averaging.current <- current_distribution[, 1]
averaging.current@data[, 1] <- apply(current_distribution@data, 1, FUN = function(x) weighted.mean(x, w = tss))/1000 
averaging.current@data[, 2] <- apply(current_distribution@data, 1, FUN = sd)/1000 # standard deviation map
if(all(is.na(averaging.current@data[, 2]))){averaging.current@data[, 2] <- 0}
averaging.current@data[, 3] <- ifelse(averaging.current@data[, 1] > cutoff, 1, 0) # binary map
names(averaging.current) <- c("Mean", "Standard.deviation", "Binary")
spplot(averaging.current, main = "Current bioclimatic envelopes")
```

<img src="SDM_files/figure-gfm/unnamed-chunk-18-2.png" style="display: block; margin: auto;" />

``` r
# Maps of mean and standard.deviation of probability and binary result (RCP 2.6) 
averaging.future <- future_distribution[, 1]
averaging.future@data[, 1] <- apply(future_distribution@data, 1, FUN = function(x) weighted.mean(x, w = tss))/1000 
averaging.future@data[, 2] <- apply(future_distribution@data, 1, FUN = sd)/1000 # Standard deviation map
averaging.future@data[, 3] <- ifelse(averaging.future@data[, 1] > cutoff, 1, 0) # Binary map
names(averaging.future) <- c("Mean", "Standard.deviation", "Binary")
spplot(averaging.future,main = "Future bioclimatic envelopes (RCP 2.6)")
```

<img src="SDM_files/figure-gfm/unnamed-chunk-18-3.png" style="display: block; margin: auto;" />

``` r
# Maps of mean and standard.deviation of probability and binary result (current)
averaging.response <- response[, 1]
averaging.response@data[, 1] <- apply(response@data,1, FUN = function(x) weighted.mean(x, w = tss)) 
averaging.response@data[, 2] <- apply(response@data,1, FUN = sd) # standard deviation map
if(all(is.na(averaging.response@data[, 2]))){averaging.response@data[, 2] <- 0}
averaging.response@data[, 3] <- ifelse(averaging.response@data[, 1] > cutoff, 1, 0) # binary map
names(averaging.response) <- c("Mean", "Standard.deviation", "Binary")
spplot(averaging.response,main = "Response curves", xlab = "Temperature", ylab = "Salinity")
```

<img src="SDM_files/figure-gfm/unnamed-chunk-18-4.png" style="display: block; margin: auto;" />

``` r
# Save the output 
dir.create("Outputs")
save(averaging.response,file = paste("Outputs/Response_", Species,".RData", sep = "")) 
save(averaging.current,file = paste("Outputs/Bioclimatic_Envelope_Current_", Species, ".RData", sep = "")) 
save(averaging.future, file = paste("Outputs/Bioclimatic_Envelope_Future_", Species, ".RData", sep = "")) 
```

### Habitat modelling

The habitat models, used only for benthic and demersal species (see
*step 5* in model framework section in Ben Rais Lasram et al (2020))

``` r
if(Vertical_habitat %in% c("Benthic","Demersal")){
  # Create empty file to save validation results
  
  Validation_habitat <- NULL
  r <- ordi[[1]] 
  OCC = readOGR(paste("Shapefiles/", Species, ".shp", sep = ""), verbose = F)
  OCC <- rasterize(OCC, r, fun = "count", field = rep(1, nrow(OCC)), background = 0) ;OCC <- mask(OCC, r)
  Presence <- coordinates(OCC)[which(values(OCC) > 0), ] ;OCC$resp.var <- rep(1, nrow(OCC))
  if (nrow(Presence) < 1000) {Nb.absence <- 1000} else{Nb.absence <- nrow(Presence)}
   
  # Formating  data for biomod
    
  CalibData <- BIOMOD_FormatingData(resp.var       = rep(1, nrow(Presence)),
                                    expl.var       = ordi, 
                                    resp.xy        = Presence,
                                    PA.nb.rep      = 1,
                                    PA.nb.absences = Nb.absence,
                                    PA.strategy    = 'sre',
                                    resp.name      = as.character(Species)) 
    
  CalibData@data.species <- ifelse(is.na(CalibData@data.species), 0, 1)
  
  # SplitTable for the k cross validation
  
  DataSplitTable <- BIOMOD_cv(CalibData, k = k, rep = 1, balance = "presences") 
    
  # Modelling
    
  Biomod_cal = BIOMOD_Modeling(data             = CalibData, 
                               models           = models, 
                               models.eval.meth = c("TSS", "ROC"),
                               DataSplitTable   = DataSplitTable,
                               do.full.models   = FALSE,
                               SaveObj          = FALSE) 
    
  eval <- get_evaluations(Biomod_cal, as.data.frame = T)
  eval <- cbind(eval[, -4], Species = rep(Species, nrow(eval)))
  Validation_habitat <- rbind(Validation_habitat, eval)
  gc()  
  
  # Projection of the models using habitat ordination axes map
  

  Biomod_current_habitat <- BIOMOD_Projection(Biomod_cal, 
                                              new.env   = ordi, 
                                              proj.name = paste0(Species, "_Current_habitat"),
                                              silent    = TRUE)
  gc()
  Biomod_current_habitat <- Biomod_current_habitat@proj@val
  load(Biomod_cal@models.prediction@link)
  selected.models <- which(substr(x     = names(Biomod_current_habitat),
                                  start = nchar(as.character(Species)) + 6,
                                  stop  = nchar(as.character(Species)) + 9) != "RUN4")
  
  # Calculate CBI (Boyce index) for each repetition

  for(i in selected.models){
      mod <- substr(x     = names(Biomod_current_habitat)[i], 
                    start = nchar(as.character(Species)) + 11, 
                    stop  = 1000000L)
      rep <- substr(x     = names(Biomod_current_habitat)[i],
                    start = nchar(as.character(Species)) + 6,
                    stop  = nchar(as.character(Species)) + 9)
      
      CBI <- ecospat.boyce(fit      = na.omit(values(Biomod_current_habitat[[i]])/1000), 
                           obs      = models.prediction[, mod, rep,]/1000, 
                           nclass   = 0, 
                           window.w = "default", 
                           res      = 100, 
                           PEplot   = FALSE)
      Validation_habitat <- rbind(Validation_habitat, 
                            data.frame(Model.name   = paste(mod, "_", rep, "_PA1", sep = ""),  
                                       Eval.metric  = "CBI",  
                                       Testing.data = CBI$Spearman.cor, 
                                       Cutoff       = NA, 
                                       Sensitivity  = NA,
                                       Specificity  = NA,  
                                       Species      = Species))
  }
    
  # Current Predictions 
    
  fullModel <- which(substr(x     = names(Biomod_current_habitat), 
                            start = nchar(as.character(Species)) + 6,
                            stop  = nchar(as.character(Species)) + 9) == "RUN4")
  Biomod_current_local <- as(Biomod_current_habitat[[fullModel]], "SpatialPixelsDataFrame")
    
  save(Validation_habitat, file = "Outputs/Validation_habitat.RData")
}
```

### Habitat model averaging

Ensemble suitability maps are obtained by calculated the average
suitability among all predictions.

``` r
if(Vertical_habitat %in% c("Benthic","Demersal")) {
  
  for (i in 1 : 4) {
      Validation_habitat[grep(Validation_habitat$Model.name,pattern = as.character(i)), "Rep"] <- i
  }
  for (i in models) {
     Validation_habitat[grep(Validation_habitat$Model.name,pattern = as.character(i)), "Algorithm"] <- i
  } 
  tmp.cbi  <-  na.omit(Validation_habitat[which(Validation_habitat$Species == Species &
                                                  Validation_habitat$Eval.metric == "CBI"), 
                                            c("Testing.data", "Algorithm")])# select only CBI results
  tmp.tss  <-  na.omit(Validation_habitat[which(Validation_habitat$Species == Species &
                                                Validation_habitat$Eval.metric == "TSS"), 
                                            c("Testing.data", "Algorithm")])
  Mean.tmptss <- aggregate(tmp.tss$Testing.data, list(tmp.tss$Algorithm), mean)
  
  # Asking for the mean of the TSS of all the 3 cross validation to be higher than 0.7
  
  sel.models <- names(which(table(Mean.tmptss[which(Mean.tmptss$x>0.7), "Group.1"]) == 1)) 
  check<-length(sel.models) > 0
  if (check){sel.cbi <-  tmp.cbi[which(tmp.cbi$Algorithm %in% sel.models), c("Algorithm", "Testing.data")]
       cbi <- aggregate(sel.cbi$Testing.data, by = list(sel.cbi$Algorithm), FUN = mean) 
       cbi <- cbi$x; names(cbi) <- sel.models
       sel.tss <-  tmp.tss[which(tmp.tss$Algorithm %in% sel.models), c("Algorithm", "Testing.data")]
       tss <- aggregate(sel.tss$Testing.data,by = list(sel.tss$Algorithm), FUN = mean)
       tss <- tss$x; names(tss) <- sel.models           
       sel.cutoff <- na.omit(Validation_habitat[which(Validation_habitat$Species == Species & 
                                              Validation_habitat$Algorithm %in% sel.models &  
                                              Validation_habitat$Eval.metric %in% c("TSS", "CBI")), ])
       cutoff <- aggregate(sel.cutoff$Cutoff,by = list(sel.cutoff$Algorithm), FUN = mean)
       cutoff <- cutoff$x; names(cutoff) <- sel.models
       cutoff <- weighted.mean(cutoff, w = tss)/1000 # Cutoff for the binary map from the TSS
       map <-  Biomod_current_local
       names(map) <- substr(x = names(Biomod_current_local), start = nchar(as.character(Species)) + 11, stop = 1000000L)
       map <- map[, sel.models]; averaging.map <- map[, 1]
       
       # Map of mean probability
       
       averaging.map@data[, 1] <- apply(map@data, 1, FUN = function(x)weighted.mean(x, w = tss))/1000 
       
       # Standard deviation map
       
       averaging.map@data[, 2] <- apply(map@data, 1, FUN = sd)/1000
       averaging.map@data[, 3] <- ifelse(averaging.map@data[, 1] > cutoff, 1, 0) # Binary map
       sel.cbi[,1] <- factor(sel.cbi[, 1])
       names(averaging.map) <- c("Mean", "Standard.deviation", "Binary")
       spplot(averaging.map,main = "habitat suitability")
       # Boxplot to show the selected habitat algorithms
       
       boxplot(Testing.data ~ Algorithm,data = sel.tss,ylab = "", xlab = " Selected algorithms") 
       if(length(sel.models) == 1){axis(1, labels = sel.models, at = 1)}
       save(averaging.map, file = paste("Outputs/Habitat_", Species, ".RData", sep = ""))
      if (check == FALSE){print("No habitat model was selected because they do not meet the defined validation criteria")}
  }
}
```

![](SDM_files/figure-gfm/unnamed-chunk-20-1.png)<!-- -->

![](SDM_files/figure-gfm/unnamed-chunk-21-1.png)<!-- -->

### Hierarchical filtering

For benthic and demersal species, bioclimatic envelope and habitat maps
are combined. See *step 6* in model framework section in Ben Rais Lasram
et al
(2020).

``` r
if(Vertical_habitat %in% c("Benthic","Demersal") & exists("averaging.map")) {
  
  # Current
  
  mask<-raster(averaging.map[,3])           
  averaging.current <- averaging.current[, c(1,3)]
  averaging.current <- crop(averaging.current, extent(ordi), snap = "near")
  averaging.current <- resample(stack(averaging.current), mask, method = "ngb")
  averaging.current.prob <-  averaging.current[[1]] * mask # Averaging with probability ranging from 0 to 1
  averaging.current.bin  <-  averaging.current[[2]] * mask # Binary averaging
  spplot(averaging.current.prob, main = "Distribution (baseline)")

            
  # RCP         
  
  averaging.future <- averaging.future[, c(1,3)]
  averaging.future <- crop(averaging.future, extent(ordi), snap = "near")
  averaging.future <- resample(stack(averaging.future), mask, method = "ngb")
  averaging.future1.prob <-  averaging.future[[1]] * mask # Averaging with probability ranging from 0 to 1
  averaging.future1.bin  <-  averaging.future[[2]] * mask # Binary averaging
  save(averaging.current.bin, file = paste("Outputs/Current_Distribution_", Species, ".RData", sep = ""))
  save(averaging.future1.bin, file = paste("Outputs/Future_Distribution_", Species, ".RData", sep = ""))
  spplot(averaging.future1.prob, main = "Distribution (projection GFDL-ESM2G RCP2.6)")
}
```

![](SDM_files/figure-gfm/unnamed-chunk-22-1.png)<!-- -->

![](SDM_files/figure-gfm/unnamed-chunk-23-1.png)<!-- -->

