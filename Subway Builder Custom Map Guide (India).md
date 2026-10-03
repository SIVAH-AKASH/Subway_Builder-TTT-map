# Beginner's Guide: Building a Custom Map in India for the Subway Builder game

## A. What this document covers

It contains the process to transform a real-world geographic area in India into a custom `city` in Subway Builder (using the community built `Subway-Builder-Modded`'s [`Depot`](https://subwaybuildermodded.com/depot/) library). It's aimed at people who have **basic familiarity with terminals and tinkering with computers but don't have a programming background**.

The entire process is separated by stages to make it easier to follow and understand the process. The process is generalized to be adaptable to any region in India, but you might find it helpful to also reference the mod files created for the Tenkasi/Tirunelveli/Thoothukudi area in Tamil Nadu, India (values used for this region are provided below) that is available under releases section of this repo. Placeholders to be replaced with your region's values are placed within angle brackets (Ex: `<CITY_CODE>`, `<BBOX_EXTRACT>`), which will need to be replaced with the respective values for your own region.

**Credits:**

1. **Creators and Maintainers of Subway Builder Modded** https://subwaybuildermodded.com

2. **OpenStreetMap (OSM) and OSM contributors** https://www.openstreetmap.org/copyright

3. **Sam Asher, Tobias Lunt, Ryu Matsuura and Paul Novosad**, "Development Research at High Geographic Resolution: An Analysis of Night-Lights, Firms, and Poverty in India Using the SHRUG Open Data Platform", *The World Bank Economic Review*, Volume 35, Issue 4, November 2021, Pages 845–871, https://doi.org/10.1093/wber/lhab003

4. **European Space Agency** (2024). Copernicus Global Digital Elevation Model. Distributed by **OpenTopography**. https://doi.org/10.5069/G9028PQB

**Note:** The author of this guide used a LLM's assistance for creating the mod files.

----

## B. Prerequisites and high-level process overview

The guide assumes the below are being used to follow the process:

- [ ] A Windows device
- [ ] A Virtual Machine (VM) on that device running Debian Server (recommended 16GB RAM, 8 cores and 20GB drive space for the VM)
- [ ] A code/text editor (Ex: Notepad, Notepad++)
- [ ] An SSH terminal (Ex: Command Prompt, Tabby)
- [ ] Willingness to learn by trial-and-error :)
- [ ] Understanding that the guide covers the actual mod process in detail but might require you to do additional self-research for items specific to your particular map/device and for any general steps not directly related to building the mod (like installing the VM)

It's possible to run the entire process within Linux itself, but the guide will follow this split approach - the Windows device runs the game, is used for testing mods and running any mod scripts that don't need to be run on Linux specifically. Some steps that can only be run on Linux (map/data/routing processing using depot/planetiler/OSRM), will be run on a VM on the same device that has Debian installed on it. An SSH connection is made to the VM from Windows to send commands and send/receive files.

Our working directory will be a Desktop folder for ease of access and which will also act as a backup. We will move the final mod files to the appropriate game folder (`%APPDATA%\metro-maker4\...`) as needed.

Below is the folder structure and the files that will live on the Windows side:

```
SB_mod_project\
├── 01_source\                                   Raw source files used to make the final mod files
│   ├── 03_census\
│   │   ├── <district-x_file_name>.xlsx          Census file at town, village and ward level
│   │   └── 01_shrug\
│   │       └── village_modified.gpkg            SHRUG geopackage
│   ├── 02_elevation\
│   │   └── output_hh.tif                        DSM for `bbox`
│   └── 01_osm\
│       └── map.osm.pbf                          OSM extract for `bbox`
├── 02_intermediate\                             Regeneratable scripts used for processing files
└── 03_mod\                                      Final deployed mod files
    └── <CITY_CODE_LOWER>-mod\
        ├── index.js                             Mod's code
        ├── manifest.json                        Mod's metadata
        ├── data\
        │   └── <CITY_CODE>\
        │       ├── demand_data.json.gz          Residence/job locations and commuting times
        │       ├── roads.geojson.gz             Roads locations
        │       └── runways_taxiways.geojson.gz
        ├── tiles\
        │   ├── boundaries.pmtiles               District, Taluk and Village boundaries
        │   ├── contours.pmtiles                 Used for contour lines
        │   ├── elevation.pmtiles                Used for hillshade and 3D terrain
        │   ├── hypso_colored.pmtiles            Used for hypsometric tinting
        │   ├── <CITY_CODE>.pmtiles              Main vector tiles containing most data for the `bbox`
        │   └── <CITY_CODE>_foundations.pmtiles  Buildings + water bodies depth data used for determining depth at which stations and tracks can be laid
        └── tools\
            ├── pmtiles.exe                      Serves the vector tiles to the game through a server
            └── tile_server.bat                  Script to simplify running the server
```

General References: ([1](https://www.subwaybuilder.com/docs/guides/custom-cities)) and ([2](https://www.subwaybuilder.com/docs/api-reference/map))

----

## C. Progress checklist

- [ ] Stage 1 - Initial Setup
- [ ] Stage 2 - Downloading OSM data for the `bbox`
- [ ] Stage 3 - Building the base map data
- [ ] Stage 4 - Setting up OSRM
- [ ] Stage 5 - Pulling census and creating demand data to simulate commuting patterns
- [ ] Stage 6 - Testing mod files in-game
- [ ] Stage 7 - Elevation map layers (Optional)
- [ ] Stage 8 - Land cover and land use map layers (Optional)
- [ ] Stage 9 - Scale Bar and Administrative boundaries (Optional)
- [ ] Stage 10 - Sharing or publishing the mod files

----

## D. Placeholders

| Placeholder                      | Meaning                                         | Example (values used for TTT) |
| -------------------------------- | ----------------------------------------------- | ----------------------------- |
| `<VM_IP>`                        | Debian VM's IP address                          |                               |
| `<VM_USER>`                      | Username on Debian                              |                               |
| `<CITY_CODE>`                    | Unique code for the city                        |                               |
| `<CITY_CODE_LOWER>`              | Lowercased `<CITY_CODE>`                        |                               |
| `<BBOX_EXTRACT>`                 | `[West, South, East, North]` extents of the map | `77.15, 8.6, 78.25, 9.05`     |
| `<BBOX_EXTRACT_TYPE-2>`          | `[South, West, North, East]` extents of the map | `8.6, 77.15, 9.05, 78.25`     |
| `<POPULATION>`                   | Population                                      | `2808357`                     |
| `<ZOOM>`                         | Initial camera zoom level                       | `13.5`                        |
| `<INITIAL_LAT>`, `<INITIAL_LON>` | Initial camera coordinates                      | `8.7312234`, `77.7083360`     |

----

## E. Terms to be familiar with

- **Bounding box (bbox):** A rectangular area representing a geographical area

- **Vector tiles:** Small square areas (`tiles`) containing map data per zoom level, so the game only loads the areas it needs instead of the whole map area. `.pmtiles` is a specific format for storing them

- **Dev console:** The developer tools panel (opens with F12 / "Ctrl+Shift+L") which allows for checking errors and running code (JavaScript) directly in the game

- **MapLibre:** An open-source library that the game's map is built on using `.pmtiles` files

- **deck.gl:** Another open-source library that the game uses specifically for in-game elements like tracks/stations/routes, which renders independently of the `MapLibre` built map.

- **Depot:** The mod is primarily built using this library's tools

- **planetiler:** The tool used by `depot` to convert raw map data (roads, buildings, etc.) into vector tiles.

- **tile-join:** Allows for merging/editing tiles when `planetiler` doesn't support a map feature

- **Open Source Routing Machine (OSRM):** Used to determine driving routes and times between two coordinates, which is later used alongside demand data in the game

- **Overture Maps:** Used by `depot` to populate buildings in the map

- **Socioeconomic High-resolution Rural-Urban Geographic Platform for India (SHRUG):** This dataset contains locations/boundaries of villages and towns in India and is used to generate demand location alongside the official census data and village level boundaries

- **PC11 code:** Location codes at the state, district, subdistrict and town/village levels from the census that is available in the `SHRUG` dataset as well and acts as the unique identifier to merge both datasets together

- **admin_level:** An OSM key used to indicate different administrative boundaries (state, district, etc.) and used to pull district and taluk boundaries only as boundaries below the taluk (a.k.a. tehsil/mandal/etc. - the level below district) level are almost non-existent in India in OSM, hence SHRUG data is used for town/village level boundaries

- **Overpass:** Used for checking/pulling OpenStreetMap data based on certain filters

- **Digital Surface Model (DSM):** Elevation data used to build hillshade, 3D terrain, contour lines, and hypsometric tinting

- **Hillshade:** A shading effect that gives a 3D look to a map's terrain

- **Hypsometric tinting:** Colors on the map used to indicate elevation

----

## Stage 0: What we are building

*Est. time: 5 minutes (reading only)*

The game needs three things at a minimum:

1. Map tiles with natural features
2. Buildings and roads dataset
3. A demand dataset containing locations of where people live and work for the game to generate commuter traffic

This guide also covers three additional, *optional* stages which improve the visual look of the base game map.

----

## Stage 1: Initial Setup

*Est. time: 45 minutes*

### 1a. Connect to the VM's Debian Server from Windows

```bash
ssh <VM_USER>@<VM_IP>
```

Use an SSH terminal like Command Prompt, Powershell or Tabby to connect to the VM's Debian Server. The VM's IP can be found by typing in `ip a` within the VM's Debian terminal.

**Note:** Depending on the VM and your router settings, the IP might change later.

### 1b. Install Miniforge and the `depot` Python environment

1. Installing Miniforge:

```bash
# Prerequisite package
sudo apt install curl

# Creating new directory and making it the working directory
mkdir -p ~/SB_mod_project/tools
cd ~/SB_mod_project/tools

# Installing Miniforge in that directory
curl -L -O "https://github.com/conda-forge/miniforge/releases/latest/download/Miniforge3-Linux-x86_64.sh"
bash Miniforge3-Linux-x86_64.sh -p ~/SB_mod_project/tools/miniforge3
```

Read and accept the license and confirm the displayed install location. Then type in '*yes*' to the prompt asking to update shell profile to automatically initialize conda.

2. Close the terminal and restart the SSH session.

3. Installing `depot`:

```bash
# Activating Miniforge
source ~/SB_mod_project/tools/miniforge3/bin/activate #this line can be skipped if shell profile was updated in step 1 to automatically initialize conda

# Prerequisite package
sudo apt install git

# Creating new directory and making it the working directory
mkdir -p ~/SB_mod_project/tools/depot
cd ~/SB_mod_project/tools/depot

# Installing depot in that directory
git clone --branch 1.2.7 https://github.com/Subway-Builder-Modded/depot ~/SB_mod_project/tools/depot
conda env create -f environment.yml #installs depot's required libraries
conda activate depot
pip install -e . #installs depot itself
```

### 1c. Note on activating the `depot` environment after VM restart

After restarting VM or the SSH terminal, the below commands need to be run each time. This is the only thing you will need to remember to do throughout the guide.

```bash
source ~/SB_mod_project/tools/miniforge3/bin/activate #this line can be skipped if shell profile was updated in step 1 to automatically initialize conda
conda activate depot
```

### 1d. Installing other packages

1. Ensure the below pastes properly in the terminal, otherwise paste in line-by-line:

```bash
sudo apt install -y default-jre jq nodejs npm osmium-tool sqlite3 gdal-bin

# mapshaper
cd ~/SB_mod_project/tools
mkdir -p ~/SB_mod_project/tools/.npm-global
npm config set prefix ~/SB_mod_project/tools/.npm-global
echo 'export PATH="$HOME/SB_mod_project/tools/.npm-global/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
npm install -g mapshaper

# tippecanoe + tile-join
sudo apt install -y build-essential libsqlite3-dev zlib1g-dev
cd ~/SB_mod_project/tools
git clone https://github.com/felt/tippecanoe.git
cd tippecanoe
make -j
make install PREFIX=$HOME/SB_mod_project/tools/tippecanoe-bin
echo 'export PATH="$HOME/SB_mod_project/tools/tippecanoe-bin/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc

# pmtiles
mkdir -p ~/SB_mod_project/tools/pmtiles
cd ~/SB_mod_project/tools/pmtiles
PMTILES_URL=$(curl -s https://api.github.com/repos/protomaps/go-pmtiles/releases/latest \
  | jq -r '.assets[] | select(.name | test("Linux_x86_64")) | .browser_download_url')
wget "$PMTILES_URL"
tar -xzf go-pmtiles_*_Linux_x86_64.tar.gz
echo 'export PATH="$HOME/SB_mod_project/tools/pmtiles:$PATH"' >> ~/.bashrc
source ~/.bashrc
pmtiles version

# planetiler.jar
cd ~/SB_mod_project/tools
wget -O ~/SB_mod_project/tools/planetiler.jar https://github.com/onthegomap/planetiler/releases/latest/download/planetiler.jar
chmod +x ~/SB_mod_project/tools/planetiler.jar
echo 'export PATH="$HOME/SB_mod_project/tools:$PATH"' >> ~/.bashrc
source ~/.bashrc
```

2. Check to see if everything is installed correctly and there is no "Missing" in the output:

```bash
for t in node mapshaper osmium java sqlite3 tile-join tippecanoe jq pmtiles planetiler.jar gdalinfo; do
  echo -n "$t: "; which $t || echo "MISSING"
done
```

### 1e. Windows Folder Structure

To help organize all files used in creating / running the mod, run the below command in Command Prompt to create an empty folder structure where we will place files later on as they are created.

```batch
set root=%USERPROFILE%\Desktop\SB_mod_project

:: 01_source
mkdir "%root%\01_source\01_osm"
mkdir "%root%\01_source\02_elevation"
mkdir "%root%\01_source\03_census\01_shrug"

:: 02_intermediate
mkdir "%root%\02_intermediate"

:: 03_mod
mkdir "%root%\03_mod\<CITY_CODE_LOWER>-mod\data\<CITY_CODE>"
mkdir "%root%\03_mod\<CITY_CODE_LOWER>-mod\tiles"
mkdir "%root%\03_mod\<CITY_CODE_LOWER>-mod\tools"
```

### 1f. Applying VM memory patch for `depot`

`depot` expects 16GB free RAM. So if the allocated RAM in VM is <= 16GB RAM or if there are other running applications leading to <=16GB free RAM, run the below command to avoid potential crashes (12GB free RAM is used as an example).

1. Taking a backup of `maps.py` first using Command Prompt (cmd) from Windows:

```batch
# Copying the file from Debian to Windows
scp <VM_USER>@<VM_IP>:~/SB_mod_project/tools/depot/src/depot/maps.py "%USERPROFILE%\Desktop\SB_mod_project\02_intermediate"

# Renaming it so another backup can be made later of the final state of `maps.py`
ren "%USERPROFILE%\Desktop\SB_mod_project\02_intermediate\maps.py" "initial_maps.py"
```

2. Applying the patch on the Debian file:

```bash
sed -i 's/-Xmx16g/-Xmx12g/' ~/SB_mod_project/tools/depot/src/depot/maps.py
```

**Note:** Free/available RAM can be checked using `free -h`.

----

## Stage 2: Downloading OSM data for the `bbox`

*Est. time: 15 minutes*

We will download the map area using an OSM extract service from BBBike, which exports map data in the PBF format. To extract data for your interested city/region:

1. Go to https://extract.bbbike.org/

2. Select "**Format**" = '*Protocolbuffer (PBF)*', any relevant name of your choosing and your email.

3. Move and zoom the map to roughly the interested area and click the "**here**" button.

4. Adjust by clicking into the rectangle and dragging the handles or click the "**show lng/lat**" button and enter the coordinates for your interested area. This will be the `bbox` / `<BBOX_EXTRACT>`.

5. Click "Extract" and download the file from the link in your email.

6. Move the file to `Desktop\SB_mod_project\01_source\01_osm` and rename to `map.osm.pbf`

7. Run this to copy the file to the VM using cmd:

```batch
# Creating a directory to store "source" files
ssh <VM_USER>@<VM_IP> "mkdir -p ~/SB_mod_project/01_source"

# Copying the file from Windows to Debian
scp "%USERPROFILE%\Desktop\SB_mod_project\01_source\01_osm\map.osm.pbf" <VM_USER>@<VM_IP>:~/SB_mod_project/01_source/
```

----

## Stage 3: Building the base map data

*Est. time: 60 minutes (mostly processing/downloading in the background)*

1. Create a python file to store the code which generates the map data:

```bash
# Creating new directories to store "intermediate" and final "output" files
mkdir -p ~/SB_mod_project/02_intermediate/depot
mkdir -p ~/SB_mod_project/03_output

# Changing the working directory so intermediate depot files are stored in there
cd ~/SB_mod_project/02_intermediate/depot

# Code to generate base game map files
nano ~/SB_mod_project/02_intermediate/depot/map.py
```

2. Paste the below into it after replacing the placeholders and adjusting values for your VM (see the comments) and then "Ctrl+X" -> "Y" -> "Enter":

```python
import os
from depot.maps import MapGen

obj = MapGen(
    city='<CITY_CODE>',       # Choose your own code
    bbox=[<BBOX_EXTRACT>],    # The same coordinates that were entered in BBBike
    osmpbf=os.path.expanduser('~/SB_mod_project/01_source/map.osm.pbf'),
    outputdir=os.path.expanduser('~/SB_mod_project/03_output'),
    building_index_filter_size=10,   # Doesn't show buildings smaller than this size (m²)
    building_tile_filter_size=10,    # Doesn't show buildings smaller than this size (m²) at a high-zoom level
    building_index_simplification=1,
    building_tile_simplification=1,
    ncores=8,          # Adjust based on VM
    RAM=12,            # Adjust based on VM - should match what is provided in Stage 1f
    cities=['city', 'town'],   # Shows up at the lowest zoom, move labels to a different level as needed
    suburbs=['suburb'],
    neighborhoods=['neighbourhood', 'village', 'hamlet', 'quarter', 'locality']
)

obj.extract_base_data()
obj.process_buildings()
obj.process_roads_and_aeroways()
obj.generate_pmtiles()
obj.add_labels()
```

**Note:** Buildings specifically are sourced from Overture Maps and not OpenStreetMaps.

3. Check to see if everything looks right:

```bash
cat ~/SB_mod_project/02_intermediate/depot/map.py
```

4. Run the code:

```bash
python3 ~/SB_mod_project/02_intermediate/depot/map.py
```

Rerun it if getting a timeout error.

----

## Stage 4: Setting up OSRM

*Est. time: 15 minutes*

1. Install Docker by following [these instructions](https://docs.docker.com/engine/install/debian/), run `sudo usermod -aG docker $USER` and then restart the SSH terminal.

2. Run the below:

```bash
# Activating depot
source ~/SB_mod_project/tools/miniforge3/bin/activate && conda activate depot

# Creating new directory
mkdir -p ~/SB_mod_project/02_intermediate/osrm

# Initial prep
printf '{"points": [], "pops": []}' > ~/SB_mod_project/03_output/demand_data.json
cp ~/SB_mod_project/03_output/<CITY_CODE>/<CITY_CODE_LOWER>.osm.pbf ~/SB_mod_project/02_intermediate/osrm

# Code to generate the routing
nano ~/SB_mod_project/02_intermediate/osrm/prepare_osrm.py
```

3. Paste the below into the editor, update the four placeholders and save it:

```python
import os
from depot.demand import DemandData

d = DemandData(
    fdemand=os.path.expanduser('~/SB_mod_project/03_output/demand_data.json'),
    map_code='<CITY_CODE>',
    outputdir=os.path.expanduser('~/SB_mod_project/02_intermediate/osrm'),
    bbox=[<BBOX_EXTRACT>],
)

d.prepare_osrm(
    osmpbf=os.path.expanduser('~/SB_mod_project/02_intermediate/osrm/<CITY_CODE_LOWER>.osm.pbf'),
    bbox=[<BBOX_EXTRACT>],
    port=5000,
)
```

4. Run the code:

```bash
python3 ~/SB_mod_project/02_intermediate/osrm/prepare_osrm.py
```

5. Check that it has installed correctly:

```bash
docker ps    # Shows the OSRM container
curl "http://localhost:5000/route/v1/driving/<LON1>,<LAT1>;<LON2>,<LAT2>" # Use any two example coordinates inside the bbox
```

----

## Stage 5: Pulling census and creating demand data to simulate commuting patterns

*Est. time: 60 minutes (mostly processing in the background)*

### 5a. Getting and cleaning the census data

We need official census data for all districts that the map area covers at the "**town, village and ward level**". Follow the below steps to obtain the relevant files:

1. Go to https://censusindia.gov.in/nada/index.php/catalog/

2. Search for "**PC11_PCA-TV <Name_of_District>**" and click on the search result with the title `PCA TV: Primary census abstract at town, village and ward level, <State> - District <District> - 2011`

3. Download the `.xlsx` file and repeat for other districts as needed

4. Move them to `Desktop\SB_mod_project\01_source\03_census`

5. Copy the files to VM using cmd:

```batch
# Creating a directory to store the census files
ssh <VM_USER>@<VM_IP> "mkdir -p ~/SB_mod_project/01_source/census"

# Copying the files from Windows to Debian
scp "%USERPROFILE%\Desktop\SB_mod_project\01_source\03_census\*.xlsx" <VM_USER>@<VM_IP>:~/SB_mod_project/01_source/census/
```

### 5b. Obtaining coordinates for census place names using `SHRUG`

There is no official, public digital map of all district (and lower) boundaries and census locations in India. OSM also can't be used as it's boundary data is hit-or-miss below district levels and it also doesn't include many smaller villages and towns.

We will use the SHRUG dataset for these purposes instead. SHRUG is an academic project which has boundary and location data based on the 2011 Census for cities/towns and villages.

Also while census tends to split a city/town's population across wards, SHRUG doesn't have wards (and local government bodies generally only seem to share raster maps online), so we will instead spread the ward populations equally across a city/town as an approximation so all commuting doesn't occur to a single point for each city/town.

We will then combine the Census, SHRUG and simulated ward data together to map the census populations and their locations on our map.

**Note:** The below process was followed at the time of the v2.2 release of SHRUG.

1. Go to https://www.devdatalab.org/shrug_download/

2. Under `Open Polygons and Spatial Statistics`, click the `GPKG` option for `PC11 Village Polygons`. Take note of the citation.

3. Extract the download to get a `village_modified.gpkg` file.

4. Move it to `Desktop\SB_mod_project\01_source\03_census\01_shrug`.

5. Copy the file to Debian using cmd:

```batch
scp "%USERPROFILE%\Desktop\SB_mod_project\01_source\03_census\01_shrug\village_modified.gpkg" <VM_USER>@<VM_IP>:~/SB_mod_project/01_source/village_modified.gpkg
```

6. Filter the SHRUG dataset to just the relevant districts. Obtain the `<State_ID>` and `<District_ID>` values from the census files' first two columns.

```python
# Prerequisite package
pip install geopandas

python3 -c "
# Reading the file
import geopandas as gpd
gdf = gpd.read_file('~/SB_mod_project/01_source/village_modified.gpkg')

# Filtering to only relevant districts
filtered = gdf[(gdf['pc11_state_id']=='<State_ID>') & (gdf['pc11_district_id'].isin(['<District_ID_1>','<District_ID_2>',...]))]
print('Filtered row count:', len(filtered))
print(filtered['pc11_district_id'].value_counts())

# Exporting the file
filtered.to_file('~/SB_mod_project/02_intermediate/shrug_filtered.gpkg', driver='GPKG')
print('Saved.')
"
```

7. Splitting a city/town into "N" equal areas (N = number of wards), obtaining "N" representative points for them and then combining with census and SHRUG data:

```python
# Prerequisite packages
pip install scikit-learn pandas openpyxl

# Creating a .py file to store script
cat > ~/SB_mod_project/02_intermediate/build_unified_points.py << 'EOF'

# Importing packages
import os
import pandas as pd
import geopandas as gpd
from shapely.geometry import Point
import numpy as np
from sklearn.cluster import KMeans
from scipy.spatial import cKDTree

# Variables initialization
BBOX = (<BBOX_EXTRACT>)
all_points = []
next_village_id = 0
next_ward_id = 0
skipped_villages = 0
skipped_towns = 0
files = {
    '<district-1>': '~/SB_mod_project/01_source/census/<district-1_file_name>.xlsx',
    '<district-2>': '~/SB_mod_project/01_source/census/<district-2_file_name>.xlsx',
    '<...>': '...',
}

# Reading filtered SHRUG data and storing place IDs
gdf = gpd.read_file(os.path.expanduser('~/SB_mod_project/02_intermediate/shrug_filtered.gpkg'))
gdf['pc11_town_village_id'] = gdf['pc11_town_village_id'].astype(str)

# Function for determining if a point is located within `bbox`
def in_bbox(pt):
    return BBOX[0] <= pt.x <= BBOX[2] and BBOX[1] <= pt.y <= BBOX[3]

# Function for creating the "N" representative wards for towns/cities
def split_polygon(polygon, n, min_multiplier=20, max_resolution=400):
    minx, miny, maxx, maxy = polygon.bounds
    resolution = 20
    inside = []
    while True:
        xs = np.linspace(minx, maxx, resolution)
        ys = np.linspace(miny, maxy, resolution)
        pts = [Point(x, y) for x in xs for y in ys]
        inside = [p for p in pts if polygon.contains(p)]
        if len(inside) >= n * min_multiplier or resolution >= max_resolution:
            break
        resolution += 20
    if len(inside) < n:
        rp = polygon.representative_point()
        return [rp] * n
    coords = np.array([[p.x, p.y] for p in inside])
    km = KMeans(n_clusters=n, random_state=42, n_init=10).fit(coords)
    centers = km.cluster_centers_
    tree = cKDTree(coords)
    _, idx = tree.query(centers)
    snapped = coords[idx]
    return [Point(x, y) for x, y in snapped]

# Obtaining list and locations of villages and wards within `bbox`
for district, path in files.items():
    path = os.path.expanduser(path)
    xl = pd.ExcelFile(path)
    df = xl.parse(xl.sheet_names[0])
    df['Town/Village'] = df['Town/Village'].astype(str)

    villages = df[df['Level']=='VILLAGE'].copy()
    villages = villages.merge(gdf[['pc11_town_village_id','geometry']], left_on='Town/Village', right_on='pc11_town_village_id', how='left')
    for _, row in villages.iterrows():
        pt = row['geometry'].representative_point()
        if not in_bbox(pt):
            skipped_villages += 1
            continue
        all_points.append({
            'id': f'village_{next_village_id}', 'name': row['Name'], 'type': 'village', 'is_job_center': False,
            'lon': pt.x, 'lat': pt.y,
            'residents': row['TOT_P'], 'workers': row['TOT_WORK_P']
        })
        next_village_id += 1

    towns = df[df['Level']=='TOWN'].copy()
    towns = towns.merge(gdf[['pc11_town_village_id','geometry']], left_on='Town/Village', right_on='pc11_town_village_id', how='left')
    wards = df[df['Level']=='WARD'].copy()

    for _, town_row in towns.iterrows():
        code = town_row['Town/Village']
        town_wards = wards[wards['Town/Village']==code].reset_index(drop=True)
        n = len(town_wards)
        if n == 0:
            continue
        poly = town_row['geometry']
        town_center = poly.representative_point()
        if not in_bbox(town_center):
            skipped_towns += 1
            continue
        pts = split_polygon(poly, n)
        for i, (_, ward_row) in enumerate(town_wards.iterrows()):
            all_points.append({
                'id': f'town_{next_ward_id}', 'name': ward_row['Name'], 'type': 'ward', 'is_job_center': True,
                'lon': pts[i].x, 'lat': pts[i].y,
                'residents': ward_row['TOT_P'], 'workers': ward_row['TOT_WORK_P']
            })
            next_ward_id += 1

# Exporting the file
out = pd.DataFrame(all_points, columns=['id','name','type','is_job_center','lon','lat','residents','workers'])
out.to_csv(os.path.expanduser('~/SB_mod_project/02_intermediate/unified_points.csv'), index=False)

# Summary stats
print('Total points:', len(out))
print(out['type'].value_counts())
print('Job centers:', out['is_job_center'].sum())
print('Total residents:', out['residents'].sum())
print('Total workers:', out['workers'].sum())
print('Villages skipped (outside bbox):', skipped_villages)
print('Towns skipped (outside bbox, all their wards dropped too):', skipped_towns)

EOF

# Running this .py script
python3 ~/SB_mod_project/02_intermediate/build_unified_points.py
```

### 5c. Generating commuting demand model

The below will be the assumptions used to generate the demand model as Indian cities generally don't have this data available.

1. Every village / ward will have 50% of workers work within their village / ward (0 commuting distance) and the rest 50% will be split across all eligible destinations by inverse-distance-squared weighting.
2. Villages don't receive commuters from other places.
3. Round off each allocation, then fix rounding drift by adjusting the largest single allocation.
4. Drop any allocation that rounds to under 1.

```python
# Creating a .py file to store script
cat > ~/SB_mod_project/02_intermediate/build_commuters.py << 'EOF'

# Importing packages
import os
import json
import numpy as np
import pandas as pd

# Reading the coordinates file from previous step
pts = pd.read_csv(os.path.expanduser('~/SB_mod_project/02_intermediate/unified_points.csv'))

# Defining residence and job locations
origins = pts.reset_index(drop=True)
jobs = pts[pts['is_job_center']].reset_index(drop=True)

# Variables initialization
N = len(origins)
M = len(jobs)

# Splitting each locations workers share b/w local work and commuting (50% each)
local_share = origins['workers'].values * 0.5
distributed_pool = origins['workers'].values * 0.5

# Function for calculating distance b/w two points
def haversine_matrix(lat1, lon1, lat2, lon2):
    lat1, lon1, lat2, lon2 = map(np.radians, [lat1, lon1, lat2, lon2])
    dlat = lat2[None, :] - lat1[:, None]
    dlon = lon2[None, :] - lon1[:, None]
    a = np.sin(dlat/2)**2 + np.cos(lat1[:, None]) * np.cos(lat2[None, :]) * np.sin(dlon/2)**2
    c = 2 * np.arcsin(np.sqrt(np.clip(a, 0, 1)))
    return 6371.0 * c  # km

# Mask to prevent the assignment of a portion of a location's commuting group to itself
job_col_of_id = {row['id']: i for i, row in jobs.iterrows()}
self_mask = np.ones((N, M))
for i, row in origins.iterrows():
    if row['id'] in job_col_of_id:
        self_mask[i, job_col_of_id[row['id']]] = 0

# Assigning weights to job centers from each location based on distance
D = haversine_matrix(origins['lat'].values, origins['lon'].values, jobs['lat'].values, jobs['lon'].values)
with np.errstate(divide='ignore'):
    weights = np.where(D > 0, 1.0 / (D ** 2), 0.0)
weights = weights * self_mask
row_sums = weights.sum(axis=1, keepdims=True)
row_sums[row_sums == 0] = 1
weights_norm = weights / row_sums

# Determining actual commuting numbers to job centers from each location
distributed_alloc = weights_norm * distributed_pool[:, None]

# Corrections for rounding errors
augmented = np.hstack([distributed_alloc, local_share[:, None]])
rounded = np.round(augmented)
drift = np.round(origins['workers'].values - rounded.sum(axis=1))
for i in range(N):
    if drift[i] != 0:
        j = np.argmax(rounded[i])
        new_val = rounded[i, j] + drift[i]
        rounded[i, j] = max(new_val, 0)

# Generating commuting pairs
pops = []
next_pop_id = 0
job_ids = jobs['id'].values
for i in range(N):
    origin_id = origins.loc[i, 'id']

    local_size = int(rounded[i, M])
    if local_size >= 1:
        pops.append({
            'id': f'pop_{next_pop_id}', 'size': local_size,
            'residenceId': origin_id, 'jobId': origin_id,
            'drivingSeconds': None, 'drivingDistance': None
        })
        next_pop_id += 1

    for j in range(M):
        size = int(rounded[i, j])
        if size >= 1:
            pops.append({
                'id': f'pop_{next_pop_id}', 'size': size,
                'residenceId': origin_id, 'jobId': job_ids[j],
                'drivingSeconds': None, 'drivingDistance': None
            })
            next_pop_id += 1

# Exporting the file
with open(os.path.expanduser('~/SB_mod_project/02_intermediate/commuters.json'), 'w') as f:
    json.dump(pops, f)

# Summary stats
print('Total OD pairs:', len(pops))
print('Total allocated workers:', sum(p['size'] for p in pops))
print('Total workers in source data:', int(origins['workers'].sum()))

EOF

# Running this .py script
python3 ~/SB_mod_project/02_intermediate/build_commuters.py
```

### 5d. Determining commute times

```python
# Can skip this line if VM wasn't restarted since setting up OSRM
docker start <CITY_CODE>

# Creating a .py file to store script
cat > ~/SB_mod_project/02_intermediate/run_routing.py << 'EOF'

# Importing packages
import os
import json
import pandas as pd
from depot.demand import DemandData

# Reading the coordinates file
pts_df = pd.read_csv(os.path.expanduser('~/SB_mod_project/02_intermediate/unified_points.csv'))

# Reading the demand file from previous step
with open(os.path.expanduser('~/SB_mod_project/02_intermediate/commuters.json')) as f:
    pops = json.load(f)

# Placeholders for commuting numbers
for p in pops:
    if p['drivingSeconds'] is None:
        p['drivingSeconds'] = 0
    if p['drivingDistance'] is None:
        p['drivingDistance'] = 0

# Preparing data into specific format for depot
points = []

for _, row in pts_df.iterrows():
    points.append({
        'id': row['id'],
        'name': row['name'],
        'type': row['type'],
        'is_job_center': bool(row['is_job_center']),
        'location': [float(row['lon']), float(row['lat'])],
        'residents': int(row['residents']),
        'workers': int(row['workers']),
    })

d = DemandData(
    fdemand=os.path.expanduser('~/SB_mod_project/03_output/demand_data.json'),
    map_code='<CITY_CODE>',
    outputdir=os.path.expanduser('~/SB_mod_project/02_intermediate/osrm'),
    bbox=[<BBOX_EXTRACT>],
)

d['points'] = points
d['pops'] = pops

# Querying OSRM
d.calculate_routes(routing_method='osrm', osrm_port=5000)

# Shares the result back with depot
d.save()

# Summary stats
print('Done. Points:', len(points), 'Pops:', len(pops))

EOF

# Running this .py script
python3 ~/SB_mod_project/02_intermediate/run_routing.py
```

Note: The above code drops any locations with zero workers and residents (Ex: Reserve Forests, which show up in census data).

### 5e. Checking the result

Check whether `Real distinct-point pairs routed` is close to 100%:

```python
python3 -c "
import json, os
with open(os.path.expanduser('~/SB_mod_project/03_output/demand_data.json')) as f:
    d = json.load(f)

print('Saved points:', len(d['points']))
print('Saved pops:', len(d['pops']))

routed = sum(1 for p in d['pops'] if p['residenceId'] != p['jobId'] and p['drivingSeconds'] > 0)
real_pairs = sum(1 for p in d['pops'] if p['residenceId'] != p['jobId'])
print(f'Real distinct-point pairs routed: {routed} / {real_pairs}')

total_workers_routed = sum(p['size'] for p in d['pops'])
print('Total workers represented:', total_workers_routed)
"
```

----

## Stage 6: Testing mod files in-game

*Est. time: 15 minutes.*

### 6a. Compressing mod files and transferring to Windows

gzip shrinks text-based files (JSON/GeoJSON) into the `.gz` format, which loads faster in the game.

1. Run the below on Debian:

```bash
gzip -kf ~/SB_mod_project/03_output/<CITY_CODE>/roads.geojson
gzip -kf ~/SB_mod_project/03_output/<CITY_CODE>/runways_taxiways.geojson
gzip -kf ~/SB_mod_project/03_output/demand_data.json
```

2. Copy mod files to Windows using cmd:

```batch
scp <VM_USER>@<VM_IP>:~/SB_mod_project/03_output/<CITY_CODE>/roads.geojson.gz "%USERPROFILE%\Desktop\SB_mod_project\03_mod\<CITY_CODE_LOWER>-mod\data\<CITY_CODE>"
scp <VM_USER>@<VM_IP>:~/SB_mod_project/03_output/demand_data.json.gz "%USERPROFILE%\Desktop\SB_mod_project\03_mod\<CITY_CODE_LOWER>-mod\data\<CITY_CODE>"
scp <VM_USER>@<VM_IP>:~/SB_mod_project/03_output/<CITY_CODE>/runways_taxiways.geojson.gz "%USERPROFILE%\Desktop\SB_mod_project\03_mod\<CITY_CODE_LOWER>-mod\data\<CITY_CODE>"
scp <VM_USER>@<VM_IP>:~/SB_mod_project/03_output/<CITY_CODE>/buildings_index.bin.gz "%USERPROFILE%\Desktop\SB_mod_project\03_mod\<CITY_CODE_LOWER>-mod\data\<CITY_CODE>"
scp <VM_USER>@<VM_IP>:~/SB_mod_project/03_output/<CITY_CODE>/ocean_depth_index.json.gz "%USERPROFILE%\Desktop\SB_mod_project\03_mod\<CITY_CODE_LOWER>-mod\data\<CITY_CODE>"

scp <VM_USER>@<VM_IP>:~/SB_mod_project/03_output/<CITY_CODE>/<CITY_CODE>.pmtiles "%USERPROFILE%\Desktop\SB_mod_project\03_mod\<CITY_CODE_LOWER>-mod\tiles"
scp <VM_USER>@<VM_IP>:~/SB_mod_project/03_output/<CITY_CODE>/<CITY_CODE>_foundations.pmtiles "%USERPROFILE%\Desktop\SB_mod_project\03_mod\<CITY_CODE_LOWER>-mod\tiles"
```

### 6b. Creating configuration files:

1. Creating `manifest.json`:

```batch
cd "%USERPROFILE%\Desktop\SB_mod_project\03_mod\<CITY_CODE_LOWER>-mod"
notepad manifest.json
```

2. Paste the below, save and close notepad:

```json
{
  "id": "com.<your_name>.<city-code-lowercase>-mod",
  "name": "<Name_of_mod_to_appear_in_game>",
  "description": "<Brief_description_of_mod>",
  "version": "1.0.0",
  "author": {"name": "<Your_Name>"},
  "main": "index.js"
}
```

3. Find `bbox`'s center coordinate and population in Debian:

```python
python3 -c "

import pandas as pd

pts = pd.read_csv('~/SB_mod_project/02_intermediate/unified_points.csv')
print('Total residents:', int(pts['residents'].sum()))

lat_center = (pts['lat'].min() + pts['lat'].max()) / 2
lon_center = (pts['lon'].min() + pts['lon'].max()) / 2
print('Center lat:', round(lat_center, 4))
print('Center lon:', round(lon_center, 4))
"
```

4. Create `index.js`:

```batch
cd "%USERPROFILE%\Desktop\SB_mod_project\03_mod\<CITY_CODE_LOWER>-mod"
notepad index.js
```

5. Paste the below, save and close notepad:

```javascript
// Information about the city and initial camera placement
window.SubwayBuilderAPI.registerCity({
  name: "<Name_of_mod_to_appear_in_game>",
  code: "<CITY_CODE>",
  description: "<Brief_description_of_mod>",
  population: <POPULATION>,   // Use the value from two steps above
  difficulty: null,
  initialViewState: {
    zoom: <ZOOM>,             // Recommend starting at 13 and then change later
    latitude: <INITIAL_LAT>,  // Use the value from two steps above or a different location if preferred
    longitude: <INITIAL_LON>, // Use the value from two steps above or a different location if preferred
    bearing: 0
  }
});

// Tile server
window.SubwayBuilderAPI.map.setTileURLOverride({
  cityCode: "<CITY_CODE>",
  tilesUrl: "http://127.0.0.1:8080/<CITY_CODE>/{z}/{x}/{y}.mvt",
  foundationTilesUrl: "http://127.0.0.1:8080/<CITY_CODE>_foundations/{z}/{x}/{y}.mvt",
  maxZoom: 15
});

// Tiles location
window.SubwayBuilderAPI.cities.setCityDataFiles("<CITY_CODE>", {
  demandData: "/data/<CITY_CODE>/demand_data.json.gz",
  roads: "/data/<CITY_CODE>/roads.geojson.gz",
  runwaysTaxiways: "/data/<CITY_CODE>/runways_taxiways.geojson.gz"
});

// Roads
window.SubwayBuilderAPI.map.registerLayer({
  id: '<CITY_CODE_LOWER>-road-lines',
  type: 'line',
  source: 'roads-source',
  paint: {
    'line-color': '#8a8a8a',
    'line-width': [
      'interpolate', ['linear'], ['zoom'],
      10, 0.5,
      15, 1.5,
      18, 3
    ]
  }
}, 'road-labels');
```

### 6c. Setting up server and getting mod files loaded

1. Download [go-pmtiles](https://github.com/protomaps/go-pmtiles/releases), extract and place `pmtiles.exe` in `Desktop\SB_mod_project\03_mod\<CITY_CODE_LOWER>-mod\tools`.

2. Write a Batch script file in `Desktop\SB_mod_project\03_mod\<CITY_CODE_LOWER>-mod\tools`. Give it any name and the `.bat` extension and paste the below code into it:

```batch
@echo off
cd /d "%~dp0"
echo Keep this window open while playing. Close it to stop the server.
pmtiles.exe serve ..\tiles
```

It will be used to start a server to serve `.pmtiles` files.

3. Move the mod files to the game folders:

```batch
mkdir "%APPDATA%\metro-maker4\mods\"
xcopy "%USERPROFILE%\Desktop\SB_mod_project\03_mod" "%APPDATA%\metro-maker4\mods"  /E /I /D /Y

mkdir "%APPDATA%\metro-maker4\cities\data\<CITY_CODE>"
copy "%USERPROFILE%\Desktop\SB_mod_project\03_mod\<CITY_CODE_LOWER>-mod\data\<CITY_CODE>\roads.geojson.gz" "%APPDATA%\metro-maker4\cities\data\<CITY_CODE>\"
copy "%USERPROFILE%\Desktop\SB_mod_project\03_mod\<CITY_CODE_LOWER>-mod\data\<CITY_CODE>\demand_data.json.gz" "%APPDATA%\metro-maker4\cities\data\<CITY_CODE>\"
copy "%USERPROFILE%\Desktop\SB_mod_project\03_mod\<CITY_CODE_LOWER>-mod\data\<CITY_CODE>\runways_taxiways.geojson.gz" "%APPDATA%\metro-maker4\cities\data\<CITY_CODE>\"
copy "%USERPROFILE%\Desktop\SB_mod_project\03_mod\<CITY_CODE_LOWER>-mod\data\<CITY_CODE>\buildings_index.bin.gz" "%APPDATA%\metro-maker4\cities\data\<CITY_CODE>\"
copy "%USERPROFILE%\Desktop\SB_mod_project\03_mod\<CITY_CODE_LOWER>-mod\data\<CITY_CODE>\ocean_depth_index.json.gz" "%APPDATA%\metro-maker4\cities\data\<CITY_CODE>\"
```

4. Validate by running the above server script (either in the desktop folder or the game mod folder), going to the game, enabling the mod, restarting the game and then loading the new map. Check if the expected area (`bbox`), place names, water bodies, buildings, roads and demand model all appear correctly.

5. If everything loads correctly, the base game functionality is all done. The next stages are all purely cosmetic and optional, where we add additional land uses, elevation data, etc. to the map.

----

## Stage 7: Elevation map layers (Optional)

*Est. time: 30 minutes*

### 7a. Download DEM for the `bbox`

1. Go to https://portal.opentopography.org/raster?opentopoID=OTSDEM.032021.4326.3.

2. Click the "**Manually enter selection coordinates**", enter `bbox` values and click the "**Validate coordinates and estimate area**" button.

3. Select '*GeoTiff*' as "**Data Output Format**", the toggle for "**Digital Surface Model (DSM)**" and the "**Submit**" button. Take note of the citation.

4. Extract the downloaded file and move the extracted file (`output_hh.tif`) to `Desktop\SB_mod_project\01_source\02_elevation`.

5. Move the file to Debian using cmd:

```batch
scp "%USERPROFILE%\Desktop\SB_mod_project\01_source\02_elevation\output_hh.tif" <VM_USER>@<VM_IP>:~/SB_mod_project/01_source/
```

### 7b. Hillshade

1. Convert the raster image to `.mbtiles`:

```bash
# Prerequisite packages
pip install rasterio rio-rgbify

# Conversion
cd ~/SB_mod_project/01_source
rio rgbify -b -10000 -i 0.1 --min-z 8 --max-z 12 --format png output_hh.tif elevation.mbtiles
```

2. Check if the below outputs a non-zero file size and the '*EPSG:4326*' coordinate system:

```bash
ls -lh elevation.mbtiles  # Non-zero file size
gdalinfo output_hh.tif | grep -i "coordinate system\|EPSG" # ID["EPSG",4326]]
```

3. Convert `.mbtiles` to `.pmtiles` and move to output folder:

```bash
pmtiles convert elevation.mbtiles elevation.pmtiles
mv elevation.pmtiles ~/SB_mod_project/03_output/
```

4. Move the file to Windows using cmd:

```batch
scp <VM_USER>@<VM_IP>:~/SB_mod_project/03_output/elevation.pmtiles "%USERPROFILE%\Desktop\SB_mod_project\03_mod\<CITY_CODE_LOWER>-mod\tiles"
```

5. Run below in cmd:

```batch
notepad "%USERPROFILE%\Desktop\SB_mod_project\02_intermediate\hillshade_index-js.js"
```

6. Paste into the notepad:

```javascript


/*----------------------------------------------------
--------------------- Hillshade ----------------------
----------------------------------------------------*/

window.SubwayBuilderAPI.map.registerSource('<CITY_CODE_LOWER>-hillshade-source', {
  type: 'raster-dem',
  tiles: ['http://127.0.0.1:8080/elevation/{z}/{x}/{y}.png'],
  encoding: 'mapbox',
  tileSize: 256,
  maxzoom: 12
});

window.SubwayBuilderAPI.map.registerLayer({
  id: '<CITY_CODE_LOWER>-hillshade',
  type: 'hillshade',
  source: '<CITY_CODE_LOWER>-hillshade-source',
  paint: {
    'hillshade-illumination-direction': 315,   
    'hillshade-illumination-anchor': 'map',
    'hillshade-exaggeration': 0.9,
    'hillshade-shadow-color': '#000000',
    'hillshade-highlight-color': '#FFFFFF',
    'hillshade-accent-color': '#000000'
  }
}, '<CITY_CODE_LOWER>-road-lines');
```

7. Append the above into `index.js`:

```batch
type "%USERPROFILE%\Desktop\SB_mod_project\02_intermediate\hillshade_index-js.js" >> "%USERPROFILE%\Desktop\SB_mod_project\03_mod\<CITY_CODE_LOWER>-mod\index.js"
```

8. You can adjust the 6 hillshade parameters above to change how it looks. For an easier way to preview them, you can save the below code as a HTML file and open the file in a browser. It has options to adjust them live (with the region being set to the southern tip of India with the Western Ghats in view - adjust '*[77.5, 8.75]*' in the code directly to see preview for a different region).

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="utf-8" />
  <title>Hillshade Test</title>
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <script src="https://unpkg.com/maplibre-gl@4.7.1/dist/maplibre-gl.js"></script>
  <link href="https://unpkg.com/maplibre-gl@4.7.1/dist/maplibre-gl.css" rel="stylesheet" />
  <style>
    body { margin: 0; padding: 0; }
    #map { position: absolute; top: 0; bottom: 0; width: 100%; }
    #controls {
      position: absolute; top: 10px; left: 10px; z-index: 1;
      background: white; padding: 10px; border-radius: 6px;
      font-family: sans-serif; font-size: 13px; max-width: 260px;
    }
    #controls label { display: block; margin-top: 6px; }
  </style>
</head>
<body>

<div id="controls">
  <strong>Hillshade Test</strong><br/>
  <label>Anchor:
    <select id="anchor">
      <option value="map">map</option>
      <option value="viewport">viewport</option>
    </select>
  </label>
  <label>Direction:
    <input type="range" id="direction" min="0" max="359" value="315" />
    <span id="directionVal">315</span>°
  </label>
  <label>Exaggeration:
    <input type="range" id="exaggeration" min="0" max="1" step="0.05" value="0.9" />
    <span id="exaggerationVal">0.9</span>
  </label>
  <label>Shadow color:
    <input type="color" id="shadowColor" value="#000000" />
  </label>
  <label>Highlight color:
    <input type="color" id="highlightColor" value="#ffffff" />
  </label>
  <label>Accent color:
    <input type="color" id="accentColor" value="#000000" />
  </label>
  <div style="margin-top:8px; color:#555;">
    Right-click + drag (or two-finger twist) to rotate the map and see how the anchor setting behaves.
  </div>
</div>

<div id="map"></div>

<script>
  window.map = new maplibregl.Map({
    container: 'map',
    zoom: 9,
    center: [77.5, 8.75],
    pitch: 60,
    style: {
      version: 8,
      sources: {
        hillshadeSource: {
          type: 'raster-dem',
          tiles: ['https://tiles.mapterhorn.com/{z}/{x}/{y}.webp'],
          encoding: 'terrarium',
          tileSize: 512,
          maxzoom: 15
        }
      },
      layers: [
        {
          id: 'background',
          type: 'background',
          paint: { 'background-color': '#e8e4d8' }
        },
        {
          id: 'hillshade',
          type: 'hillshade',
          source: 'hillshadeSource',
          paint: {
            'hillshade-illumination-direction': 315,
            'hillshade-illumination-anchor': 'map',
            'hillshade-exaggeration': 0.9,
            'hillshade-shadow-color': '#000000',
            'hillshade-highlight-color': '#FFFFFF',
            'hillshade-accent-color': '#000000'
          }
        }
      ]
    },
    maxZoom: 18,
    maxPitch: 85
  });

  window.map.addControl(new maplibregl.NavigationControl());

  document.getElementById('anchor').addEventListener('change', (e) => {
    window.map.setPaintProperty('hillshade', 'hillshade-illumination-anchor', e.target.value);
  });
  document.getElementById('direction').addEventListener('input', (e) => {
    document.getElementById('directionVal').textContent = e.target.value;
    window.map.setPaintProperty('hillshade', 'hillshade-illumination-direction', Number(e.target.value));
  });
  document.getElementById('exaggeration').addEventListener('input', (e) => {
    document.getElementById('exaggerationVal').textContent = e.target.value;
    window.map.setPaintProperty('hillshade', 'hillshade-exaggeration', Number(e.target.value));
  });
  document.getElementById('shadowColor').addEventListener('input', (e) => {
    window.map.setPaintProperty('hillshade', 'hillshade-shadow-color', e.target.value);
  });
  document.getElementById('highlightColor').addEventListener('input', (e) => {
    window.map.setPaintProperty('hillshade', 'hillshade-highlight-color', e.target.value);
  });
  document.getElementById('accentColor').addEventListener('input', (e) => {
    window.map.setPaintProperty('hillshade', 'hillshade-accent-color', e.target.value);
  });
</script>

</body>
</html>
```

### 7c. Contour lines

1. Creating vector tiles on Debian:

```bash
# Generate contour lines
cd ~/SB_mod_project/01_source
gdal_contour -a elevation -i 20 output_hh.tif contours.geojson

# Convert to MBTiles
tippecanoe -o contours.mbtiles -Z8 -z14 --extend-zooms-if-still-dropping -l contours contours.geojson

# Convert to PMTiles
pmtiles convert contours.mbtiles contours.pmtiles

# Moving to output folder
mv contours.pmtiles ~/SB_mod_project/03_output/
```

2. Move the file to Windows using cmd:

```batch
scp <VM_USER>@<VM_IP>:~/SB_mod_project/03_output/contours.pmtiles "%USERPROFILE%\Desktop\SB_mod_project\03_mod\<CITY_CODE_LOWER>-mod\tiles"
```

3. Run below in cmd:

```batch
notepad "%USERPROFILE%\Desktop\SB_mod_project\02_intermediate\contour-lines_index-js.js"
```

4. Paste into the notepad:

```javascript


/*----------------------------------------------------
------------------- Contour Lines --------------------
----------------------------------------------------*/

window.SubwayBuilderAPI.map.registerSource('<CITY_CODE_LOWER>-contours-source', {
  type: 'vector',
  tiles: ['http://127.0.0.1:8080/contours/{z}/{x}/{y}.mvt'],
  maxzoom: 15
});

// Contour lines
window.SubwayBuilderAPI.map.registerLayer({
  id: '<CITY_CODE_LOWER>-contours',
  type: 'line',
  source: '<CITY_CODE_LOWER>-contours-source',
  'source-layer': 'contours',
  minzoom: 10,
  layout: { visibility: 'none' }, // Should only be added if layer is to not be shown by default - should be combined with 'defaultOn: false' in the Legend section
  paint: {
    'line-color': '#a67c52',
    'line-width': [
      'interpolate', ['linear'], ['zoom'],
      10, 0.3,
      15, 0.8
    ],
    'line-opacity': 0.5
  }
}, 'road-labels');

// Contour line labels
window.SubwayBuilderAPI.map.registerLayer({
  id: '<CITY_CODE_LOWER>-contours-labels',
  type: 'symbol',
  source: '<CITY_CODE_LOWER>-contours-source',
  'source-layer': 'contours',
  minzoom: 13,
  layout: {
     visibility: 'none',
    'symbol-placement': 'line',
    'text-field': ['concat', ['to-string', ['get', 'elevation']], ' m'],
    'text-size': 10,
    'text-font': ['Noto Sans Regular'],
    'symbol-spacing': 400,
    'text-max-angle': 30
  },
  paint: {
    'text-color': '#6b4f33',
    'text-halo-color': '#ffffff',
    'text-halo-width': 1
  }
}, 'road-labels');
```

5. Append the above into `index.js`:

```batch
type "%USERPROFILE%\Desktop\SB_mod_project\02_intermediate\contour-lines_index-js.js" >> "%USERPROFILE%\Desktop\SB_mod_project\03_mod\<CITY_CODE_LOWER>-mod\index.js"
```

### 7d. Hypsometric tinting

1. Creating vector tiles on Debian:

```bash
cd ~/SB_mod_project/01_source

# Defining color ramp - adjust based on topography of your map
cat > color_ramp.txt << 'EOF'
0     255 255 255
150   255 255 255
200   180 220 150
400   120 190 100
700   200 170 90
1000  150 100 60
1400  230 220 210
1800  255 255 255
EOF

# Applying the color ramp
gdaldem color-relief output_hh.tif color_ramp.txt hypso_colored.tif -alpha

# Reprojecting from EPSG:4326 to EPSG:3857
gdalwarp -t_srs EPSG:3857 hypso_colored.tif hypso_colored_3857.tif

# Converting to MBTiles
gdal_translate -of MBTILES hypso_colored_3857.tif hypso_colored.mbtiles

# Adding lower resolutions for lower zoom levels
gdaladdo -r average hypso_colored.mbtiles 2 4 8 16

# Converting to PMTiles
pmtiles convert hypso_colored.mbtiles hypso_colored.pmtiles

# Moving to output folder
mv hypso_colored.pmtiles ~/SB_mod_project/03_output/

# Repeating for dark mode
cat > color_ramp_dark.txt << 'EOF'
0     0   0   0
150   0   0   0
200   60  90  50
400   70  120 55
700   140 110 60
1000  110 75  40
1400  190 180 160
1800  230 225 210
EOF
gdaldem color-relief output_hh.tif color_ramp_dark.txt hypso_colored_dark.tif -alpha
gdalwarp -t_srs EPSG:3857 hypso_colored_dark.tif hypso_colored_dark_3857.tif
gdal_translate -of MBTILES hypso_colored_dark_3857.tif hypso_colored_dark.mbtiles
gdaladdo -r average hypso_colored_dark.mbtiles 2 4 8 16
pmtiles convert hypso_colored_dark.mbtiles hypso_colored_dark.pmtiles
mv hypso_colored_dark.pmtiles ~/SB_mod_project/03_output/
```

2. Move the files to Windows using cmd:

```batch
scp <VM_USER>@<VM_IP>:~/SB_mod_project/03_output/hypso_colored.pmtiles "%USERPROFILE%\Desktop\SB_mod_project\03_mod\<CITY_CODE_LOWER>-mod\tiles"
scp <VM_USER>@<VM_IP>:~/SB_mod_project/03_output/hypso_colored_dark.pmtiles "%USERPROFILE%\Desktop\SB_mod_project\03_mod\<CITY_CODE_LOWER>-mod\tiles"
```

3. Run below in cmd:

```batch
notepad "%USERPROFILE%\Desktop\SB_mod_project\02_intermediate\hypsometric-tinting_index-js.js"
```

4. Paste into the notepad:

```javascript


/*----------------------------------------------------
---------------- Hypsometric Tinting -----------------
----------------------------------------------------*/

// Light mode
window.SubwayBuilderAPI.map.registerSource('<CITY_CODE_LOWER>-hypsometric-source', {
  type: 'raster',
  tiles: ['http://127.0.0.1:8080/hypso_colored/{z}/{x}/{y}.png'],
  tileSize: 256, minzoom: 8, maxzoom: 12
});

window.SubwayBuilderAPI.map.registerLayer({
  id: '<CITY_CODE_LOWER>-hypsometric',
  type: 'raster',
  source: '<CITY_CODE_LOWER>-hypsometric-source',
  paint: { 'raster-opacity': 0.7 }
}, 'buildings-3d');

// Dark mode
window.SubwayBuilderAPI.map.registerSource('<CITY_CODE_LOWER>-hypsometric-dark-source', {
  type: 'raster',
  tiles: ['http://127.0.0.1:8080/hypso_colored_dark/{z}/{x}/{y}.png'],
  tileSize: 256,
  minzoom: 8,
  maxzoom: 12
});

window.SubwayBuilderAPI.map.registerLayer({
  id: '<CITY_CODE_LOWER>-hypsometric-dark',
  type: 'raster',
  source: '<CITY_CODE_LOWER>-hypsometric-dark-source',
  paint: { 'raster-opacity': 0.7 }
}, 'buildings-3d');

// Function to swap between layers used for light and dark modes
(function () {
  const api = window.SubwayBuilderAPI;

  function isHypsoDarkMode() {
    if (document.documentElement.classList.contains('dark')) return true;
    if (document.documentElement.classList.contains('light')) return false;
    return !!(window.matchMedia && window.matchMedia('(prefers-color-scheme: dark)').matches);
  }

  let hypsoLastState = null;
  function applyHypsoSwap() {
    const map = api.utils.getMap();
    if (!map || !map.getLayer('<CITY_CODE_LOWER>-hypsometric') || !map.getLayer('<CITY_CODE_LOWER>-hypsometric-dark')) return;
    const dark = isHypsoDarkMode();
    const lightVis = map.getLayoutProperty('<CITY_CODE_LOWER>-hypsometric', 'visibility');
    const darkVis = map.getLayoutProperty('<CITY_CODE_LOWER>-hypsometric-dark', 'visibility');
    const enabled = (lightVis !== 'none') || (darkVis !== 'none');
    const stateKey = dark + ':' + enabled;
    if (stateKey === hypsoLastState) return;
    hypsoLastState = stateKey;
    map.setLayoutProperty('<CITY_CODE_LOWER>-hypsometric', 'visibility', (enabled && !dark) ? 'visible' : 'none');
    map.setLayoutProperty('<CITY_CODE_LOWER>-hypsometric-dark', 'visibility', (enabled && dark) ? 'visible' : 'none');
  }

  function reapplyHypsoSwap() {
    hypsoLastState = null;
    applyHypsoSwap();
  }

  let hypsoMq = null;
  let hypsoObserver = null;
  let hypsoPollId = null;
  let hypsoStyleTimer = null;
  let hypsoStyleListenerMap = null;

  function teardownHypsoSwap() {
    if (hypsoMq) { hypsoMq.removeEventListener('change', applyHypsoSwap); hypsoMq = null; }
    if (hypsoObserver) { hypsoObserver.disconnect(); hypsoObserver = null; }
    if (hypsoPollId) { clearInterval(hypsoPollId); hypsoPollId = null; }
    if (hypsoStyleTimer) { clearTimeout(hypsoStyleTimer); hypsoStyleTimer = null; }
    if (hypsoStyleListenerMap) { hypsoStyleListenerMap.off('styledata', onHypsoStyleData); hypsoStyleListenerMap = null; }
  }

  function onHypsoStyleData() {
    clearTimeout(hypsoStyleTimer);
    hypsoStyleTimer = setTimeout(reapplyHypsoSwap, 150);
  }

  api.hooks.onMapReady(function () {
    teardownHypsoSwap();

    const map = api.utils.getMap();
    hypsoLastState = null;
    applyHypsoSwap();
    if (window.matchMedia) {
      hypsoMq = window.matchMedia('(prefers-color-scheme: dark)');
      hypsoMq.addEventListener('change', applyHypsoSwap);
    }
    hypsoObserver = new MutationObserver(applyHypsoSwap);
    hypsoObserver.observe(document.documentElement, { attributes: true, attributeFilter: ['data-theme', 'class', 'style'] });
    hypsoObserver.observe(document.body, { attributes: true, attributeFilter: ['data-theme', 'class', 'style'] });
    hypsoPollId = setInterval(applyHypsoSwap, 1000);

    if (map) {
      hypsoStyleListenerMap = map;
      map.on('styledata', onHypsoStyleData);
    }
  });

  api.hooks.onGameEnd(teardownHypsoSwap);
})();
```

5. Append the above into `index.js`:

```batch
type "%USERPROFILE%\Desktop\SB_mod_project\02_intermediate\hypsometric-tinting_index-js.js" >> "%USERPROFILE%\Desktop\SB_mod_project\03_mod\<CITY_CODE_LOWER>-mod\index.js"
```

### 7e. Legend for elevation layers

The Legend helps in identifying each map layer using a representative icon and short description. To avoid too much clutter, toggles are also added so layers can be turned on and off at any time during the game.

1. Run below in cmd:

```batch
notepad "%USERPROFILE%\Desktop\SB_mod_project\02_intermediate\legend1_index-js.js"
```

2. Paste into the notepad:

```javascript


/*----------------------------------------------------
-------------------- Legend Panel --------------------
----------------------------------------------------*/

(function () {
  const api = window.SubwayBuilderAPI;
  const React = api.utils.React;
  const h = React.createElement;

  window.__<CITY_CODE_LOWER>LegendToggleState = window.__<CITY_CODE_LOWER>LegendToggleState || {}; // { [category]: { checkedMap, categoryOn } } — survives hot-reload/panel remounts, same reason __<CITY_CODE_LOWER>Patterns/__<CITY_CODE_LOWER>Icons are window-scoped

  function readBackgroundLuminance(el) {
    if (!el) return null;
    const c = getComputedStyle(el).backgroundColor;
    const m = c && c.match(/rgba?\(([^)]+)\)/);
    if (!m) return null;
    const parts = m[1].split(',').map(function (s) { return parseFloat(s); });
    const a = parts.length > 3 ? parts[3] : 1;
    if (a === 0) return null;
    return 0.299 * parts[0] + 0.587 * parts[1] + 0.114 * parts[2];
  }

  function isLegendDarkMode() {
    const attr = document.documentElement.getAttribute('data-theme');
    if (attr === 'dark') return true;
    if (attr === 'light') return false;
    const bodyLum = readBackgroundLuminance(document.body);
    if (bodyLum !== null) return bodyLum < 128;
    const htmlLum = readBackgroundLuminance(document.documentElement);
    if (htmlLum !== null) return htmlLum < 128;
    return !!(window.matchMedia && window.matchMedia('(prefers-color-scheme: dark)').matches);
  }

  const legendData = [
    {
      category: 'Terrain',
      items: [
        { label: 'Hillshade', color: '#8a8a8a', outline: '#8a8a8a', layerIds: ['<CITY_CODE_LOWER>-hillshade'], independentToggle: true, defaultOn: true },
        { label: 'Hypsometric Tinting', color: '#7c9c5c', outline: '#5c7a42', layerIds: ['<CITY_CODE_LOWER>-hypsometric', '<CITY_CODE_LOWER>-hypsometric-dark'], independentToggle: true, defaultOn: true },
        { label: 'Contour Lines', color: '#a67c52', outline: '#a67c52', layerIds: ['<CITY_CODE_LOWER>-contours', '<CITY_CODE_LOWER>-contours-labels'], independentToggle: true, defaultOn: false },  // 'defaultOn: false' turns this layer off on game start - should be combined with 'layout: { visibility: 'none' }' in the registerLayer section
        { label: '3D Terrain (hides tracks/stations)', color: '#6a5acd', outline: '#4a3d9e', independentToggle: true, isTerrainToggle: true, defaultOn: false, dependsOnSiblings: true }
      ]
    }
  ];

  function Swatch(props) {
    const item = props.item;
    const dark = props.dark;
    const isTransparent = item.color === 'transparent' || item.color === 'rgba(0,0,0,0)';
    return h('span', {
      style: Object.assign(
        {
          width: 14, height: 14, flexShrink: 0, borderRadius: 3,
          border: '1px solid ' + (item.outline || item.color)
        },
        isTransparent
          ? { background: dark ? 'rgba(255,255,255,0.08)' : undefined }
          : { background: item.color }
      )
    });
  }

  function LegendRow(props) {
    const item = props.item;
    const checked = props.checked;
    const disabled = props.disabled;
    const dark = props.dark;

    return h(
      'div',
      { style: { display: 'flex', alignItems: 'center', gap: 6, padding: '2px 0', opacity: disabled ? 0.45 : 1 } },
      h(Swatch, { item: item, dark: dark }),
      h('span', { style: { fontSize: 12, flex: 1, color: dark ? '#e8e8e8' : '#1a1a1a' } }, item.label),
      h(api.utils.components.Switch, {
        checked: checked,
        disabled: disabled,
        onCheckedChange: props.onToggle,
        style: { marginLeft: 'auto', transform: 'scale(0.75)' }
      })
    );
  }

  const BOUNDARY_ROW_COLORS = {};

  function CategoryGroup(props) {
    const group = props.group;
    const dark = props.dark;

    const openState = React.useState(group.defaultOpen === false ? false : true);
    const open = openState[0];
    const setOpen = openState[1];

    const saved = window.__<CITY_CODE_LOWER>LegendToggleState[group.category];

    const initial = {};
    group.items.forEach(function (item) {
      initial[item.label] = item.defaultOn === false ? false : true;
    });
    if (saved && saved.checkedMap) {
      Object.assign(initial, saved.checkedMap);
    }
    const checkedState = React.useState(initial);
    const checkedMap = checkedState[0];
    const setCheckedMap = checkedState[1];

    const siblingsOnCount = group.items.filter(function (item) {
      return !item.isTerrainToggle && checkedMap[item.label];
    }).length;

    function applyToggle(item, val) {
      const map = api.utils.getMap();
      if (!map) return;

      if (item.isTerrainToggle) {
        if (val) {
          map.setTerrain({ source: '<CITY_CODE_LOWER>-hillshade-source', exaggeration: 1.0 });
        } else {
          map.setTerrain(null);
        }
        return;
      }

      if (item.label === 'Hypsometric Tinting' && item.layerIds) {
        const dark = document.documentElement.classList.contains('dark')
          ? true
          : document.documentElement.classList.contains('light')
          ? false
          : !!(window.matchMedia && window.matchMedia('(prefers-color-scheme: dark)').matches);
        item.layerIds.forEach(function (id) {
          const isDarkLayer = id.indexOf('-dark') !== -1;
          map.setLayoutProperty(id, 'visibility', (val && (isDarkLayer === dark)) ? 'visible' : 'none');
        });
        return;
      }

      if (item.layerIds) {
        item.layerIds.forEach(function (id) {
          map.setLayoutProperty(id, 'visibility', val ? 'visible' : 'none');
        });
      }
    }

    React.useEffect(function () {
      group.items.forEach(function (item) {
        if (item.independentToggle) applyToggle(item, checkedMap[item.label]);
      });
    }, []);

    function handleToggle(item, val) {
      const next = Object.assign({}, checkedMap);
      next[item.label] = val;

      if (!item.isTerrainToggle && !val) {
        const stillOnCount = group.items.filter(function (other) {
          return !other.isTerrainToggle && next[other.label];
        }).length;
        const terrainItem = group.items.filter(function (other) { return other.isTerrainToggle; })[0];
        if (terrainItem && stillOnCount === 0 && next[terrainItem.label]) {
          next[terrainItem.label] = false;
          applyToggle(terrainItem, false);
        }
      }

      setCheckedMap(next);
      window.__<CITY_CODE_LOWER>LegendToggleState[group.category] = Object.assign({}, window.__<CITY_CODE_LOWER>LegendToggleState[group.category], { checkedMap: next });
      applyToggle(item, val);
    }

    return h(
      'div',
      { style: { marginBottom: 8 } },
      h(
        'button',
        {
          onClick: function () { setOpen(!open); },
          style: {
            width: '100%', textAlign: 'left', background: 'transparent', border: 'none',
            color: dark ? '#f2f2f2' : '#1a1a1a', fontWeight: 600, fontSize: 12, padding: '4px 0',
            cursor: 'pointer', opacity: 0.9
          }
        },
        (open ? '▾ ' : '▸ ') + group.category
      ),
      open ? h('div', { style: { paddingLeft: 4 } }, group.items.map(function (item) {
        const isDisabled = !!(item.dependsOnSiblings && siblingsOnCount === 0);
        const rowItem = BOUNDARY_ROW_COLORS[item.label]
          ? Object.assign({}, item, {
              outline: dark ? BOUNDARY_ROW_COLORS[item.label].dark : item.outline,
              color: (item.color === 'transparent') ? item.color : (dark ? BOUNDARY_ROW_COLORS[item.label].dark : item.color)
            })
          : item;
        return h(LegendRow, {
          key: item.label,
          item: rowItem,
          checked: checkedMap[item.label],
          disabled: isDisabled,
          dark: dark,
          onToggle: function (val) { handleToggle(item, val); }
        });
      })) : null
    );
  }

  function LegendPanel() {
    const noticeState = React.useState('');
    const notice = noticeState[0];
    const setNotice = noticeState[1];
    const rootRef = React.useRef(null);

    const themeState = React.useState(isLegendDarkMode());
    const dark = themeState[0];
    const setDark = themeState[1];

    function applyPanelChrome(isDark) {
      let el = rootRef.current;
      let steps = 0;
      while (el && steps < 8) {
        const cls = typeof el.className === 'string' ? el.className : '';
        if (cls.indexOf('bg-background') !== -1) {
          el.style.setProperty('background', isDark ? 'rgba(30,30,32,0.9)' : 'rgba(255,255,255,0.75)', 'important');
          el.style.setProperty('backdrop-filter', 'blur(4px)', 'important');
          el.style.setProperty('box-shadow', 'none', 'important');
        }
        el = el.parentElement;
        steps++;
      }
    }

    React.useEffect(function () {
      applyPanelChrome(dark);
    }, [dark]);

    React.useEffect(function () {
      function recheck() {
        setDark(isLegendDarkMode());
      }
      let mq;
      if (window.matchMedia) {
        mq = window.matchMedia('(prefers-color-scheme: dark)');
        mq.addEventListener('change', recheck);
      }
      const observer = new MutationObserver(recheck);
      observer.observe(document.documentElement, { attributes: true, attributeFilter: ['data-theme', 'class', 'style'] });
      observer.observe(document.body, { attributes: true, attributeFilter: ['data-theme', 'class', 'style'] });
      const pollId = setInterval(recheck, 1000);
      return function () {
        if (mq) mq.removeEventListener('change', recheck);
        observer.disconnect();
        clearInterval(pollId);
      };
    }, []);

    function findPanelRoot(node) {
      let el = node;
      let steps = 0;
      while (el && steps < 8) {
        const cls = typeof el.className === 'string' ? el.className : '';
        if (cls.indexOf('fixed') !== -1 && cls.indexOf('z-50') !== -1) {
          return el;
        }
        el = el.parentElement;
        steps++;
      }
      return null;
    }

    return h(
      'div',
      { ref: rootRef, style: { maxHeight: '100%', overflowY: 'auto', paddingRight: 6 } },

      legendData.map(function (group) {
        return h(CategoryGroup, { key: group.category, group: group, dark: dark });
      }),
      h('div', { style: { fontSize: 11, marginTop: 6, color: dark ? '#cfcfcf' : undefined } },
        h('strong', {}, 'Note:'),
        ' Reopen Legend after switching b/w light and dark modes to reapply the toggles applied at that moment.'
      ),
      h('button', {
        onClick: function () {
            const margin = 20;
            const width = 275;
            const height = 860;
            const x = Math.max(margin, window.innerWidth - width - margin);
            const y = 60;
            const DEFAULTS = { x: x, y: y, width: width, height: height };
            try {
              localStorage.setItem('floating-panel-<CITY_CODE_LOWER>-legend-panel', JSON.stringify(DEFAULTS));
            } catch (e) {}

            api.ui.addFloatingPanel({
              id: '<CITY_CODE_LOWER>-legend-panel',
              title: 'Legend',
              icon: 'Layers',
              defaultWidth: 275,
              defaultHeight: 860,
              minWidth: 200,
              minHeight: 150,
              render: function () { return h(LegendPanel); }
            });

            const panelRoot = findPanelRoot(rootRef.current);
            if (panelRoot) {
              panelRoot.style.setProperty('left', DEFAULTS.x + 'px', 'important');
              panelRoot.style.setProperty('top', DEFAULTS.y + 'px', 'important');
              panelRoot.style.setProperty('width', DEFAULTS.width + 'px', 'important');
              panelRoot.style.setProperty('height', DEFAULTS.height + 'px', 'important');
              setNotice('Position reset.');
            } else {
              setNotice('Could not find panel — close and reopen to apply.');
            }
          },
          style: {
            fontSize: 11, marginTop: 6, cursor: 'pointer', color: dark ? '#cfcfcf' : '#1a1a1a',
            padding: '3px 8px', borderRadius: 4,
            border: '1px solid ' + (dark ? 'rgba(255,255,255,0.25)' : 'rgba(0,0,0,0.25)'),
            background: dark ? 'rgba(255,255,255,0.06)' : 'rgba(0,0,0,0.04)'
          }
        }, 'Reset panel position'),
      notice ? h('div', { style: { fontSize: 11, marginTop: 4, opacity: 0.7, color: dark ? '#cfcfcf' : undefined } }, notice) : null
    );
  }

  let panelAdded = false;

  api.hooks.onMapReady(function () {
    if (panelAdded) return;
    panelAdded = true;
    api.ui.addFloatingPanel({
      id: '<CITY_CODE_LOWER>-legend-panel',
      title: 'Legend',
      icon: 'Layers',
      defaultWidth: 275,
      defaultHeight: 860,
      minWidth: 200,
      minHeight: 150,
      render: function () { return h(LegendPanel); }
    });
  });

  api.hooks.onGameEnd(function () {
    panelAdded = false;
  });
})();
```

The above code also adds a 3D terrain map layer. It makes for a nice visual map when it's turned on, if the area has hilly terrain. However, this layer causes issues during gameplay as game elements like stations and tracks disappear into the 3d terrain as they don't have a height element. So keep it turned off using the Legend toggle during gameplay.

3. Append the above into `index.js`:

```batch
type "%USERPROFILE%\Desktop\SB_mod_project\02_intermediate\legend1_index-js.js" >> "%USERPROFILE%\Desktop\SB_mod_project\03_mod\<CITY_CODE_LOWER>-mod\index.js"
```

### 7f. Test in game

1. Move the updated mod files to the respective game folder:

```batch
xcopy "%USERPROFILE%\Desktop\SB_mod_project\03_mod" "%APPDATA%\metro-maker4\mods"  /E /I /D /Y
```

2. Run the server again (if it was closed previously) and check if all the four elevation layers appear correctly.

----

## Stage 8: Land cover and land use map layers (Optional)

*Est. time: 90 minutes (mostly processing/downloading in the background)*

### 8a. Land cover and protected lands

The default map only shows water bodies, roads and buildings. We will additionally add forests, scrubs, grasslands/meadows, sanctuaries, protected areas, wetlands and intermittent water (which previously got mixed in with other water bodies). 

1. Backup maps.py:

```bash
cp ~/SB_mod_project/tools/depot/src/depot/maps.py ~/SB_mod_project/tools/depot/src/depot/maps-backup.py
```

2. Run the below code to add these additional features to the script (`maps.py`) that creates the base vector tile:

```python
cat > ~/SB_mod_project/02_intermediate/lclu_patch_1.py << 'PYEOF'
import os
path = "~/SB_mod_project/tools/depot/src/depot/maps.py"
with open(os.path.expanduser(path), "r") as f:
    lines = f.readlines()

def check(line_no, expected_substr):
    actual = lines[line_no - 1]
    if expected_substr not in actual:
        print(f"ABORTED: line {line_no} does not contain expected text.")
        print(f"  Expected to contain: {expected_substr!r}")
        print(f"  Actual line: {actual!r}")
        return False
    return True

anchors_ok = (
    check(1868, "def _get_kind_and_rank(self, val):") and
    check(1905, "water_geoms_to_dissolve = []") and
    check(1906, "water_id_map = []") and
    check(1928, "old_props.get('class')") and
    check(1957, "if not geom.is_empty:") and
    check(1960, "water_geoms_to_dissolve.append(geom)") and
    check(1961, "water_id_map.append((feature.get('id'), geom))") and
    check(1978, "elif is_bldg_layer or kind == 'building':") and
    check(1980, 'elif layer_name in ["transportation", "roads", "navigation"]') and
    check(1992, "elif kind == 'park':") and
    check(2007, "props['ref'] = old_props['ref']") and
    check(2017, "# Handle overlapping park features") and
    check(2070, "# Handle water features") and
    check(2127, "merged_result = None") and
    check(2129, "# Build aerodrome mask") and
    check(2157, "Clip landuse against the dissolved water mask") and
    check(2173, 'if kind not in ["park", "aerodrome"]') and
    check(2226, "# Subtract water and aerodrome from commercial") and
    check(2328, "def fix_mbtiles(self):") and
    check(2381, "out_conn.commit()")
)

if not anchors_ok:
    print("No changes written.")
else:
    meta_block = '''        cursor2 = out_conn.execute("SELECT value FROM metadata WHERE name = 'json'")
        json_val = cursor2.fetchone()[0]
        meta = json.loads(json_val)
        vlayers = meta.get('vector_layers', [])
        water_layer = next((l for l in vlayers if l['id'] == 'water'), None)
        if water_layer and not any(l['id'] == 'water_intermittent' for l in vlayers):
            new_layer = dict(water_layer)
            new_layer['id'] = 'water_intermittent'
            vlayers.append(new_layer)
            meta['vector_layers'] = vlayers
            out_conn.execute(
                "UPDATE metadata SET value = ? WHERE name = 'json'",
                (json.dumps(meta),)
            )

'''
    lines[2380:2380] = [meta_block]

    clip_block = '''
        clippable_landuse_kinds = [
            "forest", "sanctuary", "protected_area", "scrub", "wetland",
            "grassland", "aerodrome", "park"
        ]
        if "landuse" in new_layers_data:
            kept = []
            for feat in new_layers_data["landuse"]:
                kind = feat["properties"].get("kind")

                if kind not in clippable_landuse_kinds:
                    kept.append(feat)
                    continue

                geom = shape(feat["geometry"])
                geom = geom.intersection(tile_bounds)

                geom = set_precision(geom, grid_size=1.0)

                if geom.is_empty:
                    continue
                if not geom.is_valid:
                    geom = geom.buffer(0)

                if merged_result is not None and not merged_result.is_empty:
                    if geom.intersects(merged_result):
                        geom = geom.difference(merged_result)
                        if not geom.is_valid:
                            geom = geom.buffer(0)

                if kind != "aerodrome" and aerodrome_mask is not None and not aerodrome_mask.is_empty:
                    if geom.intersects(aerodrome_mask):
                        geom = geom.difference(aerodrome_mask)
                        if not geom.is_valid:
                            geom = geom.buffer(0)

                if kind != "aerodrome" and commercial_mask is not None and not commercial_mask.is_empty:
                    if geom.intersects(commercial_mask):
                        geom = geom.difference(commercial_mask)
                        if not geom.is_valid:
                            geom = geom.buffer(0)

                if geom.is_empty or geom.area < 1.0:
                    continue

                if geom.geom_type not in ("Polygon", "MultiPolygon"):
                    parts = [g for g in geom.geoms
                             if g.geom_type in ("Polygon", "MultiPolygon")]
                    if not parts:
                        continue
                    geom = unary_union(parts) if len(parts) > 1 else parts[0]

                feat["geometry"] = mapping(geom)
                kept.append(feat)

            new_layers_data["landuse"] = kept

'''
    lines[2156:2225] = [clip_block]

    intermittent_block = '''        if intermittent_geoms_to_dissolve:
            snapped_geoms = [set_precision(g, grid_size=0.1) \\
                             for g in intermittent_geoms_to_dissolve]
            merged_result_int = unary_union([g.buffer(0.5) for g in snapped_geoms])
            merged_result_int = merged_result_int.buffer(-0.5)
            merged_result_int = set_precision(merged_result_int, grid_size=1.0)
            if not merged_result_int.is_valid:
                merged_result_int = merged_result_int.buffer(0)
            merged_result_int = merged_result_int.intersection(tile_bounds)
            final_parts_int = []
            if isinstance(merged_result_int, Polygon):
                final_parts_int.append(merged_result_int)
            elif isinstance(merged_result_int, MultiPolygon):
                final_parts_int.extend(list(merged_result_int.geoms))
            elif hasattr(merged_result_int, 'geoms'):
                for g in merged_result_int.geoms:
                    if isinstance(g, Polygon):
                        final_parts_int.append(g)
                    elif isinstance(g, MultiPolygon):
                        final_parts_int.extend(list(g.geoms))
            if "water_intermittent" not in new_layers_data:
                new_layers_data["water_intermittent"] = []
            for part in final_parts_int:
                if part.is_empty or part.area < 0.01:
                    continue
                part = orient(part, sign=1.0)
                associated_ids = []
                for orig_id, orig_geom in intermittent_id_map:
                    if part.intersects(orig_geom):
                        associated_ids.append(orig_id)
                primary_id = associated_ids[0] if associated_ids else None
                intermittent_feat = {
                    "geometry": mapping(part),
                    "properties": {"kind": "water_intermittent", "sort_rank": 200},
                    "type": "Polygon"
                }
                if primary_id is not None:
                    intermittent_feat["id"] = primary_id
                new_layers_data["water_intermittent"].append(intermittent_feat)

'''
    lines[2127:2127] = [intermittent_block]

    dissolve_block = '''        dissolvable_kinds = set()
        geoms_by_kind = {}
        base_props_by_kind = {}
        if "landuse" in new_layers_data:
            kept_landuse_feats = []
            for feat in new_layers_data["landuse"]:
                kind = feat["properties"].get("kind")
                if kind in dissolvable_kinds:
                    geom = shape(feat["geometry"])
                    if not geom.is_empty:
                        if not geom.is_valid:
                            geom = geom.buffer(0)
                        geoms_by_kind.setdefault(kind, []).append(geom)
                    base_props_by_kind[kind] = feat["properties"]
                else:
                    kept_landuse_feats.append(feat)
            for kind, geoms in geoms_by_kind.items():
                if not geoms:
                    continue
                merged = unary_union(geoms)
                merged = merged.buffer(4).buffer(-4)
                merged = shapely.make_valid(merged)
                if merged.geom_type not in ("Polygon", "MultiPolygon"):
                    parts = [g for g in merged.geoms
                             if g.geom_type in ("Polygon", "MultiPolygon")]
                    if not parts:
                        merged = None
                    else:
                        merged = unary_union(parts) if len(parts) > 1 else parts[0]
                if merged and not merged.is_empty and merged.area >= 0.01:
                    kept_landuse_feats.append({
                        "geometry": mapping(merged),
                        "properties": base_props_by_kind[kind],
                        "type": merged.geom_type
                    })
            new_layers_data["landuse"] = kept_landuse_feats

'''
    lines[2016:2069] = [dissolve_block]

    dispatch_block = (
        "                elif kind in ('forest', 'sanctuary', 'protected_area', 'scrub', 'wetland', 'grassland', 'park'):\n"
        "                    dest, final_kind, final_rank = \"landuse\", kind, rank\n"
    )
    lines[1991:1993] = [dispatch_block]

    lines[2007:2007] = ["                if 'name' in old_props: props['name'] = old_props['name']\n"]

    append_block = '''                    if not geom.is_empty:
                        if not geom.is_valid:
                            geom = geom.buffer(0)
                        if old_props.get('intermittent') == 1:
                            intermittent_geoms_to_dissolve.append(geom)
                            intermittent_id_map.append((feature.get('id'), geom))
                        else:
                            water_geoms_to_dissolve.append(geom)
                            water_id_map.append((feature.get('id'), geom))
'''
    lines[1956:1961] = [append_block]

    caller_block = (
        "                kind, detail, rank = self._get_kind_and_rank(\n"
        "                    old_props.get('aeroway') or old_props.get('class') or \"\",\n"
        "                    old_props.get('subclass')\n"
        "                )\n"
    )
    lines[1926:1929] = [caller_block]

    decl_block = '''        intermittent_geoms_to_dissolve = []
        intermittent_id_map = []
'''
    lines[1906:1906] = [decl_block]

    func_block = '''    def _get_kind_and_rank(self, val, subclass_val=None):
        priority = {
            'aeroway': 400, 'river': 200, 'park': 189, 'aerodrome': 189
        }
        if not isinstance(val, str): return 'other', None, 0
        v = val.lower()
        sv = subclass_val.lower() if isinstance(subclass_val, str) else None
        check = sv if sv else v

        if 'runway' in v:
            return 'aeroway', 'runway', priority['aeroway']
        if 'taxiway' in v:
            return 'aeroway', 'taxiway', priority['aeroway']
        if 'river' in v:
            return 'river', None, priority['river']

        if 'protected_area' in check:
            return 'protected_area', None, priority['park']
        if any(x in check for x in ['nature_reserve', 'wilderness_area',
                                'wildlife_sanctuary', 'state_forest',
                                'national_wildlife_refuge', 'management_area',
                                'wildlife_management_area']):
            return 'sanctuary', None, priority['park']
        if any(x in check for x in ['wood', 'forest']):
            return 'forest', None, priority['park']
        if 'scrub' in check:
            return 'scrub', None, priority['park']
        if 'wetland' in v or 'wetland' in check:
            return 'wetland', None, priority['park']
        if any(x in check for x in ['grass', 'meadow']):
            return 'grassland', None, priority['park']

        if any(x in v for x in ['park', 'cemetery', 'pitch', 'zoo']):
            return 'park', None, priority['park']
        if 'aerodrome' in v or \\
           ('military' in v and self.color_military_like_aerodrome):
            return 'aerodrome', None, priority['aerodrome']
        return v, None, 0

'''
    lines[1867:1895] = [func_block]

    with open(os.path.expanduser(path), "w") as f:
        f.writelines(lines)
    print("SUCCESS: all 10 edits applied and file written.")
PYEOF
python3 ~/SB_mod_project/02_intermediate/lclu_patch_1.py
```

3. See if the below returns '*Exit code: 0*':

```bash
python3 -m py_compile ~/SB_mod_project/tools/depot/src/depot/maps.py
echo "Exit code: $?"
```

### 8b. Railways, agriculture, recreation, and quarries

Railway tracks, buildings and lands, farmlands, orchards, quarries, recreational areas and beaches.

1. Run the below code:

```python
cat > ~/SB_mod_project/02_intermediate/lclu_patch_2.py << 'PYEOF'
import os
path = "~/SB_mod_project/tools/depot/src/depot/maps.py"
with open(os.path.expanduser(path), "r") as f:
    lines = f.readlines()

def check(line_no, expected_substr):
    actual = lines[line_no - 1]
    if expected_substr not in actual:
        print(f"ABORTED: line {line_no} does not contain expected text.")
        print(f"  Expected to contain: {expected_substr!r}")
        print(f"  Actual line: {actual!r}")
        return False
    return True

anchors_ok = (
    check(1868, "def _get_kind_and_rank(self, val, subclass_val=None):") and
    check(1877, "if 'runway' in v:") and
    check(1895, "if 'wetland' in v or 'wetland' in check:") and
    check(1900, "if any(x in v for x in ['park', 'cemetery', 'pitch', 'zoo']):") and
    check(1905, "return v, None, 0") and
    check(1996, "elif is_bldg_layer or kind == 'building':") and
    check(1997, 'dest, final_kind, final_rank = "buildings", "building", 400') and
    check(1998, 'elif layer_name in ["transportation", "roads", "navigation"]:') and
    check(2010, "elif kind in ('forest', 'sanctuary', 'protected_area', 'scrub', 'wetland', 'grassland', 'park'):") and
    check(2201, "clippable_landuse_kinds = [") and
    check(2202, '"forest", "sanctuary", "protected_area", "scrub", "wetland",') and
    check(2203, '"grassland", "aerodrome", "park"') and
    check(2204, "]")
)

if not anchors_ok:
    print("No changes written.")
else:
    clippable_block = '''            "forest", "sanctuary", "protected_area", "scrub", "wetland",
            "grassland", "recreation", "agricultural", "orchard", "quarry",
            "aerodrome", "beach", "railway"
'''
    lines[2201:2203] = [clippable_block]

    dispatch_block = "                elif kind in ('forest', 'sanctuary', 'protected_area', 'scrub', 'wetland', 'grassland', 'recreation', 'agricultural', 'orchard', 'quarry', 'beach', 'railway'):\n"
    lines[2009:2010] = [dispatch_block]

    railway_block = '''                elif layer_name == "transportation" and kind in ("rail", "narrow_gauge", "light_rail", "subway"):
                    geom = shape(feature['geometry']).intersection(tile_bounds)
                    if geom.geom_type not in ("LineString", "MultiLineString"):
                        parts = [g for g in getattr(geom, "geoms", [geom])
                                 if g.geom_type in ("LineString", "MultiLineString")]
                        if not parts:
                            continue
                        geom = unary_union(parts) if len(parts) > 1 else parts[0]
                    if geom.is_empty:
                        continue
                    feature['geometry'] = mapping(geom)
                    dest, final_kind, final_rank = "railway_lines", kind, 400
'''
    lines[1997:1997] = [railway_block]

    kind_rank_block = '''        if 'runway' in check:
            return 'aeroway', 'runway', priority['aeroway']
        if 'taxiway' in check:
            return 'aeroway', 'taxiway', priority['aeroway']
        if 'river' in check:
            return 'river', None, priority['river']

        if 'protected_area' in check:
            return 'protected_area', None, priority['park']
        if any(x in check for x in ['nature_reserve', 'wilderness_area',
                                'wildlife_sanctuary', 'state_forest',
                                'national_wildlife_refuge', 'management_area',
                                'wildlife_management_area']):
            return 'sanctuary', None, priority['park']
        if any(x in check for x in ['wood', 'forest']):
            return 'forest', None, priority['park']
        if 'scrub' in check:
            return 'scrub', None, priority['park']
        if 'wetland' in v or 'wetland' in check:
            return 'wetland', None, priority['park']
        if any(x in check for x in ['grass', 'meadow']):
            return 'grassland', None, priority['park']

        if any(x in check for x in ['park', 'pitch', 'zoo', 'recreation_ground', 'cemetery']):
            return 'recreation', None, priority['park']
        if 'farmland' in check:
            return 'agricultural', None, priority['park']
        if 'orchard' in check:
            return 'orchard', None, priority['park']
        if 'beach' in check:
            return 'beach', None, priority['park']
        if 'railway' in check:
            return 'railway', None, priority['park']
        if 'quarry' in check:
            return 'quarry', None, priority['park']
        if 'aerodrome' in check or \\
           ('military' in check and self.color_military_like_aerodrome):
            return 'aerodrome', None, priority['aerodrome']
        return check, None, 0
'''
    lines[1876:1905] = [kind_rank_block]

    with open(os.path.expanduser(path), "w") as f:
        f.writelines(lines)
    print("SUCCESS: all 4 edits applied and file written.")
PYEOF
python3 ~/SB_mod_project/02_intermediate/lclu_patch_2.py
```

2. See if the below returns '*Exit code: 0*':

```bash
python3 -m py_compile ~/SB_mod_project/tools/depot/src/depot/maps.py
echo "Exit code: $?"
```

### 8c. Railway stations and power plants/generators

Run the below code:

```python
cat >> ~/SB_mod_project/02_intermediate/depot/map.py << 'EOF'
import json
from shapely.geometry import shape, mapping

class CityMapGen(MapGen):
    def add_power_and_stations(self, dedup_threshold_deg=0.0005):
        path_prefix = os.path.join(self.city_dir, self.city)
        final_output = f"{path_prefix}.pmtiles"
        power_stations_pmtiles = os.path.join(self.city_dir, "power_stations_only.pmtiles")
        updated_mbtiles = f"{path_prefix}-with-power.mbtiles"

        def extract(tag_filter_args, out_prefix):
            pbf = os.path.join(self.city_dir, f"{out_prefix}.osm.pbf")
            geojson = os.path.join(self.city_dir, f"{out_prefix}.geojson")
            self._run_command(["osmium", "tags-filter", self.city_osmpbf,
                               *tag_filter_args, "-o", pbf, "--overwrite"])
            self._run_command(["osmium", "export", pbf, "-o", geojson,
                               "--overwrite"])
            with open(geojson) as f:
                data = json.load(f)
            if self.cleanup_files:
                os.remove(pbf)
                os.remove(geojson)
            return data.get('features', [])

        def dedup_points(node_feats, area_feats, kind_label, source_tag=None):
            node_points = []
            out_features = []
            for feat in node_feats:
                if feat['geometry']['type'] != 'Point':
                    continue
                geom = shape(feat['geometry'])
                node_points.append(geom)
                props = {
                    "kind": kind_label,
                    "name": feat.get('properties', {}).get('name', '')
                }
                if source_tag:
                    props["source"] = feat.get('properties', {}).get(source_tag, '')
                out_features.append({
                    "type": "Feature", "geometry": mapping(geom),
                    "properties": props
                })
            for feat in area_feats:
                geom = shape(feat['geometry'])
                if geom.geom_type not in ("Polygon", "MultiPolygon"):
                    continue
                c = geom.centroid
                if any(c.distance(p) <= dedup_threshold_deg for p in node_points):
                    continue
                props = {
                    "kind": kind_label,
                    "name": feat.get('properties', {}).get('name', '')
                }
                if source_tag:
                    props["source"] = feat.get('properties', {}).get(source_tag, '')
                out_features.append({
                    "type": "Feature", "geometry": mapping(c),
                    "properties": props
                })
            return out_features

        plant_area_feats = extract(["w/power=plant", "r/power=plant"], "power_plant_areas")
        plant_polygon_features = []
        for feat in plant_area_feats:
            geom = shape(feat['geometry'])
            if geom.geom_type in ("Polygon", "MultiPolygon"):
                plant_polygon_features.append({
                    "type": "Feature", "geometry": mapping(geom),
                    "properties": {
                        "kind": "power_plant",
                        "name": feat.get('properties', {}).get('name', ''),
                        "source": feat.get('properties', {}).get('plant:source', '')
                    }
                })

        gen_node_feats = extract(["n/power=generator"], "power_gen_nodes")
        gen_area_feats = extract(["w/power=generator", "r/power=generator"], "power_gen_areas")
        generator_features = dedup_points(gen_node_feats, gen_area_feats, "power_generator", source_tag="generator:source")

        station_node_feats = extract(["n/railway=station"], "station_nodes")
        station_area_feats = extract(["w/railway=station", "r/railway=station"], "station_areas")
        station_features = dedup_points(station_node_feats, station_area_feats, "station")

        plants_geojson = os.path.join(self.city_dir, "power_plants.geojson")
        generators_geojson = os.path.join(self.city_dir, "power_generators.geojson")
        stations_geojson = os.path.join(self.city_dir, "stations.geojson")

        with open(plants_geojson, 'w') as f:
            json.dump({"type": "FeatureCollection",
                       "features": plant_polygon_features}, f)
        with open(generators_geojson, 'w') as f:
            json.dump({"type": "FeatureCollection", "features": generator_features}, f)
        with open(stations_geojson, 'w') as f:
            json.dump({"type": "FeatureCollection", "features": station_features}, f)

        if self.verb:
            from collections import Counter
            print(f"***** power_plants: {len(plant_polygon_features)} polygons *****")
            print(f"    by plant:source: {dict(Counter(f['properties']['source'] for f in plant_polygon_features))}")
            print(f"***** power_generators: {len(generator_features)} points *****")
            print(f"    by generator:source: {dict(Counter(f['properties']['source'] for f in generator_features))}")
            print(f"***** stations: {len(station_features)} points *****")

        bbox_clean = ",".join(map(str, self.bbox))
        self._run_command([
            "tippecanoe", "-Z", "6", "-z", f"{self.maxzoom}", "-r", "1",
            "-o", power_stations_pmtiles, "--no-tile-size-limit",
            f"--clip-bounding-box={bbox_clean}", "--force",
            "-L", f"power_plants:{plants_geojson}",
            "-L", f"power_generators:{generators_geojson}",
            "-L", f"stations:{stations_geojson}",
        ])
        final_output_stripped = updated_mbtiles.replace('.mbtiles', '_stripped.mbtiles')
        self._run_command(["tile-join", "-o", final_output_stripped,
                           "-L", "power_plants", "-L", "power_generators", "-L", "stations",
                           final_output,
                           "--no-tile-size-limit", "--force"])
        self._run_command(["tile-join", "-o", updated_mbtiles,
                           final_output_stripped, power_stations_pmtiles,
                           "--no-tile-size-limit", "--force"])
        self._update_mbtiles_metadata(updated_mbtiles)
        self._run_command(["pmtiles", "convert", updated_mbtiles, final_output])

        if self.cleanup_files:
            os.remove(plants_geojson)
            os.remove(generators_geojson)
            os.remove(stations_geojson)
            os.remove(power_stations_pmtiles)
            os.remove(updated_mbtiles)
            os.remove(final_output_stripped)

        if self.verb:
            print(f"***** Done. Power/stations layers merged into: *****")
            print(f"    {final_output}")

obj.__class__ = CityMapGen
obj.add_power_and_stations()
EOF
```

### 8d. Run the script to create the base vector tile:

1. Run the below in Debian:

```python
python3 ~/SB_mod_project/02_intermediate/depot/map.py
```

Rerun it if getting a timeout error.

2. Move the file to Windows using cmd:

```batch
scp <VM_USER>@<VM_IP>:~/SB_mod_project/03_output/<CITY_CODE>/<CITY_CODE>.pmtiles "%USERPROFILE%\Desktop\SB_mod_project\03_mod\<CITY_CODE_LOWER>-mod\tiles"
```

### 8e. Updating `index.js` with changes made in this stage

1. Run below in cmd:

```batch
notepad "%USERPROFILE%\Desktop\SB_mod_project\02_intermediate\lclu-1_index-js.js"
```

2. Paste into the notepad:

```javascript


/*----------------------------------------------------
---------------- Land Cover & Land Use ---------------
----------------------------------------------------*/

window.SubwayBuilderAPI.map.registerSource('<CITY_CODE_LOWER>landuse-source', {
  type: 'vector',
  tiles: ['http://127.0.0.1:8080/<CITY_CODE>/{z}/{x}/{y}.mvt'],
  maxzoom: 15
});

(function () {
  const api = window.SubwayBuilderAPI;

  window.__<CITY_CODE_LOWER>Patterns = window.__<CITY_CODE_LOWER>Patterns || {};
  window.__<CITY_CODE_LOWER>PatternDefs = window.__<CITY_CODE_LOWER>PatternDefs || null; // { key: { layerId, imageId } } -- built once, reused on re-wire

  let patternRetryTimers = [];
  let patternStyleTimer = null;
  let patternStyleListenerMap = null;

  function clearPatternRetryTimers() {
    patternRetryTimers.forEach(function (id) { clearTimeout(id); });
    patternRetryTimers = [];
  }

  function teardownPatterns() {
    clearPatternRetryTimers();
    if (patternStyleTimer) { clearTimeout(patternStyleTimer); patternStyleTimer = null; }
    if (patternStyleListenerMap) { patternStyleListenerMap.off('styledata', onPatternStyleData); patternStyleListenerMap = null; }
  }

  function onPatternStyleData() {
    clearTimeout(patternStyleTimer);
    patternStyleTimer = setTimeout(rewireAllPatterns, 150);
  }

  function rewireAllPatterns() {
    const map = api.utils.getMap();
    if (!map || !window.__<CITY_CODE_LOWER>PatternDefs) return;
    Object.keys(window.__<CITY_CODE_LOWER>PatternDefs).forEach(function (key) {
      const def = window.__<CITY_CODE_LOWER>PatternDefs[key];
      wireFillPattern(map, def.layerId, def.imageId);
    });
  }

  function wireFillPattern(map, layerId, imageId, attemptsLeft) {
    if (attemptsLeft === undefined) attemptsLeft = 50;
    if (!map.getLayer(layerId)) {
      if (attemptsLeft > 0) {
        const id = setTimeout(function () { wireFillPattern(map, layerId, imageId, attemptsLeft - 1); }, 100);
        patternRetryTimers.push(id);
      }
      return;
    }
    map.setPaintProperty(layerId, 'fill-pattern', imageId);
  }

  api.hooks.onMapReady(function () {
    teardownPatterns();

    const map = api.utils.getMap();
    if (!map) return;

    window.__<CITY_CODE_LOWER>PatternDefs = window.__<CITY_CODE_LOWER>PatternDefs || {};

    function makePattern(key, layerId, size, bgColor, drawFn) {
      const imageId = '<CITY_CODE_LOWER>' + key + '-pattern';
      const canvas = document.createElement('canvas');
      canvas.width = size;
      canvas.height = size;
      const ctx = canvas.getContext('2d');
      if (bgColor) {
        ctx.fillStyle = bgColor;
        ctx.fillRect(0, 0, size, size);
      }
      drawFn(ctx);
      window.__<CITY_CODE_LOWER>Patterns[key] = canvas.toDataURL();
      const imgData = ctx.getImageData(0, 0, size, size);
      if (map.hasImage(imageId)) map.removeImage(imageId);
      map.addImage(imageId, imgData);
      window.__<CITY_CODE_LOWER>PatternDefs[key] = { layerId: layerId, imageId: imageId };
      wireFillPattern(map, layerId, imageId);
    }

    makePattern('forest', '<CITY_CODE_LOWER>forest', 32, '#add19e', function (ctx) {
      ctx.fillStyle = '#1f5c2c';
      function tree(x, y) {
        ctx.beginPath();
        ctx.moveTo(x, y - 9);
        ctx.lineTo(x - 5, y - 2);
        ctx.lineTo(x + 5, y - 2);
        ctx.closePath();
        ctx.fill();
        ctx.beginPath();
        ctx.moveTo(x, y - 5);
        ctx.lineTo(x - 4, y + 2);
        ctx.lineTo(x + 4, y + 2);
        ctx.closePath();
        ctx.fill();
        ctx.fillRect(x - 1, y + 2, 2, 3);
      }
      tree(9, 22);
      tree(23, 10);
    });

    makePattern('scrub', '<CITY_CODE_LOWER>scrub', 32, '#c8d7ab', function (ctx) {
      ctx.fillStyle = '#4a6b2f';
      function shrub(x, y) {
        [[0,0],[-2.6,1.8],[2.6,1.8]].forEach(function (o) {
          ctx.beginPath();
          ctx.arc(x + o[0], y + o[1], 2, 0, Math.PI * 2);
          ctx.fill();
        });
      }
      shrub(8, 10); shrub(23, 7); shrub(15, 23); shrub(26, 26);
    });

    makePattern('orchard', '<CITY_CODE_LOWER>orchard', 32, '#aedfa3', function (ctx) {
      ctx.fillStyle = '#1f6e1a';
      function fruitTree(x, y) {
        ctx.beginPath();
        ctx.arc(x, y - 2, 3, 0, Math.PI * 2);
        ctx.fill();
        ctx.fillRect(x - 0.7, y + 1, 1.4, 2.5);
      }
      [[8,8],[24,8],[8,24],[24,24],[16,16],[0,0],[0,32],[32,0],[32,32]].forEach(function (p) {
        fruitTree(p[0], p[1]);
      });
    });

    makePattern('quarry', '<CITY_CODE_LOWER>quarry', 32, '#c5c3c3', function (ctx) {
      ctx.fillStyle = '#4a4744';
      function rock(x, y, s) {
        ctx.beginPath();
        ctx.moveTo(x - 3 * s, y + 2 * s);
        ctx.lineTo(x - 1 * s, y - 3 * s);
        ctx.lineTo(x + 2 * s, y - 2 * s);
        ctx.lineTo(x + 3 * s, y + 1 * s);
        ctx.lineTo(x + 1 * s, y + 3 * s);
        ctx.closePath();
        ctx.fill();
      }
      rock(7, 8, 1.1); rock(21, 5, 0.9); rock(27, 15, 1.0);
      rock(10, 21, 0.85); rock(22, 25, 1.15); rock(4, 27, 0.9);
    });

    makePattern('beach', '<CITY_CODE_LOWER>beach', 32, '#fff1ba', function (ctx) {
      ctx.strokeStyle = '#b8935a';
      ctx.lineWidth = 1.2;
      function ripple(x, y, r) {
        ctx.beginPath();
        ctx.arc(x, y, r, 0.15 * Math.PI, 0.85 * Math.PI);
        ctx.stroke();
      }
      ripple(6, 6, 3); ripple(22, 4, 3); ripple(14, 14, 3.2); ripple(28, 16, 3);
      ripple(4, 22, 3); ripple(19, 24, 3.2); ripple(10, 29, 2.8);
    });

    makePattern('wetland', '<CITY_CODE_LOWER>wetland', 32, null, function (ctx) {
      ctx.strokeStyle = '#1f5566';
      ctx.lineWidth = 1.4;
      [[8,24],[22,20],[14,10],[26,8]].forEach(function (p) {
        const x = p[0], y = p[1];
        ctx.beginPath();
        ctx.moveTo(x, y + 5);
        ctx.lineTo(x, y - 2);
        ctx.moveTo(x - 2, y - 2);
        ctx.lineTo(x, y - 5);
        ctx.moveTo(x + 2, y - 2);
        ctx.lineTo(x, y - 5);
        ctx.stroke();
      });
    });

    patternStyleListenerMap = map;
    map.on('styledata', onPatternStyleData);
  });

  api.hooks.onGameEnd(teardownPatterns);
})();

/*------------------- Land Cover -------------------*/
var LANDCOVER_KINDS = [
  { kind: 'forest',         id: 'forest',         fillColor: '#add19e', fillOpacity: 0.4,  lineColor: '#2d6a35', lineWidth: 0.5 },
  { kind: 'scrub',          id: 'scrub',          fillColor: '#c8d7ab', fillOpacity: 0.4,  lineColor: '#7daa5a', lineWidth: 0.5 },
  { kind: 'grassland',      id: 'grassland',      fillColor: '#cdebb0', fillOpacity: 0.18, lineColor: '#a8c97f', lineWidth: 0.5 },
  { kind: 'protected_area', id: 'protected-area', fillColor: '#4a9d5f', fillOpacity: 0.2,  lineColor: '#2f7d47', lineWidth: 1 },
  { kind: 'wetland',        id: 'wetland',        fillColor: 'rgba(0,0,0,0)', fillOpacity: 0.4, lineColor: '#4a8fa3', lineWidth: 0.5 }
];
LANDCOVER_KINDS.forEach(function (lc) {
  window.SubwayBuilderAPI.map.registerLayer({
    id: '<CITY_CODE_LOWER>' + lc.id, type: 'fill', source: '<CITY_CODE_LOWER>landuse-source', 'source-layer': 'landuse',
    layout: { visibility: 'none' }, filter: ['==', ['get', 'kind'], lc.kind],
    paint: { 'fill-color': lc.fillColor, 'fill-opacity': lc.fillOpacity }
  });
  window.SubwayBuilderAPI.map.registerLayer({
    id: '<CITY_CODE_LOWER>' + lc.id + '-outline', type: 'line', source: '<CITY_CODE_LOWER>landuse-source', 'source-layer': 'landuse',
    layout: { visibility: 'none' }, filter: ['==', ['get', 'kind'], lc.kind],
    paint: { 'line-color': lc.lineColor, 'line-width': lc.lineWidth }
  });
});

window.SubwayBuilderAPI.map.registerLayer({
  id: '<CITY_CODE_LOWER>landcover-labels', type: 'symbol', source: '<CITY_CODE_LOWER>landuse-source', 'source-layer': 'landuse',
  filter: ['in', ['get', 'kind'], ['literal', ['protected_area', 'forest', 'wetland']]],
  minzoom: 10,
  layout: {
    visibility: 'none',
    'text-field': ['coalesce', ['get', 'name'], ''],
    'text-size': 13, 'text-font': ['Noto Sans Medium'], 'text-max-width': 8
  },
  paint: { 'text-color': '#1a1a1a', 'text-halo-color': '#ffffff', 'text-halo-width': 1.8 }
});

/*--------------- Intermittent Water ---------------*/
window.SubwayBuilderAPI.map.registerSource('<CITY_CODE_LOWER>water-intermittent-source', {
  type: 'vector',
  tiles: ['http://127.0.0.1:8080/<CITY_CODE>/{z}/{x}/{y}.mvt'],
  maxzoom: 15
});
window.SubwayBuilderAPI.map.registerLayer({
  id: '<CITY_CODE_LOWER>water-intermittent', type: 'fill', source: '<CITY_CODE_LOWER>water-intermittent-source', 'source-layer': 'water_intermittent',
  paint: { 'fill-color': '#cbe4f7', 'fill-opacity': 0.75 }
});
window.SubwayBuilderAPI.map.registerLayer({
  id: '<CITY_CODE_LOWER>water-intermittent-outline', type: 'line', source: '<CITY_CODE_LOWER>water-intermittent-source', 'source-layer': 'water_intermittent',
  paint: { 'line-color': '#4a90d9', 'line-width': 1, 'line-dasharray': [2, 2] }
});
(function () {
  const api = window.SubwayBuilderAPI;
  let waterRetryTimers = [];
  let waterStyleTimer = null;
  let waterStyleListenerMap = null;

  function teardownWaterPattern() {
    waterRetryTimers.forEach(function (id) { clearTimeout(id); });
    waterRetryTimers = [];
    if (waterStyleTimer) { clearTimeout(waterStyleTimer); waterStyleTimer = null; }
    if (waterStyleListenerMap) { waterStyleListenerMap.off('styledata', onWaterStyleData); waterStyleListenerMap = null; }
  }

  function wireWaterPattern(map, attemptsLeft) {
    if (!map.getLayer('<CITY_CODE_LOWER>water-intermittent')) {
      if (attemptsLeft > 0) {
        const id = setTimeout(function () { wireWaterPattern(map, attemptsLeft - 1); }, 100);
        waterRetryTimers.push(id);
      }
      return;
    }
    map.setPaintProperty('<CITY_CODE_LOWER>water-intermittent', 'fill-pattern', '<CITY_CODE_LOWER>water-intermittent-dots');
    map.setPaintProperty('<CITY_CODE_LOWER>water-intermittent', 'fill-opacity', 0.75);
  }

  function onWaterStyleData() {
    clearTimeout(waterStyleTimer);
    waterStyleTimer = setTimeout(function () {
      const map = api.utils.getMap();
      if (map) wireWaterPattern(map, 0);
    }, 150);
  }

  api.hooks.onMapReady(function () {
    teardownWaterPattern();

    const map = api.utils.getMap();
    if (!map) return;
    const size = 32;
    const canvas = document.createElement('canvas');
    canvas.width = size; canvas.height = size;
    const ctx = canvas.getContext('2d');
    ctx.fillStyle = '#E3F2FB';
    ctx.fillRect(0, 0, size, size);
    ctx.fillStyle = '#4a90d9';
    [[6,8,2],[22,5,2],[14,16,2],[26,20,2],[4,24,2],[18,27,2]].forEach(function (d) {
      ctx.beginPath();
      ctx.arc(d[0], d[1], d[2], 0, Math.PI * 2);
      ctx.fill();
    });
    const imgData = ctx.getImageData(0, 0, size, size);
    if (map.hasImage('<CITY_CODE_LOWER>water-intermittent-dots')) map.removeImage('<CITY_CODE_LOWER>water-intermittent-dots');
    map.addImage('<CITY_CODE_LOWER>water-intermittent-dots', imgData);
    wireWaterPattern(map, 50);

    waterStyleListenerMap = map;
    map.on('styledata', onWaterStyleData);
  });

  api.hooks.onGameEnd(teardownWaterPattern);
})();

/*-------------------- Land Use --------------------*/
var LANDUSE_KINDS = [
  { kind: 'agricultural', id: 'agricultural', fillColor: '#eef0d5', fillOpacity: 0.18, lineColor: '#b5963e', lineWidth: 0.5 },
  { kind: 'orchard',      id: 'orchard',      fillColor: '#aedfa3', fillOpacity: 0.45, lineColor: '#a07c2e', lineWidth: 0.5 },
  { kind: 'quarry',       id: 'quarry',       fillColor: '#c5c3c3', fillOpacity: 0.5,  lineColor: '#8c6b3e', lineWidth: 0.5 },
  { kind: 'recreation',   id: 'recreation',   fillColor: '#5fad7a', fillOpacity: 0.18, lineColor: '#3a7d55', lineWidth: 0.5 },
  { kind: 'beach',        id: 'beach',        fillColor: '#fff1ba', fillOpacity: 0.55, lineColor: '#c9b880', lineWidth: 0.5 }
];
LANDUSE_KINDS.forEach(function (lu) {
  window.SubwayBuilderAPI.map.registerLayer({
    id: '<CITY_CODE_LOWER>' + lu.id, type: 'fill', source: '<CITY_CODE_LOWER>landuse-source', 'source-layer': 'landuse',
    layout: { visibility: 'none' }, filter: ['==', ['get', 'kind'], lu.kind],
    paint: { 'fill-color': lu.fillColor, 'fill-opacity': lu.fillOpacity }
  });
  window.SubwayBuilderAPI.map.registerLayer({
    id: '<CITY_CODE_LOWER>' + lu.id + '-outline', type: 'line', source: '<CITY_CODE_LOWER>landuse-source', 'source-layer': 'landuse',
    layout: { visibility: 'none' }, filter: ['==', ['get', 'kind'], lu.kind],
    paint: { 'line-color': lu.lineColor, 'line-width': lu.lineWidth }
  });
});


window.SubwayBuilderAPI.map.registerLayer({
  id: '<CITY_CODE_LOWER>landuse-labels', type: 'symbol', source: '<CITY_CODE_LOWER>landuse-source', 'source-layer': 'landuse',
  filter: ['in', ['get', 'kind'], ['literal', ['recreation', 'quarry', 'beach']]],
  minzoom: 10,
  layout: {
    visibility: 'none',
    'text-field': ['coalesce', ['get', 'name'], ''],
    'text-size': 13, 'text-font': ['Noto Sans Medium'], 'text-max-width': 8
  },
  paint: { 'text-color': '#1a1a1a', 'text-halo-color': '#ffffff', 'text-halo-width': 1.8 }
});

/*----------------- Railway Tracks -----------------*/
window.SubwayBuilderAPI.map.registerSource('<CITY_CODE_LOWER>railway-lines-source', {
  type: 'vector',
  tiles: ['http://127.0.0.1:8080/<CITY_CODE>/{z}/{x}/{y}.mvt'],
  maxzoom: 15
});
window.SubwayBuilderAPI.map.registerLayer({
  id: '<CITY_CODE_LOWER>railway-lines-rail', type: 'line', source: '<CITY_CODE_LOWER>railway-lines-source', 'source-layer': 'railway_lines',
  filter: ['in', ['get', 'kind'], ['literal', ['rail', 'narrow_gauge']]],
  paint: { 'line-color': '#1a1a1a', 'line-width': 1 }
});
window.SubwayBuilderAPI.map.registerLayer({
  id: '<CITY_CODE_LOWER>railway-lines-light-rail', type: 'line', source: '<CITY_CODE_LOWER>railway-lines-source', 'source-layer': 'railway_lines',
  filter: ['==', ['get', 'kind'], 'light_rail'],
  paint: { 'line-color': '#2f9e44', 'line-width': 1 }
});
window.SubwayBuilderAPI.map.registerLayer({
  id: '<CITY_CODE_LOWER>railway-lines-subway', type: 'line', source: '<CITY_CODE_LOWER>railway-lines-source', 'source-layer': 'railway_lines',
  filter: ['==', ['get', 'kind'], 'subway'],
  paint: { 'line-color': '#e8590c', 'line-width': 1 }
});
window.SubwayBuilderAPI.map.registerLayer({
  id: '<CITY_CODE_LOWER>railway-lines-labels', type: 'symbol', source: '<CITY_CODE_LOWER>railway-lines-source', 'source-layer': 'railway_lines',
  filter: ['in', ['get', 'kind'], ['literal', ['rail', 'narrow_gauge', 'light_rail', 'subway']]],
  layout: {
    'symbol-placement': 'line', 'text-field': ['coalesce', ['get', 'name'], ''],
    'text-size': 10, 'text-font': ['Noto Sans Medium'], 'symbol-spacing': 400, 'text-max-angle': 30
  },
  paint: { 'text-color': '#1a1a1a', 'text-halo-color': '#ffffff', 'text-halo-width': 1 }
});

/*----------- Railway Buildings & Lands ------------*/
window.SubwayBuilderAPI.map.registerLayer({
  id: '<CITY_CODE_LOWER>railway', type: 'fill', source: '<CITY_CODE_LOWER>landuse-source', 'source-layer': 'landuse',
  filter: ['==', ['get', 'kind'], 'railway'],
  paint: { 'fill-color': '#8a7a6a', 'fill-opacity': 0.25 }
});
window.SubwayBuilderAPI.map.registerLayer({
  id: '<CITY_CODE_LOWER>railway-outline', type: 'line', source: '<CITY_CODE_LOWER>landuse-source', 'source-layer': 'landuse',
  filter: ['==', ['get', 'kind'], 'railway'],
  paint: { 'line-color': '#5c4f42', 'line-width': 0.5 }
});

// Dark mode
/*------- Railway Line Dark-Mode Color Swap --------*/
(function () {
  const api = window.SubwayBuilderAPI;

  function isRailDarkMode() {
    if (document.documentElement.classList.contains('dark')) return true;
    if (document.documentElement.classList.contains('light')) return false;
    return !!(window.matchMedia && window.matchMedia('(prefers-color-scheme: dark)').matches);
  }

  const RAIL_LINE_COLOR = { light: '#1a1a1a', dark: '#d9d9d9' };

  let railLastState = null;
  function applyRailColor() {
    const map = api.utils.getMap();
    if (!map || !map.getLayer('<CITY_CODE_LOWER>railway-lines-rail') || !map.getLayer('<CITY_CODE_LOWER>railway-lines-labels')) return;
    const dark = isRailDarkMode();
    if (dark === railLastState) return;
    railLastState = dark;
    const color = dark ? RAIL_LINE_COLOR.dark : RAIL_LINE_COLOR.light;
    map.setPaintProperty('<CITY_CODE_LOWER>railway-lines-rail', 'line-color', color);
    map.setPaintProperty('<CITY_CODE_LOWER>railway-lines-labels', 'text-color', color);
  }

  function reapplyRailColor() { railLastState = null; applyRailColor(); }

  let railMq = null, railObserver = null, railPollId = null, railStyleTimer = null, railStyleListenerMap = null;

  function teardownRailColor() {
    if (railMq) { railMq.removeEventListener('change', applyRailColor); railMq = null; }
    if (railObserver) { railObserver.disconnect(); railObserver = null; }
    if (railPollId) { clearInterval(railPollId); railPollId = null; }
    if (railStyleTimer) { clearTimeout(railStyleTimer); railStyleTimer = null; }
    if (railStyleListenerMap) { railStyleListenerMap.off('styledata', onRailStyleData); railStyleListenerMap = null; }
  }

  function onRailStyleData() {
    clearTimeout(railStyleTimer);
    railStyleTimer = setTimeout(reapplyRailColor, 150);
  }

  api.hooks.onMapReady(function () {
    teardownRailColor();
    const map = api.utils.getMap();
    railLastState = null;
    applyRailColor();
    if (window.matchMedia) {
      railMq = window.matchMedia('(prefers-color-scheme: dark)');
      railMq.addEventListener('change', applyRailColor);
    }
    railObserver = new MutationObserver(applyRailColor);
    railObserver.observe(document.documentElement, { attributes: true, attributeFilter: ['data-theme', 'class', 'style'] });
    railObserver.observe(document.body, { attributes: true, attributeFilter: ['data-theme', 'class', 'style'] });
    railPollId = setInterval(applyRailColor, 1000);
    if (map) {
      railStyleListenerMap = map;
      map.on('styledata', onRailStyleData);
    }
  });

  api.hooks.onGameEnd(teardownRailColor);
})();

/*------------------- Utilities --------------------*/
window.SubwayBuilderAPI.map.registerSource('<CITY_CODE_LOWER>utilities-source', {
  type: 'vector',
  tiles: ['http://127.0.0.1:8080/<CITY_CODE>/{z}/{x}/{y}.mvt'],
  maxzoom: 15
});

/*------------------ Power Plants ------------------*/
window.SubwayBuilderAPI.map.registerLayer({
  id: '<CITY_CODE_LOWER>power-plants-fill', type: 'fill', source: '<CITY_CODE_LOWER>utilities-source', 'source-layer': 'power_plants',
  paint: { 'fill-color': '#d9822b', 'fill-opacity': 0.45 }
});
window.SubwayBuilderAPI.map.registerLayer({
  id: '<CITY_CODE_LOWER>power-plants-outline', type: 'line', source: '<CITY_CODE_LOWER>utilities-source', 'source-layer': 'power_plants',
  paint: { 'line-color': '#a85f1a', 'line-width': 1 }
});

window.SubwayBuilderAPI.map.registerLayer({
  id: '<CITY_CODE_LOWER>power-plants-icons', type: 'symbol', source: '<CITY_CODE_LOWER>utilities-source', 'source-layer': 'power_plants',
  layout: {}
});

window.SubwayBuilderAPI.map.registerLayer({
  id: '<CITY_CODE_LOWER>power-plants-labels', type: 'symbol', source: '<CITY_CODE_LOWER>utilities-source', 'source-layer': 'power_plants',
  minzoom: 12,
  layout: {
    'text-field': ['coalesce', ['get', 'name'], ''],
    'text-size': 11, 'text-font': ['Noto Sans Medium'], 'text-offset': [0, 1.2], 'text-anchor': 'top', 'text-max-width': 8
  },
  paint: { 'text-color': '#1a1a1a', 'text-halo-color': '#ffffff', 'text-halo-width': 1.5 }
});

/*---------------- Power Generators ----------------*/
window.SubwayBuilderAPI.map.registerLayer({
  id: '<CITY_CODE_LOWER>power-generators', type: 'symbol', source: '<CITY_CODE_LOWER>utilities-source', 'source-layer': 'power_generators',
  minzoom: 12,
  layout: {}
});

window.SubwayBuilderAPI.map.registerLayer({
  id: '<CITY_CODE_LOWER>power-generators-labels', type: 'symbol', source: '<CITY_CODE_LOWER>utilities-source', 'source-layer': 'power_generators',
  minzoom: 12,
  layout: {
    'text-field': ['coalesce', ['get', 'name'], ''],
    'text-size': 10, 'text-font': ['Noto Sans Medium'], 'text-offset': [0, 1.1], 'text-anchor': 'top', 'text-max-width': 8,
    'text-optional': true
  },
  paint: { 'text-color': '#1a1a1a', 'text-halo-color': '#ffffff', 'text-halo-width': 1.5 }
});

/*---------------- Railway Stations ----------------*/
window.SubwayBuilderAPI.map.registerLayer({
  id: '<CITY_CODE_LOWER>stations', type: 'symbol', source: '<CITY_CODE_LOWER>utilities-source', 'source-layer': 'stations',
});

window.SubwayBuilderAPI.map.registerLayer({
  id: '<CITY_CODE_LOWER>stations-labels', type: 'symbol', source: '<CITY_CODE_LOWER>utilities-source', 'source-layer': 'stations',
  layout: {
    'text-field': ['coalesce', ['get', 'name'], ''],
    'text-size': 11, 'text-font': ['Noto Sans Medium'], 'text-offset': [0, 1.2], 'text-anchor': 'top', 'text-max-width': 8
  },
  paint: { 'text-color': '#1a1a1a', 'text-halo-color': '#ffffff', 'text-halo-width': 1.5 }
});

/*---------------------- Icons ---------------------*/
(function () {
  const api = window.SubwayBuilderAPI;

  let iconRetryTimers = [];
  let iconStyleTimer = null;
  let iconStyleListenerMap = null;

  function teardownIcons() {
    iconRetryTimers.forEach(function (id) { clearTimeout(id); });
    iconRetryTimers = [];
    if (iconStyleTimer) { clearTimeout(iconStyleTimer); iconStyleTimer = null; }
    if (iconStyleListenerMap) { iconStyleListenerMap.off('styledata', onIconStyleData); iconStyleListenerMap = null; }
  }

  function onIconStyleData() {
    clearTimeout(iconStyleTimer);
    iconStyleTimer = setTimeout(function () {
      if (window.__<CITY_CODE_LOWER>WireIcons) window.__<CITY_CODE_LOWER>WireIcons(0);
    }, 150);
  }

  api.hooks.onMapReady(function () {
    teardownIcons();

    const map = api.utils.getMap();
    if (!map) return;

    window.__<CITY_CODE_LOWER>Icons = window.__<CITY_CODE_LOWER>Icons || {};

    const RENEWABLE_RING = '#2f9e44';
    const NONRENEWABLE_RING = '#495057';
    const UNKNOWN_RING = '#adb5bd';
    const STATION_RING = '#1d6fb8';

    function makeIcon(key, ringColor, drawFn) {
      const imageId = '<CITY_CODE_LOWER>icon-' + key;
      const size = 24;
      const canvas = document.createElement('canvas');
      canvas.width = size;
      canvas.height = size;
      const ctx = canvas.getContext('2d');

      ctx.beginPath();
      ctx.arc(size / 2, size / 2, size / 2 - 1, 0, Math.PI * 2);
      ctx.fillStyle = '#ffffff';
      ctx.fill();
      ctx.lineWidth = 2;
      ctx.strokeStyle = ringColor;
      ctx.stroke();

      drawFn(ctx, size);

      window.__<CITY_CODE_LOWER>Icons[key] = canvas.toDataURL();
      const imgData = ctx.getImageData(0, 0, size, size);
      if (map.hasImage(imageId)) map.removeImage(imageId);
      map.addImage(imageId, imgData);
    }

    function glyph(ctx, drawStrokes) {
      ctx.strokeStyle = '#1a1a1a';
      ctx.fillStyle = '#1a1a1a';
      ctx.lineWidth = 1.4;
      drawStrokes();
    }

    // Wind - turbine
    makeIcon('wind', RENEWABLE_RING, function (ctx, s) {
      const cx = s / 2, cy = s / 2;
      ctx.strokeStyle = '#1a1a1a';
      ctx.lineWidth = 1.6;
      ctx.beginPath();
      ctx.moveTo(cx, cy + 7);
      ctx.lineTo(cx, cy - 7);
      ctx.stroke();
      [0, 120, 240].forEach(function (deg) {
        const rad = deg * Math.PI / 180;
        const x2 = cx + 6 * Math.sin(rad);
        const y2 = cy - 7 - 6 * Math.cos(rad);
        ctx.beginPath();
        ctx.moveTo(cx, cy - 7);
        ctx.lineTo(x2, y2);
        ctx.lineWidth = 2;
        ctx.strokeStyle = '#1a1a1a';
        ctx.stroke();
      });
    });

    // Solar - sun
    makeIcon('solar', RENEWABLE_RING, function (ctx, s) {
      const cx = s / 2, cy = s / 2;
      ctx.fillStyle = '#e8a800';
      ctx.beginPath();
      ctx.arc(cx, cy, 4, 0, Math.PI * 2);
      ctx.fill();
      ctx.strokeStyle = '#e8a800';
      ctx.lineWidth = 1.4;
      for (let i = 0; i < 8; i++) {
        const rad = i * Math.PI / 4;
        const x1 = cx + 6 * Math.cos(rad), y1 = cy + 6 * Math.sin(rad);
        const x2 = cx + 9 * Math.cos(rad), y2 = cy + 9 * Math.sin(rad);
        ctx.beginPath();
        ctx.moveTo(x1, y1);
        ctx.lineTo(x2, y2);
        ctx.stroke();
      }
    });

    // Hydro
    makeIcon('hydro', RENEWABLE_RING, function (ctx, s) {
      const cy = s / 2;
      glyph(ctx, function () {
        [cy - 3, cy + 1, cy + 5].forEach(function (y) {
          ctx.beginPath();
          ctx.moveTo(5, y);
          ctx.quadraticCurveTo(8, y - 3, 12, y);
          ctx.quadraticCurveTo(16, y + 3, 19, y);
          ctx.stroke();
        });
      });
    });

    // Biomass - leaf
    makeIcon('biomass', RENEWABLE_RING, function (ctx, s) {
      const cx = s / 2, cy = s / 2;
      ctx.fillStyle = '#2f7d32';
      ctx.beginPath();
      ctx.moveTo(cx, cy - 8);
      ctx.quadraticCurveTo(cx + 8, cy - 6, cx, cy + 8);
      ctx.quadraticCurveTo(cx - 8, cy - 6, cx, cy - 8);
      ctx.fill();
      ctx.strokeStyle = '#1a4a1c';
      ctx.lineWidth = 1;
      ctx.beginPath();
      ctx.moveTo(cx, cy - 6);
      ctx.lineTo(cx, cy + 7);
      ctx.stroke();
    });

    // Geothermal - steam vents
    makeIcon('geothermal', RENEWABLE_RING, function (ctx, s) {
      const cx = s / 2, cy = s / 2;
      ctx.strokeStyle = '#c05621';
      ctx.lineWidth = 1.4;
      ctx.beginPath();
      ctx.moveTo(cx - 8, cy + 4);
      ctx.lineTo(cx + 8, cy + 4);
      ctx.stroke();
      [-4, 0, 4].forEach(function (dx) {
        ctx.beginPath();
        ctx.moveTo(cx + dx, cy + 3);
        ctx.quadraticCurveTo(cx + dx - 2, cy - 2, cx + dx, cy - 6);
        ctx.stroke();
      });
    });

    // Coal - dark lump
    makeIcon('coal', NONRENEWABLE_RING, function (ctx, s) {
      const cx = s / 2, cy = s / 2;
      ctx.fillStyle = '#212529';
      ctx.beginPath();
      ctx.moveTo(cx - 7, cy + 5);
      ctx.lineTo(cx - 5, cy - 4);
      ctx.lineTo(cx, cy - 7);
      ctx.lineTo(cx + 6, cy - 3);
      ctx.lineTo(cx + 7, cy + 5);
      ctx.closePath();
      ctx.fill();
    });

    // Gas - flame
    makeIcon('gas', NONRENEWABLE_RING, function (ctx, s) {
      const cx = s / 2, cy = s / 2;
      ctx.fillStyle = '#e8590c';
      ctx.beginPath();
      ctx.moveTo(cx, cy - 8);
      ctx.quadraticCurveTo(cx + 6, cy - 2, cx + 3, cy + 3);
      ctx.quadraticCurveTo(cx + 5, cy + 1, cx + 4, cy + 7);
      ctx.quadraticCurveTo(cx, cy + 9, cx - 4, cy + 7);
      ctx.quadraticCurveTo(cx - 5, cy + 1, cx - 3, cy + 3);
      ctx.quadraticCurveTo(cx - 6, cy - 2, cx, cy - 8);
      ctx.fill();
    });

    // Oil - barrel
    makeIcon('oil', NONRENEWABLE_RING, function (ctx, s) {
      const cx = s / 2, cy = s / 2;
      ctx.fillStyle = '#343a40';
      ctx.fillRect(cx - 5, cy - 7, 10, 14);
      ctx.strokeStyle = '#000000';
      ctx.lineWidth = 1;
      ctx.beginPath();
      ctx.moveTo(cx - 5, cy - 2);
      ctx.lineTo(cx + 5, cy - 2);
      ctx.moveTo(cx - 5, cy + 3);
      ctx.lineTo(cx + 5, cy + 3);
      ctx.stroke();
    });

    // Nuclear - radiation trefoil
    makeIcon('nuclear', NONRENEWABLE_RING, function (ctx, s) {
      const cx = s / 2, cy = s / 2;
      ctx.fillStyle = '#212529';
      [0, 120, 240].forEach(function (deg) {
        const rad = deg * Math.PI / 180;
        ctx.beginPath();
        ctx.moveTo(cx, cy);
        ctx.arc(cx, cy, 7, rad - 0.45, rad + 0.45);
        ctx.closePath();
        ctx.fill();
      });
      ctx.fillStyle = '#fcc419';
      ctx.beginPath();
      ctx.arc(cx, cy, 1.8, 0, Math.PI * 2);
      ctx.fill();
    });

    // Waste - trash bin
    makeIcon('waste', NONRENEWABLE_RING, function (ctx, s) {
      const cx = s / 2, cy = s / 2;
      ctx.strokeStyle = '#495057';
      ctx.fillStyle = '#868e96';
      ctx.lineWidth = 1.2;
      ctx.fillRect(cx - 5, cy - 5, 10, 12);
      ctx.strokeRect(cx - 5, cy - 5, 10, 12);
      ctx.beginPath();
      ctx.moveTo(cx - 6, cy - 5);
      ctx.lineTo(cx + 6, cy - 5);
      ctx.stroke();
      ctx.beginPath();
      ctx.moveTo(cx - 2, cy - 7);
      ctx.lineTo(cx + 2, cy - 7);
      ctx.stroke();
    });

    // Unknown source - question mark
    makeIcon('unknown', UNKNOWN_RING, function (ctx, s) {
      const cx = s / 2, cy = s / 2;
      ctx.fillStyle = '#495057';
      ctx.font = 'bold 11px sans-serif';
      ctx.textAlign = 'center';
      ctx.textBaseline = 'middle';
      ctx.fillText('?', cx, cy + 1);
    });

    // Station - train
    makeIcon('station', STATION_RING, function (ctx, s) {
      const cx = s / 2, cy = s / 2;
      ctx.fillStyle = '#1d6fb8';
      ctx.beginPath();
      ctx.moveTo(cx - 5, cy - 5);
      ctx.lineTo(cx + 5, cy - 5);
      ctx.arc(cx + 5, cy - 1, 4, -Math.PI / 2, 0);
      ctx.lineTo(cx + 9, cy + 5);
      ctx.lineTo(cx - 9, cy + 5);
      ctx.lineTo(cx - 9, cy - 1);
      ctx.arc(cx - 5, cy - 1, 4, Math.PI, Math.PI * 1.5);
      ctx.closePath();
      ctx.fill();
      ctx.fillStyle = '#ffffff';
      ctx.fillRect(cx - 6, cy - 3, 4, 3);
      ctx.fillRect(cx + 2, cy - 3, 4, 3);
      ctx.fillStyle = '#1d6fb8';
      ctx.beginPath();
      ctx.arc(cx - 5, cy + 6, 1.3, 0, Math.PI * 2);
      ctx.arc(cx + 5, cy + 6, 1.3, 0, Math.PI * 2);
      ctx.fill();
    });

    const <CITY_CODE_LOWER>_ICON_MATCH = [
      'match', ['coalesce', ['get', 'source'], 'unknown'],
      'wind', '<CITY_CODE_LOWER>icon-wind',
      'solar', '<CITY_CODE_LOWER>icon-solar',
      'hydro', '<CITY_CODE_LOWER>icon-hydro',
      'biomass', '<CITY_CODE_LOWER>icon-biomass',
      'geothermal', '<CITY_CODE_LOWER>icon-geothermal',
      'coal', '<CITY_CODE_LOWER>icon-coal',
      'gas', '<CITY_CODE_LOWER>icon-gas',
      'oil', '<CITY_CODE_LOWER>icon-oil',
      'nuclear', '<CITY_CODE_LOWER>icon-nuclear',
      'waste', '<CITY_CODE_LOWER>icon-waste',
      '<CITY_CODE_LOWER>icon-unknown'
    ];
    const <CITY_CODE_LOWER>_ICON_SIZE = ['interpolate', ['linear'], ['zoom'], 10, 0.35, 15, 0.6];

    const ICON_LAYER_IDS = ['<CITY_CODE_LOWER>power-generators', '<CITY_CODE_LOWER>power-plants-icons', '<CITY_CODE_LOWER>stations'];
    function wireIcons(attemptsLeft) {
      const ready = ICON_LAYER_IDS.every(function (id) { return !!map.getLayer(id); });
      if (!ready) {
        if (attemptsLeft > 0) {
          const id = setTimeout(function () { wireIcons(attemptsLeft - 1); }, 100);
          iconRetryTimers.push(id);
        }
        return;
      }
      map.setLayoutProperty('<CITY_CODE_LOWER>power-generators', 'icon-image', <CITY_CODE_LOWER>_ICON_MATCH);
      map.setLayoutProperty('<CITY_CODE_LOWER>power-generators', 'icon-size', <CITY_CODE_LOWER>_ICON_SIZE);
      map.setLayoutProperty('<CITY_CODE_LOWER>power-generators', 'icon-allow-overlap', true);

      map.setLayoutProperty('<CITY_CODE_LOWER>power-plants-icons', 'icon-image', <CITY_CODE_LOWER>_ICON_MATCH);
      map.setLayoutProperty('<CITY_CODE_LOWER>power-plants-icons', 'icon-size', <CITY_CODE_LOWER>_ICON_SIZE);
      map.setLayoutProperty('<CITY_CODE_LOWER>power-plants-icons', 'icon-allow-overlap', true);

      map.setLayoutProperty('<CITY_CODE_LOWER>stations', 'icon-image', '<CITY_CODE_LOWER>icon-station');
      map.setLayoutProperty('<CITY_CODE_LOWER>stations', 'icon-size', <CITY_CODE_LOWER>_ICON_SIZE);
      map.setLayoutProperty('<CITY_CODE_LOWER>stations', 'icon-allow-overlap', true);
    }
    window.__<CITY_CODE_LOWER>WireIcons = wireIcons;
    wireIcons(50);

    iconStyleListenerMap = map;
    map.on('styledata', onIconStyleData);
  });

  api.hooks.onGameEnd(teardownIcons);
})();
```

3. Append the above into `index.js`:

```batch
type "%USERPROFILE%\Desktop\SB_mod_project\02_intermediate\lclu-1_index-js.js" >> "%USERPROFILE%\Desktop\SB_mod_project\03_mod\<CITY_CODE_LOWER>-mod\index.js"
```

4. Run the below in cmd:

```batch
notepad "%USERPROFILE%\Desktop\SB_mod_project\02_intermediate\lclu-2_index-js.ps1"
```

5. Paste into the notepad:

```powershell
param(
    [Parameter(Mandatory=$true)]
    [string]$IndexPath
)

if (-not (Test-Path $IndexPath)) {
    Write-Host "ABORTED: file not found at $IndexPath"
    exit 1
}

$content = Get-Content -Raw -LiteralPath $IndexPath

function Count-Occurrences($haystack, $needle) {
    if ($needle.Length -eq 0) { return 0 }
    $count = 0
    $idx = 0
    while ($true) {
        $found = $haystack.IndexOf($needle, $idx, [System.StringComparison]::Ordinal)
        if ($found -lt 0) { break }
        $count++
        $idx = $found + $needle.Length
    }
    return $count
}

function Apply-Edit($content, $oldStr, $newStr, $editName) {
    $count = Count-Occurrences $content $oldStr
    if ($count -ne 1) {
        Write-Host "ABORTED: $editName anchor not found exactly once (found $count). File doesn't match expected content."
        exit 1
    }
    return $content.Replace($oldStr, $newStr)
}

# Edit 1: Insert Railway + Utilities categories into Legend
$oldTerrainClose = @'
        { label: '3D Terrain (hides tracks/stations)', color: '#6a5acd', outline: '#4a3d9e', independentToggle: true, isTerrainToggle: true, defaultOn: false, dependsOnSiblings: true }
      ]
    }
  ];
'@

$newTerrainClose = @'
        { label: '3D Terrain (hides tracks/stations)', color: '#6a5acd', outline: '#4a3d9e', independentToggle: true, isTerrainToggle: true, defaultOn: false, dependsOnSiblings: true }
      ]
    },
    {
      category: 'Land Cover and Protected Lands',
      defaultOn: false,
      items: [
        { label: 'Forest', color: '#add19e', outline: '#2d6a35', layerIds: ['<CITY_CODE_LOWER>forest', '<CITY_CODE_LOWER>forest-outline', '<CITY_CODE_LOWER>landcover-labels'], patternKey: 'forest' },
        { label: 'Scrub', color: '#c8d7ab', outline: '#7daa5a', layerIds: ['<CITY_CODE_LOWER>scrub', '<CITY_CODE_LOWER>scrub-outline'], patternKey: 'scrub' },
        { label: 'Grassland / Meadow', color: '#cdebb0', outline: '#a8c97f', layerIds: ['<CITY_CODE_LOWER>grassland', '<CITY_CODE_LOWER>grassland-outline'] },
        { label: 'Protected Area / National Park / Nature Reserve', color: '#4a9d5f', outline: '#2f7d47', layerIds: ['<CITY_CODE_LOWER>protected-area', '<CITY_CODE_LOWER>protected-area-outline', '<CITY_CODE_LOWER>landcover-labels'] },
        { label: 'Wetland', color: 'rgba(0,0,0,0)', outline: '#4a8fa3', layerIds: ['<CITY_CODE_LOWER>wetland', '<CITY_CODE_LOWER>wetland-outline', '<CITY_CODE_LOWER>landcover-labels'], patternKey: 'wetland' },
        { label: 'Intermittent Water (always on)', color: '#cbe4f7', outline: '#4a90d9', layerIds: ['<CITY_CODE_LOWER>water-intermittent', '<CITY_CODE_LOWER>water-intermittent-outline'], alwaysOn: true  }
      ]
    },
    {
      category: 'Land Use',
      defaultOn: false,
      items: [
        { label: 'Agricultural', color: '#eef0d5', outline: '#b5963e', layerIds: ['<CITY_CODE_LOWER>agricultural', '<CITY_CODE_LOWER>agricultural-outline'] },
        { label: 'Orchard', color: '#aedfa3', outline: '#a07c2e', layerIds: ['<CITY_CODE_LOWER>orchard', '<CITY_CODE_LOWER>orchard-outline'], patternKey: 'orchard' },
        { label: 'Quarry', color: '#c5c3c3', outline: '#8c6b3e', layerIds: ['<CITY_CODE_LOWER>quarry', '<CITY_CODE_LOWER>quarry-outline', '<CITY_CODE_LOWER>landuse-labels'], patternKey: 'quarry' },
        { label: 'Park / Pitch / Recreation Ground / Zoo', color: '#5fad7a', outline: '#3a7d55', layerIds: ['<CITY_CODE_LOWER>recreation', '<CITY_CODE_LOWER>recreation-outline', '<CITY_CODE_LOWER>landuse-labels'] },
        { label: 'Beach', color: '#fff1ba', outline: '#c9b880', layerIds: ['<CITY_CODE_LOWER>beach', '<CITY_CODE_LOWER>beach-outline', '<CITY_CODE_LOWER>landuse-labels'], patternKey: 'beach' },
      ]
    },
    {
      category: 'Railway',
      defaultOpen: false,
      items: [
        { label: 'Railway Yard/Land', color: '#8a7a6a', outline: '#5c4f42', layerIds: ['<CITY_CODE_LOWER>railway', '<CITY_CODE_LOWER>railway-outline'] },
        { label: 'Railway Line', color: '#1a1a1a', outline: '#1a1a1a', layerIds: ['<CITY_CODE_LOWER>railway-lines-rail', '<CITY_CODE_LOWER>railway-lines-labels'] },
        { label: 'Light Rail Line', color: '#2f9e44', outline: '#2f9e44', layerIds: ['<CITY_CODE_LOWER>railway-lines-light-rail'] },
        { label: 'Subway Line', color: '#e8590c', outline: '#e8590c', layerIds: ['<CITY_CODE_LOWER>railway-lines-subway'] },
        { label: 'Station', iconKey: 'station', layerIds: ['<CITY_CODE_LOWER>stations', '<CITY_CODE_LOWER>stations-labels'] }
      ]
    },
    {
      category: 'Utilities',
      defaultOpen: false,
      items: [
        { label: 'Power Plant - Wind', iconKey: 'wind', layerIds: ['<CITY_CODE_LOWER>power-plants-fill', '<CITY_CODE_LOWER>power-plants-outline', '<CITY_CODE_LOWER>power-plants-icons', '<CITY_CODE_LOWER>power-plants-labels'] },
        { label: 'Power Plant - Solar', iconKey: 'solar', layerIds: ['<CITY_CODE_LOWER>power-plants-fill', '<CITY_CODE_LOWER>power-plants-outline', '<CITY_CODE_LOWER>power-plants-icons', '<CITY_CODE_LOWER>power-plants-labels'] },
        { label: 'Power Plant - Hydro', iconKey: 'hydro', layerIds: ['<CITY_CODE_LOWER>power-plants-fill', '<CITY_CODE_LOWER>power-plants-outline', '<CITY_CODE_LOWER>power-plants-icons', '<CITY_CODE_LOWER>power-plants-labels'] },
        { label: 'Power Plant - Biomass', iconKey: 'biomass', layerIds: ['<CITY_CODE_LOWER>power-plants-fill', '<CITY_CODE_LOWER>power-plants-outline', '<CITY_CODE_LOWER>power-plants-icons', '<CITY_CODE_LOWER>power-plants-labels'] },
        { label: 'Power Plant - Geothermal', iconKey: 'geothermal', layerIds: ['<CITY_CODE_LOWER>power-plants-fill', '<CITY_CODE_LOWER>power-plants-outline', '<CITY_CODE_LOWER>power-plants-icons', '<CITY_CODE_LOWER>power-plants-labels'] },
        { label: 'Power Plant - Coal', iconKey: 'coal', layerIds: ['<CITY_CODE_LOWER>power-plants-fill', '<CITY_CODE_LOWER>power-plants-outline', '<CITY_CODE_LOWER>power-plants-icons', '<CITY_CODE_LOWER>power-plants-labels'] },
        { label: 'Power Plant - Gas', iconKey: 'gas', layerIds: ['<CITY_CODE_LOWER>power-plants-fill', '<CITY_CODE_LOWER>power-plants-outline', '<CITY_CODE_LOWER>power-plants-icons', '<CITY_CODE_LOWER>power-plants-labels'] },
        { label: 'Power Plant - Oil', iconKey: 'oil', layerIds: ['<CITY_CODE_LOWER>power-plants-fill', '<CITY_CODE_LOWER>power-plants-outline', '<CITY_CODE_LOWER>power-plants-icons', '<CITY_CODE_LOWER>power-plants-labels'] },
        { label: 'Power Plant - Nuclear', iconKey: 'nuclear', layerIds: ['<CITY_CODE_LOWER>power-plants-fill', '<CITY_CODE_LOWER>power-plants-outline', '<CITY_CODE_LOWER>power-plants-icons', '<CITY_CODE_LOWER>power-plants-labels'] },
        { label: 'Power Plant - Waste', iconKey: 'waste', layerIds: ['<CITY_CODE_LOWER>power-plants-fill', '<CITY_CODE_LOWER>power-plants-outline', '<CITY_CODE_LOWER>power-plants-icons', '<CITY_CODE_LOWER>power-plants-labels'] },
        { label: 'Power Plant - Unknown Source', iconKey: 'unknown', layerIds: ['<CITY_CODE_LOWER>power-plants-fill', '<CITY_CODE_LOWER>power-plants-outline', '<CITY_CODE_LOWER>power-plants-icons', '<CITY_CODE_LOWER>power-plants-labels'] },
        { label: 'Power Generator - Wind', iconKey: 'wind', layerIds: ['<CITY_CODE_LOWER>power-generators', '<CITY_CODE_LOWER>power-generators-labels'] },
        { label: 'Power Generator - Solar', iconKey: 'solar', layerIds: ['<CITY_CODE_LOWER>power-generators', '<CITY_CODE_LOWER>power-generators-labels'] },
        { label: 'Power Generator - Hydro', iconKey: 'hydro', layerIds: ['<CITY_CODE_LOWER>power-generators', '<CITY_CODE_LOWER>power-generators-labels'] },
        { label: 'Power Generator - Biomass', iconKey: 'biomass', layerIds: ['<CITY_CODE_LOWER>power-generators', '<CITY_CODE_LOWER>power-generators-labels'] },
        { label: 'Power Generator - Geothermal', iconKey: 'geothermal', layerIds: ['<CITY_CODE_LOWER>power-generators', '<CITY_CODE_LOWER>power-generators-labels'] },
        { label: 'Power Generator - Coal', iconKey: 'coal', layerIds: ['<CITY_CODE_LOWER>power-generators', '<CITY_CODE_LOWER>power-generators-labels'] },
        { label: 'Power Generator - Gas', iconKey: 'gas', layerIds: ['<CITY_CODE_LOWER>power-generators', '<CITY_CODE_LOWER>power-generators-labels'] },
        { label: 'Power Generator - Oil', iconKey: 'oil', layerIds: ['<CITY_CODE_LOWER>power-generators', '<CITY_CODE_LOWER>power-generators-labels'] },
        { label: 'Power Generator - Nuclear', iconKey: 'nuclear', layerIds: ['<CITY_CODE_LOWER>power-generators', '<CITY_CODE_LOWER>power-generators-labels'] },
        { label: 'Power Generator - Waste', iconKey: 'waste', layerIds: ['<CITY_CODE_LOWER>power-generators', '<CITY_CODE_LOWER>power-generators-labels'] },
        { label: 'Power Generator - Unknown Source', iconKey: 'unknown', layerIds: ['<CITY_CODE_LOWER>power-generators', '<CITY_CODE_LOWER>power-generators-labels'] }
      ]
    }
  ];
'@

$content = Apply-Edit $content $oldTerrainClose $newTerrainClose "Terrain-close / insert Railway+Utilities legendData"

# Edit 2
$oldSwatch = @'
  function Swatch(props) {
    const item = props.item;
    const dark = props.dark;
    const isTransparent = item.color === 'transparent' || item.color === 'rgba(0,0,0,0)';
    return h('span', {
      style: Object.assign(
        {
          width: 14, height: 14, flexShrink: 0, borderRadius: 3,
          border: '1px solid ' + (item.outline || item.color)
        },
        isTransparent
          ? { background: dark ? 'rgba(255,255,255,0.08)' : undefined }
          : { background: item.color }
      )
    });
  }
'@

$newSwatch = @'
  function patternBg(color, patternKey) {
    var dataUrl = window.__<CITY_CODE_LOWER>Patterns && window.__<CITY_CODE_LOWER>Patterns[patternKey];
    if (dataUrl) {
      return { backgroundImage: 'url(' + dataUrl + ')', backgroundSize: '14px 14px', backgroundRepeat: 'repeat', backgroundColor: color };
    }
    return { background: color };
  }

  function Swatch(props) {
    const item = props.item;
    const dark = props.dark;
    if (item.iconKey) {
      var iconUrl = window.__<CITY_CODE_LOWER>Icons && window.__<CITY_CODE_LOWER>Icons[item.iconKey];
      return h('span', { style: { width: 14, height: 14, flexShrink: 0, display: 'inline-block' } },
        iconUrl ? h('img', { src: iconUrl, style: { width: '100%', height: '100%' } }) : null
      );
    }
    const isTransparent = item.color === 'transparent' || item.color === 'rgba(0,0,0,0)';
    return h('span', {
      style: Object.assign(
        {
          width: 14, height: 14, flexShrink: 0, borderRadius: 3,
          border: '1px solid ' + (item.outline || item.color)
        },
        isTransparent
          ? { background: dark ? 'rgba(255,255,255,0.08)' : undefined }
          : patternBg(item.color, item.patternKey)
      )
    });
  }
'@

$content = Apply-Edit $content $oldSwatch $newSwatch "Swatch"

# Edit 3
$oldLegendRow = @'
  function LegendRow(props) {
    const item = props.item;
    const checked = props.checked;
    const disabled = props.disabled;
    const dark = props.dark;

    return h(
      'div',
      { style: { display: 'flex', alignItems: 'center', gap: 6, padding: '2px 0', opacity: disabled ? 0.45 : 1 } },
      h(Swatch, { item: item, dark: dark }),
      h('span', { style: { fontSize: 12, flex: 1, color: dark ? '#e8e8e8' : '#1a1a1a' } }, item.label),
      h(api.utils.components.Switch, {
        checked: checked,
        disabled: disabled,
        onCheckedChange: props.onToggle,
        style: { marginLeft: 'auto', transform: 'scale(0.75)' }
      })
    );
  }
'@

$newLegendRow = @'
  function LegendRow(props) {
    const item = props.item;
    const checked = props.checked;
    const disabled = props.disabled;
    const dark = props.dark;
    const showToggle = !!item.independentToggle;

    return h(
      'div',
      { style: { display: 'flex', alignItems: 'center', gap: 6, padding: '2px 0', opacity: disabled ? 0.45 : 1 } },
      h(Swatch, { item: item, dark: dark }),
      h('span', { style: { fontSize: 12, flex: 1, color: dark ? '#e8e8e8' : '#1a1a1a' } }, item.label),
      showToggle ? h(api.utils.components.Switch, {
        checked: checked,
        disabled: disabled,
        onCheckedChange: props.onToggle,
        style: { marginLeft: 'auto', transform: 'scale(0.75)' }
      }) : null
    );
  }
'@

$content = Apply-Edit $content $oldLegendRow $newLegendRow "LegendRow"

# Edit 4: Category-level toggle
$oldCategoryGroup = @'
  const BOUNDARY_ROW_COLORS = {};

  function CategoryGroup(props) {
    const group = props.group;
    const dark = props.dark;

    const openState = React.useState(group.defaultOpen === false ? false : true);
    const open = openState[0];
    const setOpen = openState[1];

    const saved = window.__<CITY_CODE_LOWER>LegendToggleState[group.category];

    const initial = {};
    group.items.forEach(function (item) {
      initial[item.label] = item.defaultOn === false ? false : true;
    });
    if (saved && saved.checkedMap) {
      Object.assign(initial, saved.checkedMap);
    }
    const checkedState = React.useState(initial);
    const checkedMap = checkedState[0];
    const setCheckedMap = checkedState[1];

    const siblingsOnCount = group.items.filter(function (item) {
      return !item.isTerrainToggle && checkedMap[item.label];
    }).length;

    function applyToggle(item, val) {
      const map = api.utils.getMap();
      if (!map) return;

      if (item.isTerrainToggle) {
        if (val) {
          map.setTerrain({ source: '<CITY_CODE_LOWER>-hillshade-source', exaggeration: 1.0 });
        } else {
          map.setTerrain(null);
        }
        return;
      }

      if (item.label === 'Hypsometric Tinting' && item.layerIds) {
        const dark = document.documentElement.classList.contains('dark')
          ? true
          : document.documentElement.classList.contains('light')
          ? false
          : !!(window.matchMedia && window.matchMedia('(prefers-color-scheme: dark)').matches);
        item.layerIds.forEach(function (id) {
          const isDarkLayer = id.indexOf('-dark') !== -1;
          map.setLayoutProperty(id, 'visibility', (val && (isDarkLayer === dark)) ? 'visible' : 'none');
        });
        return;
      }

      if (item.layerIds) {
        item.layerIds.forEach(function (id) {
          map.setLayoutProperty(id, 'visibility', val ? 'visible' : 'none');
        });
      }
    }

    React.useEffect(function () {
      group.items.forEach(function (item) {
        if (item.independentToggle) applyToggle(item, checkedMap[item.label]);
      });
    }, []);

    function handleToggle(item, val) {
      const next = Object.assign({}, checkedMap);
      next[item.label] = val;

      if (!item.isTerrainToggle && !val) {
        const stillOnCount = group.items.filter(function (other) {
          return !other.isTerrainToggle && next[other.label];
        }).length;
        const terrainItem = group.items.filter(function (other) { return other.isTerrainToggle; })[0];
        if (terrainItem && stillOnCount === 0 && next[terrainItem.label]) {
          next[terrainItem.label] = false;
          applyToggle(terrainItem, false);
        }
      }

      setCheckedMap(next);
      window.__<CITY_CODE_LOWER>LegendToggleState[group.category] = Object.assign({}, window.__<CITY_CODE_LOWER>LegendToggleState[group.category], { checkedMap: next });
      applyToggle(item, val);
    }

    return h(
      'div',
      { style: { marginBottom: 8 } },
      h(
        'button',
        {
          onClick: function () { setOpen(!open); },
          style: {
            width: '100%', textAlign: 'left', background: 'transparent', border: 'none',
            color: dark ? '#f2f2f2' : '#1a1a1a', fontWeight: 600, fontSize: 12, padding: '4px 0',
            cursor: 'pointer', opacity: 0.9
          }
        },
        (open ? '▾ ' : '▸ ') + group.category
      ),
      open ? h('div', { style: { paddingLeft: 4 } }, group.items.map(function (item) {
        const isDisabled = !!(item.dependsOnSiblings && siblingsOnCount === 0);
        const rowItem = BOUNDARY_ROW_COLORS[item.label]
          ? Object.assign({}, item, {
              outline: dark ? BOUNDARY_ROW_COLORS[item.label].dark : item.outline,
              color: (item.color === 'transparent') ? item.color : (dark ? BOUNDARY_ROW_COLORS[item.label].dark : item.color)
            })
          : item;
        return h(LegendRow, {
          key: item.label,
          item: rowItem,
          checked: checkedMap[item.label],
          disabled: isDisabled,
          dark: dark,
          onToggle: function (val) { handleToggle(item, val); }
        });
      })) : null
    );
  }
'@

$newCategoryGroup = @'
  const BOUNDARY_ROW_COLORS = {};

  function CategoryGroup(props) {
    const group = props.group;
    const dark = props.dark;

    const openState = React.useState(group.defaultOpen === false ? false : true);
    const open = openState[0];
    const setOpen = openState[1];

    const saved = window.__<CITY_CODE_LOWER>LegendToggleState[group.category];

    const initial = {};
    group.items.forEach(function (item) {
      initial[item.label] = item.defaultOn === false ? false : true;
    });
    if (saved && saved.checkedMap) {
      Object.assign(initial, saved.checkedMap);
    }
    const checkedState = React.useState(initial);
    const checkedMap = checkedState[0];
    const setCheckedMap = checkedState[1];

    const toggleState = React.useState(
      saved && typeof saved.categoryOn === 'boolean' ? saved.categoryOn : (group.defaultOn === false ? false : true)
    );
    const categoryOn = toggleState[0];
    const setCategoryOn = toggleState[1];

    const toggleableLayerIds = [];
    group.items.forEach(function (item) {
      if (item.layerIds && !item.independentToggle && !item.isTerrainToggle && !item.alwaysOn) {
        toggleableLayerIds.push.apply(toggleableLayerIds, item.layerIds);
      }
    });

    React.useEffect(function () {
      const map = api.utils.getMap();
      if (!map) return;
      group.items.forEach(function (item) {
        if (item.independentToggle) applyToggle(item, checkedMap[item.label]);
      });
      if (toggleableLayerIds.length) {
        toggleableLayerIds.forEach(function (id) {
          map.setLayoutProperty(id, 'visibility', categoryOn ? 'visible' : 'none');
        });
      }
    }, []);

    function handleCategoryToggle(val) {
      setCategoryOn(val);
      window.__<CITY_CODE_LOWER>LegendToggleState[group.category] = Object.assign({}, window.__<CITY_CODE_LOWER>LegendToggleState[group.category], { categoryOn: val });
      const map = api.utils.getMap();
      if (map) {
        toggleableLayerIds.forEach(function (id) {
          map.setLayoutProperty(id, 'visibility', val ? 'visible' : 'none');
        });
      }
    }

    const siblingsOnCount = group.items.filter(function (item) {
      return !item.isTerrainToggle && checkedMap[item.label];
    }).length;

    function applyToggle(item, val) {
      const map = api.utils.getMap();
      if (!map) return;

      if (item.isTerrainToggle) {
        if (val) {
          map.setTerrain({ source: '<CITY_CODE_LOWER>-hillshade-source', exaggeration: 1.0 });
        } else {
          map.setTerrain(null);
        }
        return;
      }

      if (item.label === 'Hypsometric Tinting' && item.layerIds) {
        const dark = document.documentElement.classList.contains('dark')
          ? true
          : document.documentElement.classList.contains('light')
          ? false
          : !!(window.matchMedia && window.matchMedia('(prefers-color-scheme: dark)').matches);
        item.layerIds.forEach(function (id) {
          const isDarkLayer = id.indexOf('-dark') !== -1;
          map.setLayoutProperty(id, 'visibility', (val && (isDarkLayer === dark)) ? 'visible' : 'none');
        });
        return;
      }

      if (item.layerIds) {
        item.layerIds.forEach(function (id) {
          map.setLayoutProperty(id, 'visibility', val ? 'visible' : 'none');
        });
      }
    }

    function handleToggle(item, val) {
      const next = Object.assign({}, checkedMap);
      next[item.label] = val;

      if (!item.isTerrainToggle && !val) {
        const stillOnCount = group.items.filter(function (other) {
          return !other.isTerrainToggle && next[other.label];
        }).length;
        const terrainItem = group.items.filter(function (other) { return other.isTerrainToggle; })[0];
        if (terrainItem && stillOnCount === 0 && next[terrainItem.label]) {
          next[terrainItem.label] = false;
          applyToggle(terrainItem, false);
        }
      }

      setCheckedMap(next);
      window.__<CITY_CODE_LOWER>LegendToggleState[group.category] = Object.assign({}, window.__<CITY_CODE_LOWER>LegendToggleState[group.category], { checkedMap: next });
      applyToggle(item, val);
    }

    return h(
      'div',
      { style: { marginBottom: 8 } },
      h(
        'div',
        { style: { display: 'flex', alignItems: 'center', gap: 6 } },
        h(
          'button',
          {
            onClick: function () { setOpen(!open); },
            style: {
              flex: 1, textAlign: 'left', background: 'transparent', border: 'none',
              color: dark ? '#f2f2f2' : '#1a1a1a', fontWeight: 600, fontSize: 12, padding: '4px 0',
              cursor: 'pointer', opacity: 0.95
            }
          },
          (open ? '▾ ' : '▸ ') + group.category
        ),
        toggleableLayerIds.length > 0 ? h(api.utils.components.Switch, {
          checked: categoryOn,
          onCheckedChange: handleCategoryToggle,
          style: { transform: 'scale(0.75)' }
        }) : null
      ),
      open ? h('div', { style: { paddingLeft: 4 } }, group.items.map(function (item) {
        const isDisabled = !!(item.dependsOnSiblings && siblingsOnCount === 0);
        const rowItem = BOUNDARY_ROW_COLORS[item.label]
          ? Object.assign({}, item, {
              outline: dark ? BOUNDARY_ROW_COLORS[item.label].dark : item.outline,
              color: (item.color === 'transparent') ? item.color : (dark ? BOUNDARY_ROW_COLORS[item.label].dark : item.color)
            })
          : item;
        return h(LegendRow, {
          key: item.label,
          item: rowItem,
          checked: item.independentToggle ? checkedMap[item.label] : categoryOn,
          disabled: isDisabled,
          dark: dark,
          onToggle: function (val) { handleToggle(item, val); }
        });
      })) : null
    );
  }
'@

$content = Apply-Edit $content $oldCategoryGroup $newCategoryGroup "CategoryGroup"

Set-Content -LiteralPath $IndexPath -Value $content -NoNewline
Write-Host "SUCCESS: Railway/Utilities categories inserted, Station switched to icon swatch, Swatch/LegendRow/CategoryGroup updated to support category-level toggling."
```

6. Merge into `index.js`:

```batch
powershell -ExecutionPolicy Bypass -File "%USERPROFILE%\Desktop\SB_mod_project\02_intermediate\lclu-2_index-js.ps1" -IndexPath "%USERPROFILE%\Desktop\SB_mod_project\03_mod\<CITY_CODE_LOWER>-mod\index.js"
```

### 8f. Test in game

1. Move the updated mod files to the respective game folder:

```batch
xcopy "%USERPROFILE%\Desktop\SB_mod_project\03_mod" "%APPDATA%\metro-maker4\mods"  /E /I /D /Y
```

2. Run the server again (if it was closed previously) and check if all the new LCLU layers appear correctly.

----

## Stage 9: Scale Bar and Administrative boundaries (Optional)

*Est. time: 45 minutes*

### 9a. Scale Bar

1. Run below in cmd:

```batch
notepad "%USERPROFILE%\Desktop\SB_mod_project\02_intermediate\scalebar_index-js.js"
```

2. Paste into the notepad:

```javascript


/*----------------------------------------------------
--------------------- Scale Bar ----------------------
----------------------------------------------------*/
(function () {
  const api = window.SubwayBuilderAPI;
  const React = api.utils.React;
  const h = React.createElement;

  function readBackgroundLuminance(el) {
    if (!el) return null;
    const c = getComputedStyle(el).backgroundColor;
    const m = c && c.match(/rgba?\(([^)]+)\)/);
    if (!m) return null;
    const parts = m[1].split(',').map(function (s) { return parseFloat(s); });
    const a = parts.length > 3 ? parts[3] : 1;
    if (a === 0) return null;
    return 0.299 * parts[0] + 0.587 * parts[1] + 0.114 * parts[2];
  }

  function isScaleBarDarkMode() {
    const attr = document.documentElement.getAttribute('data-theme');
    if (attr === 'dark') return true;
    if (attr === 'light') return false;
    const bodyLum = readBackgroundLuminance(document.body);
    if (bodyLum !== null) return bodyLum < 128;
    const htmlLum = readBackgroundLuminance(document.documentElement);
    if (htmlLum !== null) return htmlLum < 128;
    return !!(window.matchMedia && window.matchMedia('(prefers-color-scheme: dark)').matches);
  }

  function getRoundNum(num) {
    const pow10 = Math.pow(10, (Math.floor(num) + '').length - 1);
    let d = num / pow10;
    d = d >= 10 ? 10 : d >= 5 ? 5 : d >= 3 ? 3 : d >= 2 ? 2 : 1;
    return pow10 * d;
  }

  function haversineMeters(a, b) {
    const R = 6371000;
    const toRad = function (deg) { return deg * Math.PI / 180; };
    const dLat = toRad(b[1] - a[1]);
    const dLng = toRad(b[0] - a[0]);
    const lat1 = toRad(a[1]);
    const lat2 = toRad(b[1]);
    const sinDLat = Math.sin(dLat / 2);
    const sinDLng = Math.sin(dLng / 2);
    const h2 = sinDLat * sinDLat + Math.cos(lat1) * Math.cos(lat2) * sinDLng * sinDLng;
    return 2 * R * Math.asin(Math.sqrt(h2));
  }

  const TARGET_PX = 120;

  function computeScale(map) {
    const container = map.getContainer();
    const w = container.clientWidth;
    const h2 = container.clientHeight / 2;
    const cx = w / 2;
    const left = map.unproject([cx - TARGET_PX / 2, h2]);
    const right = map.unproject([cx + TARGET_PX / 2, h2]);
    const metersForTargetPx = haversineMeters([left.lng, left.lat], [right.lng, right.lat]);
    const metersPerPx = metersForTargetPx / TARGET_PX;
    const niceMeters = getRoundNum(metersForTargetPx);
    const barPx = niceMeters / metersPerPx;
    let label;
    if (niceMeters >= 1000) {
      label = (niceMeters / 1000) + ' km';
    } else {
      label = niceMeters + ' m';
    }
    return { barPx: barPx, label: label };
  }

  function ScaleBarPanel() {
    const containerRef = React.useRef(null);
    const state = React.useState({ barPx: TARGET_PX, label: '' });
    const scale = state[0];
    const setScale = state[1];

    const themeState = React.useState(isScaleBarDarkMode());
    const dark = themeState[0];
    const setDark = themeState[1];

    React.useEffect(function () {
      const map = api.utils.getMap();
      if (!map) return;

      function update() {
        setScale(computeScale(map));
      }

      update();
      map.on('move', update);
      map.on('zoom', update);

      return function () {
        map.off('move', update);
        map.off('zoom', update);
      };
    }, []);

    React.useEffect(function () {
      function recheck() {
        setDark(isScaleBarDarkMode());
      }
      let mq;
      if (window.matchMedia) {
        mq = window.matchMedia('(prefers-color-scheme: dark)');
        mq.addEventListener('change', recheck);
      }
      const observer = new MutationObserver(recheck);
      observer.observe(document.documentElement, { attributes: true, attributeFilter: ['data-theme', 'class', 'style'] });
      observer.observe(document.body, { attributes: true, attributeFilter: ['data-theme', 'class', 'style'] });
      const pollId = setInterval(recheck, 1000);
      return function () {
        if (mq) mq.removeEventListener('change', recheck);
        observer.disconnect();
        clearInterval(pollId);
      };
    }, []);

    React.useEffect(function () {
      const node = containerRef.current;
      if (!node) return;
      let el = node.parentElement;
      let steps = 0;
      let panelRoot = null;
      while (el && steps < 8) {
        const cls = typeof el.className === 'string' ? el.className : '';
        if (cls.indexOf('bg-background') !== -1) {
          el.style.setProperty('background', dark ? 'rgba(20,20,22,0.55)' : 'rgba(255,255,255,0.55)', 'important');
          el.style.setProperty('backdrop-filter', 'blur(3px)', 'important');
          el.style.setProperty('box-shadow', 'none', 'important');
          el.style.setProperty('border', 'none', 'important');
          el.style.setProperty('border-radius', '8px', 'important');
        }
        if (cls.indexOf('fixed') !== -1 && cls.indexOf('z-50') !== -1) {
          panelRoot = el;
        }
        el = el.parentElement;
        steps++;
      }
      if (panelRoot) {
        const header = panelRoot.querySelector('.cursor-move');
        if (header) {
          header.style.setProperty('display', 'none', 'important');
        }
        const handles = panelRoot.querySelectorAll('.resize-handle');
        handles.forEach(function (handle) {
          handle.style.setProperty('border', 'none', 'important');
          handle.style.setProperty('outline', 'none', 'important');
          handle.style.setProperty('background', 'transparent', 'important');
          handle.style.setProperty('box-shadow', 'none', 'important');
        });
      }
    }, [dark]);

    const inkColor = dark ? '#f0f0f0' : '#1a1a1a';

    return h(
      'div',
      { ref: containerRef, style: { padding: '8px 12px', fontFamily: '-apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif' } },
      h(
        'div',
        { style: { fontSize: 11, color: inkColor, opacity: 0.9, marginBottom: 4, textAlign: 'center', textShadow: dark ? '0 0 3px rgba(0,0,0,0.8)' : 'none' } },
        scale.label
      ),
      h(
        'div',
        {
          style: {
            width: scale.barPx + 'px',
            height: 6,
            margin: '0 auto',
            borderLeft: '2px solid ' + inkColor,
            borderRight: '2px solid ' + inkColor,
            borderBottom: '2px solid ' + inkColor,
            filter: dark ? 'drop-shadow(0 0 2px rgba(0,0,0,0.8))' : 'none'
          }
        }
      )
    );
  }

  let panelAdded = false;

  api.hooks.onMapReady(function () {
    if (panelAdded) return;
    panelAdded = true;
    try {
      localStorage.setItem('floating-panel-<CITY_CODE_LOWER>-scalebar-panel', JSON.stringify({ x: 1400, y: 950, width: 180, height: 50 }));
    } catch (e) {}
    api.ui.addFloatingPanel({
      id: '<CITY_CODE_LOWER>-scalebar-panel',
      title: 'Scale',
      icon: 'Ruler',
      defaultWidth: 180,
      defaultHeight: 70,
      minWidth: 120,
      minHeight: 60,
      render: function () { return h(ScaleBarPanel); }
    });
  });

  api.hooks.onGameEnd(function () {
    panelAdded = false;
  });
})();
```

3. Append the above into `index.js`:

```batch
type "%USERPROFILE%\Desktop\SB_mod_project\02_intermediate\scalebar_index-js.js" >> "%USERPROFILE%\Desktop\SB_mod_project\03_mod\<CITY_CODE_LOWER>-mod\index.js"
```

### 9b. Administrative boundaries

We will have to use different sources for the boundaries. OpenStreetMap data is poor below the taluk level, hence we will use SHRUG data for village boundaries.

They don't always align since they are two independent sources, hence we will be seeing overlapping village-taluk and village-district boundaries.

1. Determine OSM Relation IDs and names for districts and taluks within `bbox`. Take note of them for the following step:

```python
cat > ~/SB_mod_project/02_intermediate/discover_admin_relations.py << 'PYEOF'
import json, time, requests

OVERPASS_URL = "https://overpass-api.de/api/interpreter"
HEADERS = {"User-Agent": "SB-mod-boundary-discovery/1.0 (contact: <YOUR_EMAIL>)"}
BBOX = [<BBOX_EXTRACT_TYPE-2>]  # [south, west, north, east] not [<west>, <south>, <east>, <north>] that was used earlier

LEVELS = [5, 6]

def discover(level, retries=6, backoff=30):
    south, west, north, east = BBOX
    query = f"""
    [out:json][timeout:180];
    relation["boundary"="administrative"]["admin_level"="{level}"]({south},{west},{north},{east});
    out tags center;
    """
    for attempt in range(retries):
        try:
            resp = requests.post(OVERPASS_URL, data={"data": query}, headers=HEADERS, timeout=200)
            if resp.status_code == 200:
                return resp.json()["elements"]
            print(f"  [level {level}] HTTP {resp.status_code}, attempt {attempt+1}/{retries}")
        except requests.RequestException as e:
            print(f"  [level {level}] request error: {e}, attempt {attempt+1}/{retries}")
        time.sleep(backoff)
    raise RuntimeError(f"Overpass discovery failed for admin_level={level} after {retries} attempts")

if __name__ == "__main__":
    for level in LEVELS:
        label = "district (admin_level=5)" if level == 5 else "taluk (admin_level=6)"
        print(f"\n===== {label} candidates intersecting bbox =====")
        elements = discover(level)
        for el in sorted(elements, key=lambda e: e.get("tags", {}).get("name", "")):
            tags = el.get("tags", {})
            name = tags.get("name", "<no name tag>")
            name_en = tags.get("name:en", "")
            state = tags.get("is_in:state", tags.get("addr:state", ""))
            print(f"  id={el['id']:<12} name={name!r:<30} name:en={name_en!r:<25} state={state!r}")
        print(f"  ({len(elements)} candidates)")

    print('\nNext: Copy this output into "DISTRICT_RELATIONS" / "TALUK_RELATIONS" in the next step.')

PYEOF
python3 ~/SB_mod_project/02_intermediate/discover_admin_relations.py
```

2. Obtain boundaries for those districts and taluks:

```python
cat > ~/SB_mod_project/02_intermediate/extract_admin_boundaries.py << 'PYEOF'
import json, os, time, requests
from shapely.geometry import Polygon, MultiPolygon, box, mapping
from shapely.ops import polygonize, unary_union

OVERPASS_URL = "https://overpass-api.de/api/interpreter"
HEADERS = {"User-Agent": "SB-mod-boundary-extractor/1.0 (contact: <YOUR_EMAIL>)"}
BBOX = [<BBOX_EXTRACT_TYPE-2>]  # [south, west, north, east] not [<west>, <south>, <east>, <north>] that was used earlier
BBOX_POLY = box(<BBOX_EXTRACT>)

DISTRICT_RELATIONS = {
    <District_1_ID>: "<District_1_Name>",
    <District_2_ID>: "<District_2_Name>",
    <...>: "<...>",
}
TALUK_RELATIONS = {
    <Taluk_1_ID>: "<Taluk_1_Name>",
    <Taluk_2_ID>: "<Taluk_2_Name>",
    <...>: "<...>",
}

def fetch_relation_full_geometry(rel_id, retries=6, backoff=30):
    query = f"""
    [out:json][timeout:180];
    rel({rel_id});
    (._;>;);
    out geom;
    """
    for attempt in range(retries):
        try:
            resp = requests.post(OVERPASS_URL, data={"data": query}, headers=HEADERS, timeout=200)
            if resp.status_code == 200:
                return resp.json()
            print(f"  [rel {rel_id}] HTTP {resp.status_code}, attempt {attempt+1}/{retries}")
        except requests.RequestException as e:
            print(f"  [rel {rel_id}] request error: {e}, attempt {attempt+1}/{retries}")
        time.sleep(backoff)
    raise RuntimeError(f"Overpass fetch failed for relation {rel_id} after {retries} attempts")

def assemble_polygon(osm_json, rel_id):
    """Build the relation's own real polygon from its member ways,
    respecting inner/outer roles, via polygonize. No dependency on
    which fragments happen to fall inside any bbox."""
    ways = {}
    rel = None
    for el in osm_json["elements"]:
        if el["type"] == "way":
            coords = [(pt["lon"], pt["lat"]) for pt in el["geometry"]]
            ways[el["id"]] = coords
        elif el["type"] == "relation" and el["id"] == rel_id:
            rel = el

    if rel is None:
        raise ValueError(f"Relation {rel_id} not found in response")

    outer_lines, inner_lines = [], []
    for member in rel.get("members", []):
        if member["type"] != "way" or member["ref"] not in ways:
            continue
        coords = ways[member["ref"]]
        if len(coords) < 2:
            continue
        if member.get("role") == "inner":
            inner_lines.append(coords)
        else:
            outer_lines.append(coords)

    from shapely.geometry import LineString
    outer_polys = list(polygonize([LineString(c) for c in outer_lines]))
    if not outer_polys:
        raise ValueError(f"Relation {rel_id}: outer ways did not close into a polygon")
    outer = unary_union(outer_polys)

    if inner_lines:
        inner_polys = list(polygonize([LineString(c) for c in inner_lines]))
        if inner_polys:
            outer = outer.difference(unary_union(inner_polys))

    return outer

def process_level(relations, level_name):
    features = []
    for rel_id, name in relations.items():
        print(f"[{level_name}] fetching relation {rel_id} ({name})...")
        osm_json = fetch_relation_full_geometry(rel_id)
        try:
            full_geom = assemble_polygon(osm_json, rel_id)
        except ValueError as e:
            print(f"  SKIPPED: {e}")
            continue

        clipped = full_geom.intersection(BBOX_POLY)
        if clipped.is_empty:
            print(f"  NOTE: {name} (rel {rel_id}) does not actually reach the bbox -- excluding.")
            continue
        if not clipped.is_valid:
            clipped = clipped.buffer(0)

        features.append({
            "type": "Feature",
            "geometry": mapping(clipped),
            "properties": {"level": level_name, "name": name, "rel_id": rel_id}
        })
    return features

if __name__ == "__main__":
    district_feats = process_level(DISTRICT_RELATIONS, "district")
    taluk_feats = process_level(TALUK_RELATIONS, "taluk")

    out_dir = os.path.expanduser("~/SB_mod_project/02_intermediate")
    with open(os.path.join(out_dir, "admin_districts.geojson"), "w") as f:
        json.dump({"type": "FeatureCollection", "features": district_feats}, f)
    with open(os.path.join(out_dir, "admin_taluks.geojson"), "w") as f:
        json.dump({"type": "FeatureCollection", "features": taluk_feats}, f)

    print(f"Done: {len(district_feats)} districts, {len(taluk_feats)} taluks written.")
PYEOF
python3 ~/SB_mod_project/02_intermediate/extract_admin_boundaries.py
```

3. Obtaining boundaries for villages from `SHRUG`:

```python
cat > ~/SB_mod_project/02_intermediate/extract_village_boundaries.py << 'PYEOF'
import json, os
import geopandas as gpd
from shapely.geometry import box, mapping

SHRUG_GPKG = os.path.expanduser("~/SB_mod_project/01_source/village_modified.gpkg")
BBOX = (<BBOX_EXTRACT>)
BBOX_POLY = box(*BBOX)

gdf = gpd.read_file(SHRUG_GPKG, bbox=BBOX)
before_count = len(gdf)

gdf["geometry"] = gdf["geometry"].intersection(BBOX_POLY)
gdf = gdf[~gdf["geometry"].is_empty]

features = []
for _, row in gdf.iterrows():
    geom = row["geometry"]
    if not geom.is_valid:
        geom = geom.buffer(0)
    features.append({
        "type": "Feature",
        "geometry": mapping(geom),
        "properties": {
            "level": "village",
            "name": row.get("town_village_name", ""),
            "pc11_code": str(row.get("pc11_town_village_id", ""))
        }
    })

out_path = os.path.expanduser("~/SB_mod_project/02_intermediate/admin_villages.geojson")
with open(out_path, "w") as f:
    json.dump({"type": "FeatureCollection", "features": features}, f)

print(f"before bbox-filter clip: {before_count} rows; after clip & empty-drop: {len(features)} features written.")
PYEOF
python3 ~/SB_mod_project/02_intermediate/extract_village_boundaries.py
```

4. Boundary name labelling:

```python
cat > ~/SB_mod_project/02_intermediate/build_boundary_labels.py << 'PYEOF'
import json, os
from shapely.geometry import shape, mapping, box, GeometryCollection, Point, MultiPoint
from shapely.ops import unary_union

IN_DIR = os.path.expanduser("~/SB_mod_project/02_intermediate")
LEVELS = {
    "district": "admin_districts.geojson",
    "taluk": "admin_taluks.geojson",
    "village": "admin_villages.geojson"
}
SHARED_SNAP_TOL = 0.00005

BBOX_MINX_MINY_MAXX_MAXY = (<BBOX_EXTRACT>)
BBOX_POLY = box(*BBOX_MINX_MINY_MAXX_MAXY)
BBOX_EDGE_TOL = 0.0001

def load(level):
    with open(os.path.join(IN_DIR, LEVELS[level])) as f:
        data = json.load(f)
    regions = []
    for idx, feat in enumerate(data["features"]):
        props = feat["properties"]
        ext_id = props.get("rel_id", props.get("pc11_code", ""))
        regions.append({
            "idx": idx,
            "name": props.get("name", ""),
            "ext_id": ext_id,
            "geom": shape(feat["geometry"])
        })
    return regions

def boundary_line(poly):
    if poly.geom_type == "Polygon":
        return poly.boundary
    return unary_union([p.boundary for p in poly.geoms])

def extract_lines_only(geom):
    if geom.is_empty:
        return None
    if geom.geom_type in ("LineString", "MultiLineString"):
        return geom
    if geom.geom_type in ("Point", "MultiPoint"):
        return None
    if geom.geom_type == "GeometryCollection":
        lines = [g for g in geom.geoms if g.geom_type in ("LineString", "MultiLineString")]
        if not lines:
            return None
        return unary_union(lines) if len(lines) > 1 else lines[0]
    return None

def is_bbox_clip_artifact(shared_geom):
    edge_band = BBOX_POLY.boundary.buffer(BBOX_EDGE_TOL)
    remainder = shared_geom.difference(edge_band)
    if remainder.is_empty:
        return True
    remainder_length = getattr(remainder, "length", 0)
    shared_length = shared_geom.length
    return shared_length > 0 and (remainder_length / shared_length) < 0.02

def build_labels_for_level(level):
    regions = load(level)
    solo_feats, shared_feats = [], []
    remaining_boundary = {r["idx"]: boundary_line(r["geom"]) for r in regions}
    bbox_artifacts_skipped = 0
    point_touches_skipped = 0

    n = len(regions)
    for i in range(n):
        a = regions[i]
        for j in range(i + 1, n):
            b = regions[j]
            if not a["geom"].buffer(SHARED_SNAP_TOL).intersects(b["geom"].buffer(SHARED_SNAP_TOL)):
                continue

            shared_raw = remaining_boundary[a["idx"]].intersection(
                remaining_boundary[b["idx"]].buffer(SHARED_SNAP_TOL)
            )
            if shared_raw.is_empty:
                continue

            shared = extract_lines_only(shared_raw)
            if shared is None or shared.is_empty:
                point_touches_skipped += 1
                continue

            if is_bbox_clip_artifact(shared):
                bbox_artifacts_skipped += 1
                continue

            if not shared.is_valid:
                shared = shared.buffer(0)

            shared_feats.append({
                "type": "Feature",
                "geometry": mapping(shared),
                "properties": {
                    "level": level, "label_type": "shared",
                    "label": f"{a['name']} \u2013 {b['name']}",
                    "idx_a": a["idx"], "idx_b": b["idx"],
                    "ext_id_a": a["ext_id"], "ext_id_b": b["ext_id"]
                }
            })
            remaining_boundary[a["idx"]] = remaining_boundary[a["idx"]].difference(shared.buffer(SHARED_SNAP_TOL))
            remaining_boundary[b["idx"]] = remaining_boundary[b["idx"]].difference(shared.buffer(SHARED_SNAP_TOL))

    for r in regions:
        solo_line = remaining_boundary[r["idx"]]
        if solo_line.is_empty:
            continue
        solo_feats.append({
            "type": "Feature",
            "geometry": mapping(solo_line),
            "properties": {"level": level, "label_type": "solo", "label": r["name"], "idx": r["idx"], "ext_id": r["ext_id"]}
        })

    print(f"  [{level}] skipped: {bbox_artifacts_skipped} bbox-clip-edge artifacts, {point_touches_skipped} point-only touches")
    return solo_feats + shared_feats

if __name__ == "__main__":
    all_feats = []
    for level in LEVELS:
        feats = build_labels_for_level(level)
        print(f"[{level}] {sum(1 for f in feats if f['properties']['label_type']=='solo')} solo, "
              f"{sum(1 for f in feats if f['properties']['label_type']=='shared')} shared")
        all_feats.extend(feats)

    out_path = os.path.join(IN_DIR, "admin_boundary_labels.geojson")
    with open(out_path, "w") as f:
        json.dump({"type": "FeatureCollection", "features": all_feats}, f)
    print(f"Total label features written: {len(all_feats)}")
PYEOF
python3 ~/SB_mod_project/02_intermediate/build_boundary_labels.py
```

5. Generating vector tiles file:

```bash
cd ~/SB_mod_project/02_intermediate

tippecanoe -o boundaries.mbtiles --force \
  -Z 6 -z 14 \
  --generate-ids \
  -L admin_districts:admin_districts.geojson \
  -L admin_taluks:admin_taluks.geojson \
  -L admin_villages:admin_villages.geojson \
  -L admin_boundary_labels:admin_boundary_labels.geojson \
  --no-tile-size-limit

pmtiles convert boundaries.mbtiles boundaries.pmtiles
```

6. Send the file to Windows using cmd:

```batch
scp <VM_USER>@<VM_IP>:~/SB_mod_project/02_intermediate/boundaries.pmtiles "%USERPROFILE%\Desktop\SB_mod_project\03_mod\<CITY_CODE_LOWER>-mod\tiles"
```

7. Run below in cmd:

```batch
notepad "%USERPROFILE%\Desktop\SB_mod_project\02_intermediate\boundaries-1_index-js.js"
```

8. Paste into the notepad:

```javascript


/*----------------------------------------------------
--------------------- Boundaries ---------------------
----------------------------------------------------*/

window.SubwayBuilderAPI.map.registerSource('<CITY_CODE_LOWER>boundaries-source', {
  type: 'vector',
  tiles: ['http://127.0.0.1:8080/boundaries/{z}/{x}/{y}.mvt'],
  maxzoom: 14
});

/*-------------------- District --------------------*/
window.SubwayBuilderAPI.map.registerLayer({
  id: '<CITY_CODE_LOWER>admin-district-line', type: 'line',
  source: '<CITY_CODE_LOWER>boundaries-source', 'source-layer': 'admin_districts',
  layout: { visibility: 'none' },
  paint: {'line-color': '#8b2f4a','line-width': ['interpolate', ['linear'], ['zoom'], 8, 1, 13, 2.5],'line-opacity': 0.75}
}, 'road-labels');
window.SubwayBuilderAPI.map.registerLayer({
  id: '<CITY_CODE_LOWER>admin-district-fill', type: 'fill',
  source: '<CITY_CODE_LOWER>boundaries-source', 'source-layer': 'admin_districts',
  layout: { visibility: 'none' },
  paint: { 'fill-color': ['match', ['%', ['id'], 5], 0, '#e63946', 1, '#f4a261', 2, '#2a9d8f', 3, '#264653', '#8b2f4a'], 'fill-opacity': 0.12 }
}, 'road-labels');

/*--------------------- Taluk ----------------------*/
window.SubwayBuilderAPI.map.registerLayer({
  id: '<CITY_CODE_LOWER>admin-taluk-line', type: 'line',
  source: '<CITY_CODE_LOWER>boundaries-source', 'source-layer': 'admin_taluks',
  layout: { visibility: 'none' },
  paint: {'line-color': '#7a5ca8','line-width': ['interpolate', ['linear'], ['zoom'], 8, 0.6, 13, 1.5],'line-dasharray': [3, 2],'line-opacity': 0.65}
}, 'road-labels');
window.SubwayBuilderAPI.map.registerLayer({
  id: '<CITY_CODE_LOWER>admin-taluk-fill', type: 'fill',
  source: '<CITY_CODE_LOWER>boundaries-source', 'source-layer': 'admin_taluks',
  layout: { visibility: 'none' },
  paint: {
    'fill-color': ['match', ['%', ['id'], 8],
      0, '#e63946', 1, '#f4a261', 2, '#2a9d8f', 3, '#264653',
      4, '#8ecae6', 5, '#ffb703', 6, '#9d4edd', '#606c38'],
    'fill-opacity': 0.10
  }
}, 'road-labels');

/*-------------------- Village ---------------------*/
window.SubwayBuilderAPI.map.registerLayer({
  id: '<CITY_CODE_LOWER>admin-village-line', type: 'line',
  source: '<CITY_CODE_LOWER>boundaries-source', 'source-layer': 'admin_villages',
  layout: { visibility: 'none' },
  minzoom: 11,
  paint: {'line-color': '#5c8a6e', 'line-width': ['interpolate', ['linear'], ['zoom'], 11, 0.4, 15, 1], 'line-dasharray': [1, 1.5], 'line-opacity': 0.55}
}, 'road-labels');
window.SubwayBuilderAPI.map.registerLayer({
  id: '<CITY_CODE_LOWER>admin-village-fill', type: 'fill',
  source: '<CITY_CODE_LOWER>boundaries-source', 'source-layer': 'admin_villages',
  layout: { visibility: 'none' },
  paint: {
    'fill-color': ['match', ['%', ['id'], 8],
      0, '#e63946', 1, '#f4a261', 2, '#2a9d8f', 3, '#264653',
      4, '#8ecae6', 5, '#ffb703', 6, '#9d4edd', '#606c38'],
    'fill-opacity': 0.14
  }
}, 'road-labels');

/*---------------- Boundary Labels -----------------*/
var BOUNDARY_LABEL_COLOR = { district: '#8b2f4a', taluk: '#7a5ca8', village: '#5c8a6e' };
['district', 'taluk', 'village'].forEach(function (lvl) {
  window.SubwayBuilderAPI.map.registerLayer({
    id: '<CITY_CODE_LOWER>admin-' + lvl + '-boundary-labels', type: 'symbol',
    source: '<CITY_CODE_LOWER>boundaries-source', 'source-layer': 'admin_boundary_labels',
    filter: ['==', ['get', 'level'], lvl],
    layout: {
      visibility: 'none',
      'symbol-placement': 'line',
      'text-field': ['coalesce', ['get', 'label'], ''],
      'text-size': 10, 'text-font': ['Noto Sans Regular'],
      'symbol-spacing': 300, 'text-max-angle': 30
    },
    paint: { 'text-color': BOUNDARY_LABEL_COLOR[lvl], 'text-halo-color': '#ffffff', 'text-halo-width': 1 }
  }, 'road-labels');
});

/*------------------ Fill Labels -------------------*/
window.SubwayBuilderAPI.map.registerLayer({
  id: '<CITY_CODE_LOWER>admin-district-fill-labels', type: 'symbol',
  source: '<CITY_CODE_LOWER>boundaries-source', 'source-layer': 'admin_districts',
  layout: { visibility: 'none', 'text-field': ['coalesce', ['get', 'name'], ''], 'text-size': 14, 'text-font': ['Noto Sans Medium'] },
  paint: { 'text-color': '#8b2f4a', 'text-halo-color': '#ffffff', 'text-halo-width': 1.5 }
}, 'road-labels');
window.SubwayBuilderAPI.map.registerLayer({
  id: '<CITY_CODE_LOWER>admin-taluk-fill-labels', type: 'symbol',
  source: '<CITY_CODE_LOWER>boundaries-source', 'source-layer': 'admin_taluks',
  layout: { visibility: 'none', 'text-field': ['coalesce', ['get', 'name'], ''], 'text-size': 14, 'text-font': ['Noto Sans Medium'] },
  paint: { 'text-color': '#7a5ca8', 'text-halo-color': '#ffffff', 'text-halo-width': 1.5 }
}, 'road-labels');
window.SubwayBuilderAPI.map.registerLayer({
  id: '<CITY_CODE_LOWER>admin-village-fill-labels', type: 'symbol',
  source: '<CITY_CODE_LOWER>boundaries-source', 'source-layer': 'admin_villages',
  layout: { visibility: 'none', 'text-field': ['coalesce', ['get', 'name'], ''], 'text-size': 14, 'text-font': ['Noto Sans Medium'] },
  paint: { 'text-color': '#5c8a6e', 'text-halo-color': '#ffffff', 'text-halo-width': 1.5 }
}, 'road-labels');
```

9. Append the above into `index.js`:

```batch
type "%USERPROFILE%\Desktop\SB_mod_project\02_intermediate\boundaries-1_index-js.js" >> "%USERPROFILE%\Desktop\SB_mod_project\03_mod\<CITY_CODE_LOWER>-mod\index.js"
```

10. Run the below in cmd:

```batch
notepad "%USERPROFILE%\Desktop\SB_mod_project\02_intermediate\boundaries-2_index-js.ps1"
```

11. Paste into the notepad:

```powershell
param(
    [Parameter(Mandatory=$true)]
    [string]$IndexPath
)

if (-not (Test-Path $IndexPath)) {
    Write-Host "ABORTED: file not found at $IndexPath"
    exit 1
}

$content = Get-Content -Raw -LiteralPath $IndexPath

function Count-Occurrences($haystack, $needle) {
    if ($needle.Length -eq 0) { return 0 }
    $count = 0
    $idx = 0
    while ($true) {
        $found = $haystack.IndexOf($needle, $idx, [System.StringComparison]::Ordinal)
        if ($found -lt 0) { break }
        $count++
        $idx = $found + $needle.Length
    }
    return $count
}

function Apply-Edit($content, $oldStr, $newStr, $editName) {
    $count = Count-Occurrences $content $oldStr
    if ($count -ne 1) {
        Write-Host "ABORTED: $editName anchor not found exactly once (found $count). File doesn't match expected content."
        exit 1
    }
    return $content.Replace($oldStr, $newStr)
}

# Edit 1: Insert Admin Boundaries into Legend
# Anchored on the Utilities category's closing `}` + the array's closing
# `];`. layerIds match
# the actual layer ids already registered in the Boundaries section at the
# bottom of the file).
$oldUtilitiesClose = @'
        { label: 'Power Generator - Unknown Source', iconKey: 'unknown', layerIds: ['<CITY_CODE_LOWER>power-generators', '<CITY_CODE_LOWER>power-generators-labels'] }
      ]
    }
  ];
'@

$newUtilitiesClose = @'
        { label: 'Power Generator - Unknown Source', iconKey: 'unknown', layerIds: ['<CITY_CODE_LOWER>power-generators', '<CITY_CODE_LOWER>power-generators-labels'] }
      ]
    },
    {
      category: 'Admin / Ward Boundaries',
      items: [
        { label: 'District Boundary', color: 'transparent', outline: '#8b2f4a', layerIds: ['<CITY_CODE_LOWER>admin-district-line', '<CITY_CODE_LOWER>admin-district-boundary-labels'], independentToggle: true, defaultOn: false },
        { label: 'District Fill', color: '#8b2f4a', outline: '#8b2f4a', layerIds: ['<CITY_CODE_LOWER>admin-district-fill', '<CITY_CODE_LOWER>admin-district-fill-labels'], independentToggle: true, defaultOn: false },
        { label: 'Taluk Boundary', color: 'transparent', outline: '#7a5ca8', layerIds: ['<CITY_CODE_LOWER>admin-taluk-line', '<CITY_CODE_LOWER>admin-taluk-boundary-labels'], independentToggle: true, defaultOn: false },
        { label: 'Taluk Fill', color: '#7a5ca8', outline: '#7a5ca8', layerIds: ['<CITY_CODE_LOWER>admin-taluk-fill', '<CITY_CODE_LOWER>admin-taluk-fill-labels'], independentToggle: true, defaultOn: false },
        { label: 'Village/Town Boundary', color: 'transparent', outline: '#5c8a6e', layerIds: ['<CITY_CODE_LOWER>admin-village-line', '<CITY_CODE_LOWER>admin-village-boundary-labels'], independentToggle: true, defaultOn: false, info: 'Village boundaries may not match with taluk and district boundaries due to using different sources' },
        { label: 'Village/Town Fill', color: '#5c8a6e', outline: '#5c8a6e', layerIds: ['<CITY_CODE_LOWER>admin-village-fill', '<CITY_CODE_LOWER>admin-village-fill-labels'], independentToggle: true, defaultOn: false, info: 'Village boundaries may not match with taluk and district boundaries due to using different sources' }
      ]
    }
  ];
'@

$content = Apply-Edit $content $oldUtilitiesClose $newUtilitiesClose "Utilities-close / insert Admin-Ward-Boundaries legendData"

# Edit 2
$oldBoundaryColors = @'
  const BOUNDARY_ROW_COLORS = {};
'@

$newBoundaryColors = @'
  const BOUNDARY_ROW_COLORS = {
    'District Boundary': { light: '#8b2f4a', dark: '#e08bab' },
    'District Fill': { light: '#8b2f4a', dark: '#e08bab' },
    'Taluk Boundary': { light: '#7a5ca8', dark: '#c3aef0' },
    'Taluk Fill': { light: '#7a5ca8', dark: '#c3aef0' },
    'Village/Town Boundary': { light: '#5c8a6e', dark: '#9bd6ae' },
    'Village Fill': { light: '#5c8a6e', dark: '#9bd6ae' },
    'Railway Line': { light: '#1a1a1a', dark: '#d9d9d9' }
  };
'@

$content = Apply-Edit $content $oldBoundaryColors $newBoundaryColors "Populate BOUNDARY_ROW_COLORS"

# Edit 3
$oldSwatchStart = @'
  function Swatch(props) {
'@

$newSwatchStart = @'
  const PLACE_LABEL_LAYERS = ['city-markers-label', 'neighborhood-labels', 'suburb-labels', 'city-labels'];

  function setPlaceLabelsVisible(map, visible) {
    PLACE_LABEL_LAYERS.forEach(function (id) {
      if (map.getLayer(id)) {
        map.setLayoutProperty(id, 'visibility', visible ? 'visible' : 'none');
      }
    });
  }

  function Swatch(props) {
'@

$content = Apply-Edit $content $oldSwatchStart $newSwatchStart "Insert PLACE_LABEL_LAYERS constant"

# Edit 4: Hiding base map place labels while any boundary fill is on
$oldHandleToggleEnd = @'
      }

      setCheckedMap(next);
      window.__<CITY_CODE_LOWER>LegendToggleState[group.category] = Object.assign({}, window.__<CITY_CODE_LOWER>LegendToggleState[group.category], { checkedMap: next });
      applyToggle(item, val);
    }
'@

$newHandleToggleEnd = @'
      }

      if (group.category === 'Admin / Ward Boundaries') {
        const FILL_LABELS = ['District Fill', 'Taluk Fill', 'Village Fill'];
        if (FILL_LABELS.indexOf(item.label) !== -1) {
          const anyFillOn = FILL_LABELS.some(function (label) { return next[label]; });
          const map = api.utils.getMap();
          if (map) setPlaceLabelsVisible(map, !anyFillOn);
        }
      }

      setCheckedMap(next);
      window.__<CITY_CODE_LOWER>LegendToggleState[group.category] = Object.assign({}, window.__<CITY_CODE_LOWER>LegendToggleState[group.category], { checkedMap: next });
      applyToggle(item, val);
    }
'@

$content = Apply-Edit $content $oldHandleToggleEnd $newHandleToggleEnd "Hook place-label hiding into handleToggle"

# Edit 5: InfoTooltip component
$oldSwatchStart2 = @'
  function Swatch(props) {
'@

$newSwatchStart2 = @'
  function InfoTooltip(props) {
    const c = api.utils.components;
    return h(c.TooltipProvider, {},
      h(c.Tooltip, {},
        h('span', { style: { position: 'relative', display: 'inline-flex' } },
          h(c.TooltipTrigger, { asChild: true },
            h('span', {
              style: { marginLeft: 4, cursor: 'help', fontSize: 11, opacity: 0.6, flexShrink: 0 }
            }, 'ⓘ')
          ),
          h(c.TooltipContent, {
            style: {
              width: 150, maxWidth: 150, fontSize: 12, whiteSpace: 'normal',
              position: 'absolute', bottom: 'calc(100% + 4px)', left: '50%',
              transform: 'translateX(-50%)'
            }
          }, props.text)
        )
      )
    );
  }

  function Swatch(props) {
'@

$content = Apply-Edit $content $oldSwatchStart2 $newSwatchStart2 "Insert InfoTooltip component"

# Edit 6: Wire InfoTooltip into Legend
$oldLegendRowLabel = @'
      h('span', { style: { fontSize: 12, flex: 1, color: dark ? '#e8e8e8' : '#1a1a1a' } }, item.label),
'@

$newLegendRowLabel = @'
      h('span', { style: { fontSize: 12, flex: 1, display: 'flex', alignItems: 'center', color: dark ? '#e8e8e8' : '#1a1a1a' } },
        item.label,
        item.info ? h(InfoTooltip, { text: item.info }) : null
      ),
'@

$content = Apply-Edit $content $oldLegendRowLabel $newLegendRowLabel "Wire InfoTooltip into Legend"

Set-Content -LiteralPath $IndexPath -Value $content -NoNewline
Write-Host "SUCCESS: Admin/Ward Boundaries category inserted, place-label auto-hide wired in, Village Fill info-tooltip added (verify visually)."
```

12. Merge into `index.js`:

```batch
powershell -ExecutionPolicy Bypass -File "%USERPROFILE%\Desktop\SB_mod_project\02_intermediate\boundaries-2_index-js.ps1" -IndexPath "%USERPROFILE%\Desktop\SB_mod_project\03_mod\<CITY_CODE_LOWER>-mod\index.js"
```

13. Take a backup of the final state of `maps.py`:

```batch
scp <VM_USER>@<VM_IP>:~/SB_mod_project/tools/depot/src/depot/maps.py "%USERPROFILE%\Desktop\SB_mod_project\02_intermediate"
ren "%USERPROFILE%\Desktop\SB_mod_project\02_intermediate\maps.py" "final_maps.py"
```

### 9c. Test in game

1. Move the updated mod files to the respective game folder:

```batch
xcopy "%USERPROFILE%\Desktop\SB_mod_project\03_mod" "%APPDATA%\metro-maker4\mods"  /E /I /D /Y
```

2. Run the server again (if it was closed previously) and check if all the scale bar and boundaries appear correctly.
3. Remove the easily regeneratable scripts on the Windows side if needed:

```bash
setlocal enabledelayedexpansion & set "root=%USERPROFILE%\Desktop\SB_mod_project\02_intermediate" & for %F in ("%root%\*") do (if /I not "%~nxF"=="initial_maps.py" if /I not "%~nxF"=="final_maps.py" del /q "%F")
```

----

## Stage 10: Sharing or publishing the mod files

*Est. time: 30 minutes*

### 10a. Sharing directly with someone

Just directly share the `<CITY_CODE_LOWER>-mod\` folder (either from the Desktop or the game folder). Ask them to copy it to `%USERPROFILE%\AppData\Roaming\metro-maker4\mods` and then copy the `<CITY_CODE>` sub-folder under `%USERPROFILE%\AppData\Roaming\metro-maker4\mods\<CITY_CODE_LOWER>-mod\data` to `%USERPROFILE%\AppData\Roaming\metro-maker4\cities\data`. After that, they will have to activate the mod in-game and run the tile server script located under the tools subfolder.

### 10b. Publishing broadly via `Subway Builder Modded`

There's a community-run registry (`Subway-Builder-Modded/registry` on GitHub) that lets players find and install custom cities. Its companion app, `Railyard` allows installing and playing without needing to set up a tile server manually.

Note that the registry only stores metadata and a pointer to where the mod files are hosted (e.g. a GitHub Release).

1. Zip the `data` and `tiles` folders and the `index.js` and `manifest.json` files into a '*<CITY_CODE>-mod-v1.0.0.zip*' file.

2. Create a repo on Codeberg/GitHub/etc. and create a release with the zip.

3. Create a `<CITY_CODE_LOWER>.json` file in the repo if using something other than GitHub and add the below. SHA256 can be obtained using `certutil -hashfile "<CITY_CODE>-mod-v1.0.0.zip" SHA256` in cmd.

```json
{
  "schema_version": 1,
  "versions": [
    {
      "version": "1.0.0",
      "game_version": ">=1.0.0",
      "date": "<Today_date>",
      "changelog": "Initial release",
      "download": "https://codeberg.org/<account_name>/<repo_name>/releases/download/v1.0.0/<CITY_CODE>-mod-v1.0.0.zip",
      "sha256": "<sha256_of_the_zip>"
    }
  ]
}
```

4. Go to https://github.com/Subway-Builder-Modded/registry/issues/new/choose and click "**Publish New Map**" to submit the mod. "**Source URL**" is the repo URL and "**Custom Update URL**" is the JSON url.

5. After submission, a PR is auto-created. Once it's merged, the mod is listed on `Railyard`.

----