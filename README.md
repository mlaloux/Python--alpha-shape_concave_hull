Python alpha shape ou concave-hull d'un nuage de points
===============================

repris de [The fading shape of alpha](http://sgillies.net/blog/1155/the-fading-shape-of-alpha/) de Sean Gillies pour m'en rappeler:

![](http://i.imgur.com/t0uJfUq.jpg)

1) calcul de Delanunay et extraction des arêtes (edges)


```Python

import fiona
points = [f['geometry']['coordinates'] for f in fiona.open('points_alpha_hull.shp')]
from scipy.spatial import Delaunay
tri = Delaunay(points)
edges = set()
edge_points = []
def add_edge(i, j):
     """Add a line between the i-th and j-th points, if not in the list already"""
     if (i, j) in edges or (j, i) in edges:
        # already added
        return
     edges.add( (i, j) )
     edge_points.append(points[ [i, j] ])
     
import numpy as np
points= np.array(points)
for ia, ib, ic in tri.vertices:
     add_edge(ia, ib)
     add_edge(ib, ic)
     add_edge(ic, ia)
     
from shapely.geometry import mapping
from shapely.geometry import MultiLineString
from shapely.ops import cascaded_union, polygonize
import json
m = MultiLineString(edge_points)
````

Delaunay

```Python
schema = {'geometry': 'LineString','properties': {'test': 'str'}}
with fiona.open('delaunay.shp','w','ESRI Shapefile', schema) as e:
       e.write({'geometry':mapping(m), 'properties':{'test':1}})
```

![](http://i.imgur.com/KOnQZhC.jpg)


Concave Hull - Alpha shapes

```Python

triangles = list(polygonize(m))

# sauver triangles

schema = {'geometry': 'Polygon','properties': {'test': 'int'}}
with fiona.open('triangles.shp','w','ESRI Shapefile', schema) as e:
      for geom in triangles:
          e.write({'geometry':mapping(geom), 'properties':{'test':'Delaunay'}})
          
```


![](http://i.imgur.com/JUKrVmE.jpg) 


```python
schema = {'geometry': 'Polygon','properties': {'test': 'int'}}
with fiona.open('alpha_hull.shp','w','ESRI Shapefile', schema) as e:
          e.write({'geometry':mapping(cascaded_union(triangles)), 'properties':{'test':1}})
          
```

![](http://i.imgur.com/13dInJ0.jpg)

