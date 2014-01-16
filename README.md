Python alpha shape ou concave-hull d'un nuage de points
===============================

repris de [The fading shape of alpha](http://sgillies.net/blog/1155/the-fading-shape-of-alpha/) de Sean Gillies pour m'en rappeler:


<img src="http://i.imgur.com/yDfb5uv.jpg" width="40%" height="40%">

La première démarche utilise la solution de [python scipy Delaunay plotting point cloud](http://stackoverflow.com/questions/6537657/python-scipy-delaunay-plotting-point-cloud):

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

m = MultiLineString(edge_points)
````

Delaunay

```Python
schema = {'geometry': 'LineString','properties': {'test': 'str'}}
with fiona.open('delaunay.shp','w','ESRI Shapefile', schema) as e:
       e.write({'geometry':mapping(m), 'properties':{'test':'Delaunay'}})
```

<img src="http://i.imgur.com/qs1vEpt.jpg" width="40%" height="40%">


Transformation des triangles en polygones

```Python

triangles = list(polygonize(m))

# sauver triangles

schema = {'geometry': 'Polygon','properties': {'test': 'str'}}
with fiona.open('triangles.shp','w','ESRI Shapefile', schema) as e:
      for geom in triangles:
          e.write({'geometry':mapping(geom), 'properties':{'test':'Triangle'}})
          
```


Alpha shape mais le résultat est un convex-hull classique:


```python
schema = {'geometry': 'Polygon','properties': {'test': 'str'}}
with fiona.open('alpha_hull.shp','w','ESRI Shapefile', schema) as e:
          e.write({'geometry':mapping(cascaded_union(triangles)), 'properties':{'test':'triangles'}})
          
```

<img src="http://i.imgur.com/hPrOAxh.jpgwidth" width="40%" height="40%">


La deuxième  démarche utilise un seuil de distance:

```python

for ia, ib, ic in tri.vertices:
    pa = points[ia]
    pb = points[ib]
    pc = points[ic]
    # Lengths of sides of triangle
    a = math.sqrt((pa[0]-pb[0])**2 + (pa[1]-pb[1])**2)
    b = math.sqrt((pb[0]-pc[0])**2 + (pb[1]-pc[1])**2)
    c = math.sqrt((pc[0]-pa[0])**2 + (pc[1]-pa[1])**2)
    # Semiperimeter of triangle
    s = (a + b + c)/2.0
    # Area of triangle by Heron's formula
    area = math.sqrt(s*(s-a)*(s-b)*(s-c))
    circum_r = a*b*c/(4.0*area)
    print circum_r
    173.793885591
903.491316516
194.990808947
240.715310408
985.479481529
848.094721523
673.836917324
1172.76572127
1550.26722128
1307.65503066
1119.77895414
461.450660891
427.046556743
385.025072963
633.505657258
407.60398854
247.385948175
213.433489426
251.220470997
560.76543119
416.546935089
595.027755236
1133.00687802
1084.47653221
1137.26084438
1057.56354056
271.08415096
234.276595336
244.437545434
551.211727584
274.632363417
259.634763839
261.264726795
557.615961547
274.292148254
284.765631262
175.603194785
280.895023228
374.093346275
350.696893868
308.606816609
238.25061405
247.850355165
257.085812573
234.474827487
347.426832156
346.595723189
298.657886428
307.345841561
325.652170928
321.247479543
364.905863109
415.531836635
278.20202359
8576.49630217
484.767247865
437.72075359
447.149883351
439.16524218
838.392822021
571.42421605
212.24558765
252.706511448
250.955134789
490.085165336
260.025220439
350.944920473
305.437875767
305.565783303
296.827029739
367.182749057
366.217070714
190.485397437
262.241091905
135544.923778
4095.89006484
298.381959363
255.292740419
267.017914621
241.470784634
244.367724265
170.022109522
1278.46788535
```

Il n'y a que quelques valeurs qui dénotent > 1000

Sean Gillies utilise un seuil basé sur la surface du triangle

"All of these are radii on the order of 1 except for 2 outliers. Those outliers are the narrow triangles we want to exclude from our hull. Setting a radius limit of 2 (α=0.5) via the code below removes the outlier triangles"

ces valeurs sont en effet situées autour de 1, sauf 2 valeurs. Mon problème que je ne pige pas encoree, c'est son test 

```python
if circum_r < 1.0/alpha:
```

j'ai choisi dans mon cas:

```python
if circum_r < alpha:
```

Avec un seuil de 1000:

```python
edges = set()
edge_points = []
alpha = 1000
# loop over triangles:
# ia, ib, ic = indices of corner points of the triangle
for ia, ib, ic in tri.vertices:
    pa = points[ia]
    pb = points[ib]
    pc = points[ic]
    # Lengths of sides of triangle
    a = math.sqrt((pa[0]-pb[0])**2 + (pa[1]-pb[1])**2)
    b = math.sqrt((pb[0]-pc[0])**2 + (pb[1]-pc[1])**2)
    c = math.sqrt((pc[0]-pa[0])**2 + (pc[1]-pa[1])**2)
    # Semiperimeter of triangle
    s = (a + b + c)/2.0
    # Area of triangle by Heron's formula
    area = math.sqrt(s*(s-a)*(s-b)*(s-c))
    circum_r = a*b*c/(4.0*area)
    # Here's the radius filter.
    #if circum_r < 1.0/alpha:
    if circum_r < alpha:
        add_edge(ia, ib)
        add_edge(ib, ic)
        add_edge(ic, ia)
        
```


<img src="http://i.imgur.com/UmiBiXs.jpg" width="40%" height="40%">

Il faut donc jouer sur ce seuil
