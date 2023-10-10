### Anwendungen

Zeitliche Analyse:

- Überwachung von Veränderungen im Laufe der Zeit: Datacubes ermöglichen die Analyse von Veränderungen der Erdoberfläche im Laufe der Zeit, z. B. Veränderungen der Bodenbedeckung, Entwaldung, Verstädterung und saisonale Schwankungen.

Umweltüberwachung:

- Studien zum Klimawandel: Datacubes werden zur Untersuchung des Klimawandels verwendet, indem langfristige Trends bei Temperatur, Niederschlag und anderen Klimavariablen analysiert werden.

Überwachung von Naturkatastrophen: 
- Überwachung und Analyse von Daten vor, während und nach Naturkatastrophen, wie z. B. Wirbelstürmen, Erdbeben und Waldbränden, um die Auswirkungen zu bewerten und Reaktionsmaßnahmen zu planen.

Fernerkundungsdaten: 
- Integration verschiedener Fernerkundungsdatenquellen in einen einheitlichen datacube für umfassende Analysen.


### Beispiele
- Open Data Cube (ODC) ist ein Open-Source-Projekt zur Verwaltung und Analyse von Geodaten, das hilft die Leistungsfähigkeit von Satellitendaten zu nutzen. Im Kern besteht der ODC aus einer Reihe von Python-Bibliotheken und einer PostgreSQL-Datenbank
- Google Earth Engine verwendet datacubes, um eine skalierbare und interaktive Plattform für die Analyse von EO Daten bereitzustellen. Google Earth Engine ermöglicht den Zugriff auf und die Analyse von großen Geodatensätzen für Anwendungen wie Umweltüberwachung, Landbedeckungskartierung und Studien zum Klimawandel.
- Euro Data Cube wurde geschaffen, um die technischen Hürden zu senken, die bei der Gewinnung von Informationen aus Earth ovservation und deren Aufbereitung für die Analyse bestehen. Sie kombiniert EO-Daten aus verschiedenen Quellen an einem Ort, darunter Daten von den Sentinel-Satelliten, Daten von anderen kommerziellen Satellitenmissionen mit sehr hoher Auflösung sowie Produkte und Daten von Copernicus-Diensten und anderen EO-Initiativen.
- EODataBee integriert Sentinel-Daten, Daten des Copernicus-Dienstes und vom Nutzer gelieferte Daten in einem DataCube-System und generiert so hochwertige Informationen für die Industrie auf dem Küsten- und Binnenwassermarkt.


## Datacubes in openEO 
Daten werden in openEO als datacubes representiert. 
Die folgenden Eigenschaften sind in der Regel für Dimensionen verfügbar:

    name
    type (potential types include: spatial (raster or vector data), temporal and other data such as bands)
    axis (for spatial dimensions) / number
    labels (usually exposed through textual or numerical representations, in the metadata as nominal values and/or extents)
    reference system / projection
    resolution / step size
    unit (either explicitly specified or implicitly given by the reference system)
    additional information specific to the dimension type (e.g. the geometry types for a dimension containing geometries)



In this chapter we will show a full example of an earth observation use case using the JavaScript client in a Node.js environment and the Google Earth Engine back-end. 
We want to produce a monthly RGB composite of Sentinel 1 backscatter data over the area of Vienna, Austria for three months in 2017. This can be used for classification and crop monitoring.
https://openeo.org/assets/img/getting-started-result-example.7820ee84.jpg
Full example: 
```javascript
// Make the client available to the Node.js script
// Also include the Formula library for simple math expressions
const { OpenEO, Formula } = require('@openeo/js-client');

async function example() {
  // Connect to the back-end
  var con = await OpenEO.connect("https://earthengine.openeo.org");
  // Authenticate ourselves via Basic authentication
  await con.authenticateBasic("group11", "test123");
  // Create a process builder
  var builder = await con.buildProcess();
  // We are now loading the Sentinel-1 data over the Area of Interest
  var datacube = builder.load_collection(
    "COPERNICUS/S1_GRD",
    {west: 16.06, south: 48.06, east: 16.65, north: 48.35},
    ["2017-03-01", "2017-06-01"],
    ["VV"]
  );

  // Since we are creating a monthly RGB composite, we need three separated time ranges (March aas R, April as G and May as G).
  // Therefore, we split the datacube into three datacubes using a temporal filter.
  var march = builder.filter_temporal(datacube, ["2017-03-01", "2017-04-01"]);
  var april = builder.filter_temporal(datacube, ["2017-04-01", "2017-05-01"]);
  var may = builder.filter_temporal(datacube, ["2017-05-01", "2017-06-01"]);

  // We aggregate the timeseries values into a single image by reducing the time dimension using a mean reducer.
  var mean = function(data) {
    return this.mean(data);
  };
  march = builder.reduce_dimension(march, mean, "t");
  april = builder.reduce_dimension(april, mean, "t");
  may = builder.reduce_dimension(may, mean, "t");

  // Now the three images will be combined into the temporal composite.
  // We rename the bands to R, G and B as otherwise the bands are overlapping and the merge process would fail.
  march = builder.rename_labels(march, "bands", ["R"], ["VV"]);
  april = builder.rename_labels(april, "bands", ["G"], ["VV"]);
  may = builder.rename_labels(may, "bands", ["B"], ["VV"]);

  datacube = builder.merge_cubes(march, april);
  datacube = builder.merge_cubes(datacube, may);

  // To make the values match the RGB values from 0 to 255 in a PNG file, we need to scale them.
  // We can simplify expressing math formulas using the openEO Formula parser.
  datacube = builder.apply(datacube, new Formula("linear_scale_range(x, -20, -5, 0, 255)"));

  // Finally, save the result as PNG file.
  // In the options we specify which band should be used for "red", "green" and "blue" color.
  datacube = builder.save_result(datacube, "PNG", {
    red: "R",
    green: "G",
    blue: "B"
  });

  // Now send the processing instructions to the back-end for (synchronous) execution and save the file as result.png
  await con.downloadResult(datacube, "result.png");
}

// Run the example, write errors to the console.
example().catch(error => console.error(error));


```


https://openeo.org/documentation/1.0/javascript/
