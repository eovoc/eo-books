---
jupytext:
  text_representation:
    extension: .md
    format_name: myst
    format_version: 0.13
    jupytext_version: 1.13.8
kernelspec:
  display_name: Python 3 (ipykernel)
  language: python
  name: python3
---

# OpenSearch with Atom

+++

<a name='Overview'></a>     
## Overview  
 The subjects covered in this Notebook are:    
 * [Colllection Search](#collection-search) 
   * [Access API Description](#access-api-description)
   * [Search by free text](#search-by-free-text)
   * [Search by title](#search-by-title)
   * [Search by free text](#search-by-platform)
   * [Search by instrument](#search-by-instrument)
   * [Search by organisation](#search-by-organisation)
   * [Search by identifier](#search-by-identifier)
   * [Search by concept](#search-by-concept)
 * [Colllection Properties](#collection-properties) 
   * [Collection geometry](#collection-geometry) 
   * [Collection temporal extent](#collection-temporal-extent) 
   * [Collection identifier](#collection-identifier) 
   * [Collection keywords](#collection-keywords) 
   * [Collection other representations](#collection-other-representations) 
   * [Collection related documentation](#collection-related-documentation) 
 * [Granule Search](#granule-search) 
   * [Access API Description](#granule-access-api-description)
   * [Search by bounding box](#granule-search-by-bounding-box)
   * [Search by geometry](#granule-search-by-geometry)
   * [Search by temporal extent](#granule-search-by-temporal-extent)
   * [Search by identifier](#granule-search-by-identifier)
 * [Granule Properties](#granule-properties) 
   * [Geometry](#geometry) 
   * [Temporal extent](#temporal-extent) 
   * [Granule identifier](#granule-identifier) 
   * [Quicklook](#quicklook)
   * [Granule download](#granule-download)
   * [Other representations](#other-representations) 
   * [Embedding additional properties](#embedding-additional-properties) 
 * [Advanced Topics](#advanced-topics)
   * [Result paging](#result-paging)
   * [Sorting results](#sorting-results)
   * [Faceted search](#faceted-search)
   * [Content negotiation](#content-negotiation)
 * [Further Reading](#further-reading) 

```{code-cell} ipython3
:tags: [hide_input]
:tags: [remove-cell]

#:tags: [remove-cell]
import json, requests, xml
import pandas as pd
from xml.dom import minidom
from IPython.display import Image

from xml.etree import ElementTree
import ipywidgets as widgets

from IPython.display import HTML
from IPython.display import Markdown as md

import re

# %pip install ipyleaflet
from ipyleaflet import (Map, GeoData, basemaps, WidgetControl, GeoJSON, Polygon,
   LayersControl, Icon, Marker, basemap_to_tiles, Choropleth,
   MarkerCluster, Heatmap, SearchControl, 
   FullScreenControl)

# from lxml import etree

# Select the top-level OSDD from which the user will be able to choose.
url_osdd_choices = [ 
    'https://fedeo.ceos.org' + '/opensearch/description.xml' ,
    'https://eocat.esa.int' + '/opensearch/description.xml' ,
    'https://eovoc.spacebel.be' + '/api?httpAccept=application%2Fopensearchdescription%2Bxml' ,  # original
   # 'https://eovoc.spacebel.be' + '/api?httpAccept=application/opensearchdescription+xml' ,
   # 'https://eovoc.spacebel.be' + '/api' ,
    
    'https://geo.spacebel.be' + '/opensearch/description.xml' ]

url_explain_choices = [ 
    'https://fedeo.ceos.org' + '/opensearch/request' ,
    'https://eocat.esa.int' + '/opensearch/request' ,
    'https://eovoc.spacebel.be' + '/api?httpAccept=application/sru%2Bxml' ,
    'https://geo.spacebel.be' + '/opensearch/request' ]


# url_explain = 'https://fedeo.ceos.org' + '/opensearch/request'
# url_explain = 'https://eovoc.spacebel.be' + '/api?httpAccept=application/sru%2Bxml'

# Verification of SSL certificate is to be set to False for the eocat endpoint to work.
# verify_ssl = False
verify_ssl = True
```

```{code-cell} ipython3
:tags: [hide_input]
:tags: [remove-cell]

#:tags: [remove-cell]
def load_dataframe( resp ):
  
  df = pd.DataFrame(columns=['dc:identifier', 'atom:title', 'atom:updated', 'atom:link[rel="search"]', 'atom:link[rel="enclosure"]', 'atom:link[rel="icon"]'])

  rt = ElementTree.fromstring(response.text)
  for r in rt.findall('{http://www.w3.org/2005/Atom}entry'):
     name = r.find('{http://purl.org/dc/elements/1.1/}identifier').text
     title = r.find('{http://www.w3.org/2005/Atom}title').text
     updated = r.find('{http://www.w3.org/2005/Atom}updated').text
     dcdate = r.find('{http://purl.org/dc/elements/1.1/}date').text
     # print('collection',count,'-', name, ':')
     # print('\tidentifier: ',name)
     
     try:
         href = r.find('{http://www.w3.org/2005/Atom}link[@rel="search"][@type="application/opensearchdescription+xml"]').attrib['href']
     except AttributeError:
         href= ''

     try:
         rel_enclosure = r.find('{http://www.w3.org/2005/Atom}link[@rel="enclosure"]').attrib['href']
     except AttributeError:
         rel_enclosure= ''

     try:
         rel_icon = r.find('{http://www.w3.org/2005/Atom}link[@rel="icon"]').attrib['href']
     except AttributeError:
         rel_icon= ''

     # append a row to the df 
     new_row = { 'dc:identifier': name, 'atom:title': title, 'dc:date': dcdate, 'atom:updated': updated, 'atom:link[rel="search"]': href, 
        'atom:link[rel="enclosure"]': rel_enclosure , 'atom:link[rel="icon"]': rel_icon}
     df = df.append(new_row, ignore_index=True)

  return df

def load_facet( root, facet ):
    
  el = root.find('.//{http://a9.com/-/opensearch/extensions/sru/2.0/}facet[{http://a9.com/-/opensearch/extensions/sru/2.0/}index="' + facet + '"]')
  # el = root.find('.//{http://a9.com/-/opensearch/extensions/sru/2.0/}facet[{http://a9.com/-/opensearch/extensions/sru/2.0/}index="eo:instrument"]')
  
  df = pd.DataFrame(columns=['name', 'count'])

  for r in el.findall('.//{http://a9.com/-/opensearch/extensions/sru/2.0/}term'):
     name = r.find('{*}actualTerm').text
     count = r.find('{*}count').text
    
     # append a row to the df 
     new_row = { 'name': name, 'count': int(count) }
     df = df.append(new_row, ignore_index=True)

  df.set_index('name', inplace=True)
  return df


def show_features_on_map( georss_box, georss_polygon ):
  # data = query_response.json()
  # print (georss_polygon)
  list1 = georss_polygon.split()
  list2 = georss_box.split()
      
  points = []    
  for i in range(0,len(list1),2):
      # print (list2[i], list2[i+1])
      points = points + [ (float(list1[i]), float(list1[i+1])) ]
      
  # use center of the bounding box.
  
  try:
         center = [ (float(list2[0])+float(list2[2]))/2.0 , (float(list2[1])+float(list2[3]))/2.0 ]  
  except:
         # default center and zoom factor for an empty map
         center = [50.85, 4.3488]
         zoom = 3
         
  # center = [ float(list1[0]), float(list1[1]) ]  
  # print ("center", center)
  
  # center = [43, -49]
  # zoom = 3
  m = Map(basemap=basemaps.OpenStreetMap.Mapnik, center=center)
  # m = Map(basemap=basemaps.OpenStreetMap.Mapnik, center=center, zoom=zoom)
  # geolayer = GeoJSON(
  #  name = 'Features',
  #  data=data,
  #  style={ 'opacity': 1, 'dashArray': '9', 'fillOpacity': 0.7, 'weight': 1 },
  #  hover_style={ 'color': 'white', 'dashArray': '0', 'fillOpacity': 0.5 }
  # )
  # m.add_layer(geolayer)
    
  polygon = Polygon(locations=points, color="green", fill_color="green")
  m.add_layer(polygon)
  
  m.add_control(FullScreenControl())
  m.add_control(LayersControl(position='topright'))
  
  return m
```

```{code-cell} ipython3
:tags: [remove-cell]

#:tags: [remove-cell]

import re;

def get_api_request(template, os_querystring):
  # Fill (URL) template with OpenSearch parameter values provided in os_querystring and return as short HTTP URL without empty parameters.
  
  print("URL template: " + template)
      
  # perform substitutions in template
  for p in os_querystring:
      print("  .. replacing:", p, "by", os_querystring[p])
      template = re.sub('\{'+p+'.*?\}', os_querystring[p] , template)
      # print("- intermediate new template:" + template)
      
  # remove empty search parameters
  template=re.sub('&?[a-zA-Z]*=\{.*?\}', '' , template)
  # print("- Shorter template step 1: " + template)
  
  # remove remaining empty search parameters which did not have an HTTP query parameter attached (e.g. time:end).
  template=re.sub('\{.*?\}', '' , template)
  
  print("API request: " + template)
            
  return (template)
```

The Notebook can be used with a number of different endpoints.  Select the OSDD to be used for collection search from the list below.

```{code-cell} ipython3
:tags: [hide_input]
:tags: [remove-cell]

#:tags: [remove-cell]
list = widgets.Dropdown(options=url_osdd_choices, description="Select OSDD", index=0)

# def on_change(change):
#    if change['type'] == 'change' and change['name'] == 'value':
#        print("changed to %s" % change['new'])

# list.observe(on_change)

list
```

```{code-cell} ipython3
:tags: [hide_input]
:tags: [remove-input]

#:tags: [remove-input]
# Get the selected OSDD endpoint from the list.
url_osdd = list.value
# select the corresponding url_explain accoridng to the list selection
url_explain = url_explain_choices[list.index]

# remove next line
# url_osdd = url_osdd_choices[1]

url_osdd
```

##  Collection Search

+++

### Access API Description

```{code-cell} ipython3
:tags: [hide_input]
:tags: [remove-input]

#:tags: [remove-input]
md("The OpenSearch Description Document is accessible at the fixed location {} and contains the URL template to be used for collection search.".format(url_osdd))
```

**Example: 2.1**  
>  Access the API Description in OpenSearch Description Document (OSDD) format.  

```{code-cell} ipython3
:tags: [output_scroll]

# response = requests.get(url_osdd, verify=bool(verify_ssl), headers={'Accept': 'application/opensearchdescription+xml'})
# above does not work for eovoc (from inside Spacebel)
response = requests.get(url_osdd, verify=bool(verify_ssl) )

xmlstr = minidom.parseString(response.text).toprettyxml(indent='  ',newl='')
md("```xml\n" + xmlstr + "\n```\n")
```

```{code-cell} ipython3
:tags: [output_scroll]

```

```{code-cell} ipython3
:tags: [remove-input]

#:tags: [remove-input]
md("The Explain Document is accessible at the location {} and contains additional information about the API such as default values, definitions of available record schemas at `/explain/responseFormats` etc.".format(url_explain))
```

<a name='selecting-mediatype-in-osdd'></a>   
**Example: 2.2**  
>  Extract collection search URL temmplate for Atom  

Extract the URL template for collection search `rel="collection"` corresponding to the media type `type="application/atom+xml"` of the search result.

```{code-cell} ipython3
:tags: [output_scroll]

from xml.etree import ElementTree
root = ElementTree.fromstring(response.text)

collection_url_atom = root.find('{http://a9.com/-/spec/opensearch/1.1/}Url[@rel="collection"][@type="application/atom+xml"]')

template = collection_url_atom.attrib['template']
collection_template = template
print('Collection search template: ', template)


# Use the fixed part and first query parameter as endpoint
# x = collection_url_atom.attrib['template'].split("?")
# endpoint = x[0]
# print('Collection search endpoint: ', endpoint)
```



+++


### Search by free text

+++

**Example: 2.3**  
>  Search collections by free text {searchTerms}  

```{code-cell} ipython3
:tags: [output_scroll]

osquerystring = {}
osquerystring['count'] = '1'
osquerystring['searchTerms'] = 'forestry'

request_url = get_api_request(template, osquerystring)

response = requests.get(request_url, verify=bool(verify_ssl))
xmlstr = minidom.parseString(response.text).toprettyxml(indent='   ', newl='')
md("```xml\n" + xmlstr + "\n```\n")
```

<a name='Collection-Search-by-Title'></a>     
### Search by title

+++

**Example: 2.4**  
>  Search collections by title {dc:title}  

```{code-cell} ipython3
:tags: [output_scroll]

osquerystring = {}
osquerystring['dc:title'] = 'Column'

request_url = get_api_request(template, osquerystring)
response = requests.get(request_url, verify=bool(verify_ssl))

rt = ElementTree.fromstring(response.text)
# Get title of all entries in result page
for r in rt.findall('{http://www.w3.org/2005/Atom}entry'):
     title = r.find('{http://www.w3.org/2005/Atom}title').text
     print("title: ", title)
```

<a name='Collection-Search-by-Platform'></a>     
### Search by platform

+++

The `<url>` element contains additional information about the parameters available in the template using the OpenSearch Parameter extension syntax.  For example, the `eo:platform` parameter provides the following additional information, including the list of possible values for which the server can provide results.

+++

**Example: 2.5**  
>  Extract available values for the `eo:platform` parameter from the OSDD.  

```{code-cell} ipython3
:tags: [output_scroll]

# Extract <Parameter> element for eo:platform
el = collection_url_atom.find('{http://a9.com/-/spec/opensearch/extensions/parameters/1.0/}Parameter[@value="{eo:platform}"]')

# el2 = ElementTree.indent(el)
# https://docs.python.org/3/library/xml.etree.elementtree.html
xmltxt = ElementTree.tostring(el, encoding='unicode', method='xml')
md("```xml\n" + xmltxt + "\n```\n")

# todo: add output scrolling tag to the cell output tag: "output_scroll"
# insert Markdown string in an HTML frame with scrollbar ?
```

**Example: 2.6**  
>  Search collections by platform {eo:platform} [[RD3]](#RD3). 

Search parameters which are optional can be skipped in the search template or their value can be left empty.
Prepare a search request by replacing all mandatory search parameters with a value.  

By default, each `<atom:entry>` represents one search result (an EO collection) and the search response contains faceted search results under the element `<sru:facetedResults>`.  The faceted search information groups the results by `platform`, by `instrument`, by `organisation` etc.  The original metadata for the collection is not embedded in the response but available as an `atom:link`.

```{code-cell} ipython3
:tags: [output_scroll]

osquerystring = {}
osquerystring['eo:platform'] = 'proba-1'
osquerystring['count'] = '2'

request_url = get_api_request(template, osquerystring)
response = requests.get(request_url, verify=bool(verify_ssl))

xmlstr = minidom.parseString(response.text).toprettyxml(indent='   ', newl='')
md("```xml\n" + xmlstr + "\n```\n")
```

The above response indicates the number of results (`totalResults`) and contains information about a number of collections including a link to the OSDD document to use for granule search.  If more than TBD records are found, then the results are returned in pages and paging links are included to navigate to the next results (`rel=next`).

```{code-cell} ipython3
:tags: [output_scroll]

root = ElementTree.fromstring(response.text)

# extract total results
el = root.find('{http://a9.com/-/spec/opensearch/1.1/}totalResults')
print('totalResults: ', el.text)

dataframe = load_dataframe(response)
dataframe.head(20)

url_osdd_granules = dataframe.iat[0,3]
print(url_osdd_granules)
```

**Example: 2.7**  
>  Obtain allowed values for {sru:recordSchema} from the OSDD.   

 
The OSDD template lists the `sur:recordSchema` values that can be used in a collection search request.  They correspond to metadata formats that can be directly embedded in the Search response.  The value `server-choice` can be used to allow the server to propose an appropriate metadata encoding.

The shortnames used for some of the recordSchemas are for backward compatibility and they are explained in the Explain document. 

```{code-cell} ipython3
:tags: [output_scroll]

# Extract <Parameter> element for sru:recordSchema
el = collection_url_atom.find('{http://a9.com/-/spec/opensearch/extensions/parameters/1.0/}Parameter[@value="{sru:recordSchema}"]')
xmltxt = ElementTree.tostring(el, encoding='unicode', method='xml')
md("```xml\n" + xmltxt + "\n```\n")
```

**Example: 2.8**  
>  Use of parameters {sru:recordSchema} and facetLimit.   


The following request is similar to the one above, but requests to embed the complete collection metadata inside the search response (`recordSchema=server-choice`) and disables the faceted search information (`facetLimit=0`).

```{code-cell} ipython3
:tags: [output_scroll]

osquerystring = {}
osquerystring['eo:platform'] = 'proba-1'
osquerystring['sru:recordSchema'] = 'server-choice'
# issue: sru:facteLimit is not advertised in OSDD.
osquerystring['sru:facetLimit'] = '0'

request_url = get_api_request(template, osquerystring)

response = requests.get(request_url, verify=bool(verify_ssl))

xmlstr = minidom.parseString(response.text).toprettyxml(indent='   ',newl='')
md("```xml\n" + xmlstr + "\n```\n")
```

<a name='Collection-Search-by-Instrument'></a>     
### Search by instrument

+++

**Example: 2.9**  
>  Search collections by instrument {eo:instrument} [[RD3]](#RD3). 

```{code-cell} ipython3
:tags: [output_scroll]

osquerystring = {}
osquerystring['eo:instrument'] = 'SAR'

request_url = get_api_request(template, osquerystring)
response = requests.get(request_url, verify=bool(verify_ssl))

rt = ElementTree.fromstring(response.text)
# Get title of all entries in result page
for r in rt.findall('{http://www.w3.org/2005/Atom}entry'):
     title = r.find('{http://www.w3.org/2005/Atom}title').text
     print("title: ", title)
```

<a name='Collection-Search-by-Organisation'></a>     
### Search by organisation

+++

**Example: 2.10**  
>  Search collections by organisation {eo:organisationName} [[RD3]](#RD3). 

TBD: check topics to be covered...
TBD: faceted search
TBD: script to make output scrollable ?

```{code-cell} ipython3
:tags: [output_scroll]

osquerystring = {}
osquerystring['eo:organisationName'] = 'ESA/ESRIN'

request_url = get_api_request(template, osquerystring)
response = requests.get(request_url, verify=bool(verify_ssl))

rt = ElementTree.fromstring(response.text)
# Get title of all entries in result page
for r in rt.findall('{http://www.w3.org/2005/Atom}entry'):
     title = r.find('{http://www.w3.org/2005/Atom}title').text
     print("title: ", title)
```

<a name='Collection-Search-by-Identifier'></a>     
### Search by identifier

+++

**Example: 2.11**  
>  Search collections by identifier {geo:uid} [[RD2]](#RD2). 

The `geo:uid` optionally combined with a subcatalogue identifier `eo:parentIdentifier` allows retrieving collection metadata for a specific collection.

```{code-cell} ipython3
:tags: [output_scroll]

osquerystring = {}
osquerystring['geo:uid'] = 'PROBA.HRC.1A' 

request_url = get_api_request(template, osquerystring)
response = requests.get(request_url, verify=bool(verify_ssl))

xmlstr = minidom.parseString(response.text).toprettyxml(indent='   ')
print(xmlstr)
```

### Search by concept

+++

**Example: 2.12**  
>  Search collections by concept URI {semantic:classifiedAs}  - TBD: not working on OVH ?

Collection metadata includes platform, instrument and science keywords, including the URI of these concepts expressed in the ESA Thesauri (https://thesauri.spacebel.be/) and NASA GCMD thesauri.  The URI of these concepts can be used as search parameter.  

In the current version of the software, the following concept URI are supported:

* GCMD thesaurus science keyword URI
* ESA thesaurus platform URI
* ESA thesaurus instrument URI

Future versions of the software derived from the EOVOC developments may support both GCMD and ESA URI for all three categories in addition to GEMET, INSPIRE Themes, Dbpedia, Wikidata and other URI.

```{code-cell} ipython3
:tags: [output_scroll]

osquerystring = {}
# Envisat concept (https://thesauri.spacebel.be/).
# querystring['classifiedAs'] = 'https://earth.esa.int/concept/11ea961b-1d0b-5d6d-a55a-b58aed01d430'
# ENVISAT concept NASA GCMD
# osquerystring['semantic:classifiedAs'] = 'https://gcmd.earthdata.nasa.gov/kms/concept/a1498dff-002d-4d67-9091-16822c608221'
# osquerystring['eo:parentIdentifier'] = 'EOP:ESA:EARTH-ONLINE'
# Proba-1 concept NASA GCMD
# osquerystring['semantic:classifiedAs'] = 'https://gcmd.earthdata.nasa.gov/kms/concept/fe4a4604-029e-4cdc-93f0-6d8799dd25e5'
# Proba-1 concept in ESA thesaurus
osquerystring['semantic:classifiedAs'] = 'https://earth.esa.int/concept/b3979ff2-d27d-5f22-9e06-a18c5759d9a5'

request_url = get_api_request(template, osquerystring)
response = requests.get(request_url, verify=bool(verify_ssl))

dataframe = load_dataframe(response)
dataframe.head(20)
```

## Collection properties

```{code-cell} ipython3
:tags: [output_scroll]

rt = ElementTree.fromstring(response.text)
r = rt.find('{http://www.w3.org/2005/Atom}entry')  # return first entry
```

### Collection geometry

Geometry information for each collection is included in the Atom entry using GeoRSS response elements.

```{code-cell} ipython3
:tags: [output_scroll]

try:
    box = r.find('{http://www.georss.org/georss}box').text
except AttributeError:
    box= ''

try:
    polygon = r.find('{http://www.georss.org/georss}polygon').text
except AttributeError:
    polygon= ''

print("georss:box:", box )
print("georss:polygon:", polygon )

```

```{code-cell} ipython3
:tags: [remove-input]

#:tags: [remove-input]
show_features_on_map(box, polygon)
```

### Collection temporal extent

The `<dc:date>` response element provides temporal information for a collection, i.e. the start time and end time separated by a `/`, encoded as per [RFC-3339](https://www.rfc-editor.org/rfc/rfc3339.txt).  The end time may be absent indicating that the collection is not completed.

```{code-cell} ipython3
:tags: [output_scroll]

try:
    date = r.find('{http://purl.org/dc/elements/1.1/}date').text
except AttributeError:
    date= ''

date
```

### Collection identifier

The `<dc:identifier>` response element includes the idenfifier of the collection that can be used as value for the `geo:uid` search parameter.

```{code-cell} ipython3
:tags: [output_scroll]

try:
    id = r.find('{http://purl.org/dc/elements/1.1/}identifier').text
except AttributeError:
    id= ''

id
```

### Collection keywords

The optional `<atom:category>` response elements provide keywords related to the collection.  Keywords can be free text keywords or originate from a controlled thesaurus.  The `term` attribute is used to hold the full concept URI (if available) as per [[RD10]](#RD10).  
When keywords provide a concept URI, then this URI can be used to search for collections by concept with the `semantic:classifiedAs` search parameter.

```{code-cell} ipython3
:tags: [output_scroll]

# build table with extracted keywords
list = pd.DataFrame(columns=['label', 'term'])
for lnk in r.findall('{http://www.w3.org/2005/Atom}category'):
    label = ''
    term = ''
    try:
        label = lnk.attrib['label']
        term = lnk.attrib['term']
    except:
        pass
    list = list.append( { 'label': label, 'term': term }, ignore_index=True )

#HTML(altList.to_html(render_links=True, escape=False))
list
```

### Collection other representations

Alternative metadata formats for the collection represented by the Atom entry are available as `<atom:link>` with `rel="alternate"`.  Different servers may advertize different metadata formats.

```{code-cell} ipython3
:tags: [output_scroll]

# build table with rel=alternate links
altList = pd.DataFrame(columns=['title', 'type', 'href'])
for lnk in r.findall('{http://www.w3.org/2005/Atom}link[@rel="alternate"]'):
    altList = altList.append( { 'type': lnk.attrib['type'], 'title': lnk.attrib['title'], 'href': lnk.attrib['href'] }, ignore_index=True )

#HTML(altList.to_html(render_links=True, escape=False))
altList
```

### Collection related documentation

Collections can optionaly provide access to related documentation via an `<atom:link>` with `rel="desribedby"`.

```{code-cell} ipython3
:tags: [output_scroll]

# build table with rel=describedby links
relList = pd.DataFrame(columns=['title', 'type', 'href'])
for lnk in r.findall('{http://www.w3.org/2005/Atom}link[@rel="describedby"]'):
    relList = relList.append( { 'type': lnk.attrib['type'], 'title': lnk.attrib['title'], 'href': lnk.attrib['href'] }, ignore_index=True )

# HTML(relList.to_html(render_links=True, escape=False))
relList
```

## Granule Search   

+++

<a name='granule-access-api-description'></a>    
### Access API Description

```{code-cell} ipython3
:tags: [remove-input]

#:tags: [remove-input]
f"The OpenSearch Description Document is accessible at the location {url_osdd_granules} which is extracted from the collection search response and contains the URL template to be used for granule search."
```

**Example: 4.1**  
>  Obtain API Description (OSDD) for the collection via the URL fouind in collection search response.   

```{code-cell} ipython3
:tags: [output_scroll]

response = requests.get(url_osdd_granules, verify=bool(verify_ssl), headers={'Accept': 'application/opensearchdescription+xml'})

xmlstr = minidom.parseString(response.text).toprettyxml(indent='  ', newl='')
md("```xml\n" + xmlstr + "\n```\n")
# print(xmlstr)
```

Extract the URL template for collection search `rel="results"` corresponding to the media type `type="application/atom+xml"` of the search result.

```{code-cell} ipython3
:tags: [output_scroll]

root = ElementTree.fromstring(response.text)

granules_url_atom = root.find('{http://a9.com/-/spec/opensearch/1.1/}Url[@rel="results"][@type="application/atom+xml"]')

print('Granule search template: ', granules_url_atom.attrib['template'])

# create the url without any parameters, preserving fix parts.
template = granules_url_atom.attrib['template']
print('Granule search template: ', template)
```

<a name='granule-find-available-record-schemas'></a>    
### Find available record schemas

+++

**Example: 4.2**  
>  Obtain allowed values for {sru:recordSchema} from the OSDD.   

 
The OSDD template lists the `sru:recordSchema` values that can be used in a granule search request.  They correspond to metadata formats that can be directly embedded in the Search response.  The value `server-choice` can be used to allow the server to propose an appropriate metadata encoding.

```{code-cell} ipython3
:tags: [output_scroll]

# Extract <Parameter> element for sru:recordSchema
el = granules_url_atom.find('{http://a9.com/-/spec/opensearch/extensions/parameters/1.0/}Parameter[@value="{sru:recordSchema}"]')
xmltxt = ElementTree.tostring(el, encoding='unicode', method='xml')
md("```xml\n" + xmltxt + "\n```\n")
```

<a name='granule-search-by-bounding-box'></a>    
### Search by bounding box

+++

**Example: 4.3**  
>  Search granules by bounding box {geo:box} [[RD2]](#RD2).

Search parameters which are optional in the search template (end with a "?") can be omitted or their value can be left empty.
Prepare a search request by replacing all mandatory search parameters with a value.  

By default, each `<atom:entry>` represents one search result (an EO granule).  The original metadata for the collection is not embedded in the response but available as an `atom:link`.

```{code-cell} ipython3
:tags: [output_scroll]

osquerystring = {}
osquerystring['count'] = '3'
osquerystring['geo:box'] = '14.90,37.700,14.99,37.780' 

request_url = get_api_request(template, osquerystring)
response = requests.get(request_url, verify=bool(verify_ssl))

# xmlstr = minidom.parseString(response.text).toprettyxml(indent='   ')
# print(xmlstr)

root = ElementTree.fromstring(response.text)

# extract total results
el = root.find('{http://a9.com/-/spec/opensearch/1.1/}totalResults')
print('totalResults: ', el.text)

dataframe = load_dataframe(response)
dataframe.head(20)

```

```{code-cell} ipython3
:tags: [output_scroll]

# extract a value

print(dataframe.loc[0]['dc:identifier'])
```

```{code-cell} ipython3
:tags: [output_scroll]

# display the table with clickable hyperlinks.

HTML(dataframe.transpose(copy=True).to_html(render_links=True, escape=False))
```

<a name='granule-search-by-geometry'></a>    
### Search by geometry

Collections may advertise the availability of the optional `geo:geometry` [[RD2]](#RD2) search parameter in the collection OSDD.

+++

**Example: 4.4**  
>  Obtain profiles for {geo:geometry} from the OSDD.   

 
If the parameter is supported for the collection, then the OSDD template identifies one or more profiles of the `geo:geometry` values that can be used in a granule search request.  Possible profiles include searches by `point`, `linestring`, `multipoint`, `multilinestring` or `polygon`.  In all cases, the geometry value is to be provided in Well-Known Text [(WKT)](https://en.wikipedia.org/wiki/Well-known_text_representation_of_geometry) format.

```{code-cell} ipython3
:tags: [output_scroll]

# Extract <Parameter> element for geo:geometry
el = granules_url_atom.find('{http://a9.com/-/spec/opensearch/extensions/parameters/1.0/}Parameter[@value="{geo:geometry}"]')
xmltxt = ElementTree.tostring(el, encoding='unicode', method='xml')
md("```xml\n" + xmltxt + "\n```\n")
```

**Example: 4.5**  
>  Search granules by polygon geometry {geo:geometry} [[RD2]](#RD2).

```{code-cell} ipython3
:tags: [output_scroll]

osquerystring = {}
osquerystring['count'] = '3'
osquerystring['geo:geometry'] = 'POLYGON((14.90 37.700, 14.90 37.780, 14.99 37.780, 14.99 37.700, 14.90 37.700))' 

request_url = get_api_request(template, osquerystring)
response = requests.get(request_url, verify=bool(verify_ssl))

root = ElementTree.fromstring(response.text)

# extract total results
el = root.find('{http://a9.com/-/spec/opensearch/1.1/}totalResults')
print('totalResults: ', el.text)

dataframe = load_dataframe(response)
dataframe.head(20)
# HTML(dataframe.transpose(copy=True).to_html(render_links=True, escape=False))
```

<a name='granule-search-by-temporal-extent'></a>    
### Search by temporal extent

+++

**Example: 4.6**  
>  Search granules by temporal extent {time:start} and {time:end} [[RD2]](#RD2).  Each granule has an acquisition start time and end time.  A granule is returned if the intersection of the temporal search interval with the acquisition start/end interval is not empty.  

```{code-cell} ipython3
:tags: [output_scroll]

osquerystring = {}
osquerystring['count'] = '3'
osquerystring['geo:box'] = '14.90,37.700,14.99,37.780' 
osquerystring['time:start'] = '2021-01-01T00:00:00Z'
osquerystring['time:end'] = '2021-03-31T23:59:59Z'

request_url = get_api_request(template, osquerystring)
response = requests.get(request_url, verify=bool(verify_ssl))

# xmlstr = minidom.parseString(response.text).toprettyxml(indent='   ')
# print(xmlstr)

root = ElementTree.fromstring(response.text)
# extract total results
el = root.find('{http://a9.com/-/spec/opensearch/1.1/}totalResults')
print('totalResults: ', el.text)

dataframe = load_dataframe(response)
# dataframe.head(20)
HTML(dataframe.transpose(copy=True).to_html(render_links=True, escape=False))
```

<a name='granule-search-by-identifier'></a>     
### Search by identifier

+++

**Example: 4.7**  
>  Search granules by identifier {geo:uid} [[RD2]](#RD2). 

The `geo:uid` combined with the collection identifier `eo:parentIdentifier` which is already prefilled in the OSDD template extracted from the collection search response allows retrieving granule metadata for a specific granule.

```{code-cell} ipython3
:tags: [output_scroll]

osquerystring = {}
osquerystring['geo:uid'] = 'PR1_OPER_HRC_HRC_1P_20210401T131711_N37-069_E014-099_0001' 

request_url = get_api_request(template, osquerystring)
response = requests.get(request_url, verify=bool(verify_ssl))

xmlstr = minidom.parseString(response.text).toprettyxml(indent='   ')
print(xmlstr)
```

## Granule properties

```{code-cell} ipython3
:tags: [output_scroll]

# get the first entry (granule) from the previous search results;
rt = ElementTree.fromstring(response.text)
r = rt.find('{http://www.w3.org/2005/Atom}entry')  
```

### Geometry

Geometry information for each granule is included in the Atom entry using GeoRSS encoding.

```{code-cell} ipython3
:tags: [output_scroll]

try:
    box = r.find('{http://www.georss.org/georss}box').text
except AttributeError:
    box= ''

try:
    polygon = r.find('{http://www.georss.org/georss}polygon').text
except AttributeError:
    polygon= ''

print("georss:box:", box )
print("georss:polygon:", polygon )
```

```{code-cell} ipython3
:tags: [remove-input]

#:tags: [remove-input]
show_features_on_map(box, polygon)
```

### Temporal extent

The `<dc:date>` response element provides temporal information for each granule, i.e. the acquistion start time and end time, encoded as per [RFC-3339](https://www.rfc-editor.org/rfc/rfc3339.txt).

```{code-cell} ipython3
:tags: [output_scroll]

try:
    date = r.find('{http://purl.org/dc/elements/1.1/}date').text
except AttributeError:
    date= ''

date
```

### Granule identifier

The `<dc:identifier>` response element includes the idenfifier of the granule that can be used as value for the `geo:uid` search parameter.

```{code-cell} ipython3
:tags: [output_scroll]

try:
    id = r.find('{http://purl.org/dc/elements/1.1/}identifier').text
except AttributeError:
    id= ''

id
```

### Quicklook

The `atom:link` with rel="icon" can optionally provide access to a quicklook or browse image.

```{code-cell} ipython3
:tags: [output_scroll]

try:
    href = r.find('{http://www.w3.org/2005/Atom}link[@rel="icon"]').attrib['href']
except AttributeError:
    href= ''

print("Quicklook URL:", href )
Image(url=href, width=200, height=200)
```

### Granule download

The `atom:link` with rel="enclosure" provides access to the granule as a file download URL (if available).

```{code-cell} ipython3
:tags: [output_scroll]

try:
    href = r.find('{http://www.w3.org/2005/Atom}link[@rel="enclosure"]').attrib['href']
except AttributeError:
    href= ''
    
print("Granule download URL:", href )
```

### Other representations

Alternative metadata formats for the granule represented by the Atom entry are available as Atom links with rel="alternate".  Different servers may advertize different metadata formats.

```{code-cell} ipython3
:tags: [output_scroll]

# Present list of alternate links in table

altList = pd.DataFrame(columns=['title', 'type', 'href'])
for lnk in r.findall('{http://www.w3.org/2005/Atom}link[@rel="alternate"]'):
    altList = altList.append( { 'type': lnk.attrib['type'], 'title': lnk.attrib['title'], 'href': lnk.attrib['href'] }, ignore_index=True )

HTML(altList.to_html(render_links=True, escape=False))
```

### Embedding additional properties

Alternative metadata formats for the granule provide additional metadata properties and can be directly embedded in the Atom entry using the `sru:recordSchema` parameter.  The O&M format (OGC 10-157r4) provides the most detailed representation. 

+++

**Example: 4.9**  
>  Get list of supported record schemas {sru:recordSchema} from the OSDD.   

 
The OSDD template for the collection lists the `sur:recordSchema` values that can be used in a granule search request.  They correspond to metadata formats that can be directly embedded in the Search response.  The value `server-choice` can be used to allow the server to propose an appropriate metadata encoding.

```{code-cell} ipython3
:tags: [output_scroll]

# Extract corresponding <Parameter> element
el = granules_url_atom.find('{http://a9.com/-/spec/opensearch/extensions/parameters/1.0/}Parameter[@value="{sru:recordSchema}"]')
xmltxt = ElementTree.tostring(el, encoding='unicode', method='xml')
md("```xml\n" + xmltxt + "\n```\n")
```

**Example: 4.10**  
>  Embed O&M metadata in search response {sru:recordSchema} [[RD8]](#RD8).   

The additional properties are include in an `<eop:EarthObservation>` element inside the `<atom:entry>`.  Depending on the type of observation, the `EarthObservation` element may be in the `eop` (General), `opt` (Optical), `sar` (Radar), `atm` (Atmospheric) or other namespace defined by OGC 10-157r4.

```{code-cell} ipython3
:tags: [output_scroll]

osquerystring = {}
osquerystring['count'] = '1'
osquerystring['geo:box'] = '14.90,37.700,14.99,37.780' 
# osquerystring['sru:sortKeys'] = 'start,time,1'
osquerystring['sru:recordSchema'] = 'om'

request_url = get_api_request(template, osquerystring)
response = requests.get(request_url, verify=bool(verify_ssl))

rt = ElementTree.fromstring(response.text)
r = rt.find('{http://www.w3.org/2005/Atom}entry')  # return first entry

try:
    el = r.find('{*}EarthObservation')
    xmltxt = ElementTree.tostring(el, encoding='unicode', method='xml')
except AttributeError:
    xmltxt= 'Not found.'

md("```xml\n" + xmltxt + "\n```\n")
```

<a name='Advanced-Topics'></a>  
## Advanced topics

+++

### Result paging

Collection and granule search results are provided in pages.  Search responses contain `atom:link` navigation links providing access to tbe `first`, `last`, `previous` or `next` result pages.

+++

**Example: 6.1**  
>  Obtain navigation links from search response. 

```{code-cell} ipython3
:tags: [output_scroll]

osquerystring = {}
osquerystring['count'] = '10'

request_url = get_api_request(template, osquerystring)
response = requests.get(request_url, verify=bool(verify_ssl))

root = ElementTree.fromstring(response.text)
# extract total results
el = root.find('{http://a9.com/-/spec/opensearch/1.1/}totalResults')
print('totalResults: ', el.text)

# Present list of navigation links in table
relList = pd.DataFrame(columns=['rel', 'href'])
for lnk in root.findall('{http://www.w3.org/2005/Atom}link[@type="application/atom+xml"]'):
    if lnk.attrib['rel'] in  ['first', 'last', 'prev', 'next']:
      relList = relList.append( { 'rel': lnk.attrib['rel'], 'href': lnk.attrib['href'] }, ignore_index=True )

HTML(relList.to_html(render_links=True, escape=False))
```

**Example: 6.2**  
>  Traverse search results using `{startIndex}`.

The search response includes information about the `totalResults`, `itemsPerPage` and `startIndex`.

```{warning}
The use of large startIndex values is discouraged as the backend implementation is not optimized for larger values.  The implementation may return an HTTP error code when the value exceeds the maximum value allowed.  As a work around, you can narrow down your search by applying (additional) temporal or geographical search parameters thereby avoiding having to navigate very large result sets.
```

```{code-cell} ipython3
:tags: [output_scroll]

osquerystring = {}
osquerystring['count'] = '10'
osquerystring['startIndex'] = '11'

request_url = get_api_request(template, osquerystring)
response = requests.get(request_url, verify=bool(verify_ssl))

root = ElementTree.fromstring(response.text)

el = root.find('{http://a9.com/-/spec/opensearch/1.1/}totalResults')
print('totalResults: ', el.text)
el = root.find('{http://a9.com/-/spec/opensearch/1.1/}itemsPerPage')
print('itemsPerPage: ', el.text)
el = root.find('{http://a9.com/-/spec/opensearch/1.1/}startIndex')
print('startIndex: ', el.text)

# Present list of navigation links in table
relList = pd.DataFrame(columns=['rel', 'href'])
for lnk in root.findall('{http://www.w3.org/2005/Atom}link[@type="application/atom+xml"]'):
    if lnk.attrib['rel'] in  ['first', 'last', 'previous', 'next', 'prev']:
      relList = relList.append( {  'rel': lnk.attrib['rel'], 'href': lnk.attrib['href'] }, ignore_index=True )

HTML(relList.to_html(render_links=True, escape=False))
```

**Example: 6.3**  
>  Traverse collection search results using `{startPage}`.

`{startPage}` can be used as an alternative for `{startIndex}`.

```{code-cell} ipython3
:tags: [output_scroll]

osquerystring = {}
osquerystring['count'] = '10'
osquerystring['startPage'] = '2'

request_url = get_api_request(collection_template, osquerystring)
response = requests.get(request_url, verify=bool(verify_ssl))

root = ElementTree.fromstring(response.text)

# xmltxt = ElementTree.tostring(root, encoding='unicode', method='xml')
# md("```xml\n" + xmltxt + "\n```\n")

el = root.find('{http://a9.com/-/spec/opensearch/1.1/}totalResults')
print('totalResults: ', el.text)
el = root.find('{http://a9.com/-/spec/opensearch/1.1/}itemsPerPage')
print('itemsPerPage: ', el.text)
el = root.find('{http://a9.com/-/spec/opensearch/1.1/}startIndex')
print('startIndex: ', el.text)

# Present list of navigation links in table
relList = pd.DataFrame(columns=['rel', 'type', 'href'])
for lnk in root.findall('{http://www.w3.org/2005/Atom}link[@type="application/atom+xml"]'):
    if lnk.attrib['rel'] in  ['first', 'last', 'previous', 'next', 'prev']:
      relList = relList.append( { 'type': lnk.attrib['type'], 'rel': lnk.attrib['rel'], 'href': lnk.attrib['href'] }, ignore_index=True )

HTML(relList.to_html(render_links=True, escape=False))
```

**Example: 6.4**  
>  Traverse granule search results using `{startPage}`.

`{startPage}` can be used as an alternative for `{startIndex}`.

```{note}
As for all search parameters, `{startPage}` is only available when it is advertised in the corresponding OSDD document for the collection.
```

```{code-cell} ipython3
:tags: [output_scroll]

osquerystring = {}
osquerystring['count'] = '10'
osquerystring['startPage'] = '2'

request_url = get_api_request(template, osquerystring)
response = requests.get(request_url, verify=bool(verify_ssl))

root = ElementTree.fromstring(response.text)

xmltxt = ElementTree.tostring(root, encoding='unicode', method='xml')
md("```xml\n" + xmltxt + "\n```\n")

```

```{code-cell} ipython3
:tags: [output_scroll]

try: 
  el = root.find('{http://a9.com/-/spec/opensearch/1.1/}totalResults')
  print('totalResults: ', el.text)
  el = root.find('{http://a9.com/-/spec/opensearch/1.1/}itemsPerPage')
  print('itemsPerPage: ', el.text)
  el = root.find('{http://a9.com/-/spec/opensearch/1.1/}startIndex')
  print('startIndex: ', el.text)
except:
  print('invalid response')
  
# Present list of navigation links in table
relList = pd.DataFrame(columns=['rel', 'type', 'href'])
for lnk in root.findall('{http://www.w3.org/2005/Atom}link[@type="application/atom+xml"]'):
    if lnk.attrib['rel'] in  ['first', 'last', 'previous', 'next', 'prev']:
      relList = relList.append( { 'type': lnk.attrib['type'], 'rel': lnk.attrib['rel'], 'href': lnk.attrib['href'] }, ignore_index=True )

HTML(relList.to_html(render_links=True, escape=False))
```

### Sorting results

Sorting of search results is currently only available for granule searches.  Future versions of the software may support sorting for collection searches as well.

+++

**Example: 6.5**  
>  Obtain supported (granule) sorting criteria {sru:sortKeys} [RD8](#RD8) from the OSDD.   

The OSDD template lists the `sur:sortkeys` values that can be used in a granule search request.  

```{code-cell} ipython3
:tags: [output_scroll]

# Extract corresponding <Parameter> element
el = granules_url_atom.find('{http://a9.com/-/spec/opensearch/extensions/parameters/1.0/}Parameter[@value="{sru:sortKeys}"]')
xmltxt = ElementTree.tostring(el, encoding='unicode', method='xml')
md("```xml\n" + xmltxt + "\n```\n")
```

**Example: 6.6**  
>  Results can be sorted according to various criteria with {sru:sortKeys} [[RD8]](#RD8), in descending or ascending order which can be discovered in the OSDD. The example sorts in descending chronological order according to the {time:start} value. 

```{code-cell} ipython3
:tags: [output_scroll]

osquerystring = {}
osquerystring['count'] = '4'
osquerystring['geo:box'] = '14.90,37.700,14.99,37.780' 
osquerystring['sru:sortKeys'] = 'start,time,0'

request_url = get_api_request(template, osquerystring)
response = requests.get(request_url, verify=bool(verify_ssl))

root = ElementTree.fromstring(response.text)
# extract total results
el = root.find('{http://a9.com/-/spec/opensearch/1.1/}totalResults')
print('totalResults: ', el.text)

dataframe = load_dataframe(response)
# dataframe.head(20)
HTML(dataframe.transpose(copy=True).to_html(render_links=True, escape=False))
```

**Example: 6.7**  
>  Results can be sorted according to various criteria with {sru:sortKeys}, in descending or ascending order which can be discovered in the OSDD. The example sorts in ascending chronological order according to the {time:start} value. 

```{code-cell} ipython3
:tags: [output_scroll]

osquerystring = {}
osquerystring['count'] = '4'
osquerystring['geo:box'] = '14.90,37.700,14.99,37.780' 
osquerystring['sru:sortKeys'] = 'start,time,1'

request_url = get_api_request(template, osquerystring)
response = requests.get(request_url, verify=bool(verify_ssl))

root = ElementTree.fromstring(response.text)
# extract total results
el = root.find('{http://a9.com/-/spec/opensearch/1.1/}totalResults')
print('totalResults: ', el.text)

dataframe = load_dataframe(response)
# dataframe.head(20)
HTML(dataframe.transpose(copy=True).to_html(render_links=True, escape=False))
```

### Faceted search

Faceted search results are currently only available for collection searches.  Future versions of the software may support faceted search results for granule searches as well.

The server can supply faceted results for a query: i.e. an analysis of how the search results are distributed over various categories (or "facets"). For example, the analysis may reveal how the results are distributed by organization. The client might then refine the query to one particular organization among those listed.

By default, the search response <atom:feed> contains faceted search results under the element `<sru:facetedResults>`.  In the current implementation, the faceted search information groups the results by a number of cobfigurable facts including `platform`, by `instrument`, by `organisation` etc.  Future versions of the software will all the client to specify which facets it would like to receive a part of the request.

The faceted Results are consistent with the OASIS searchRetrieve facetedResults XML schema available at http://docs.oasis-open.org/search-ws/searchRetrieve/v1.0/os/schemas/facetedResults.xsd.

+++

**Example: 6.8**  
>  Current content of `<sru:facetedResults>` of a collection search response and extract facet information for 'platform'. 

```{code-cell} ipython3
:tags: [output_scroll]

osquerystring = {}
osquerystring['eo:platform'] = 'Cryosat-2' # 'proba-1'
osquerystring['count'] = '10'

request_url = get_api_request(collection_template, osquerystring)
response = requests.get(request_url, verify=bool(verify_ssl))
root = ElementTree.fromstring(response.text)

# Extract faceted results for platform
# el = root.find('{http://a9.com/-/opensearch/extensions/sru/2.0/}facetedResults')
el = root.find('.//{http://a9.com/-/opensearch/extensions/sru/2.0/}facet[{http://a9.com/-/opensearch/extensions/sru/2.0/}index="eo:platform"]')

# Show eo:platform facet information as XML
xmltxt = ElementTree.tostring(el, encoding='unicode', method='xml')
md("```xml\n" + xmltxt + "\n```\n")
```

**Example: 6.9**  
>  Convert `<sru:facetedResults>` facet information for 'eo:platform' to a dataframe and display as a bar chart.

```{code-cell} ipython3
:tags: [output_scroll]

df = load_facet(root, "eo:platform")
# df.set_index('name', inplace=True)
df.plot(kind='barh', figsize=(9, 7))
```

**Example: 6.10**  
>  Convert `<sru:facetedResults>` facet information for 'eo:instrument' to a dataframe and display as a bar chart.

```{code-cell} ipython3
:tags: [output_scroll]

df = load_facet(root, "eo:instrument")
# df.set_index('name', inplace=True)
df.plot(kind='barh', figsize=(9, 7))
```

**Example: 6.11**  
>  Convert `<sru:facetedResults>` facet information for 'eo:organizationName' to a dataframe and display as a bar chart.

```{code-cell} ipython3
:tags: [output_scroll]

df = load_facet(root, "eo:organisationName")
# df.set_index('name', inplace=True)
df.plot(kind='barh', figsize=(9,7))
```

### Content negotiation

The API provides the `httpAccept` HTTP query parameter to request for different media types which has the same behaviour as the [searchRetrieve httpAccept parameter](http://docs.oasis-open.org/search-ws/searchRetrieve/v1.0/os/part3-sru2.0/searchRetrieve-v1.0-os-part3-sru2.0.html#_Toc324162475) in [[RD7]](#RD7).  When using the OpenSearch interface, this parameter is prefilled with the media type selected when
the URL template was extracted from the OSDD](#selecting-mediatype-in-osdd).  Future versions of the API will allow to use the HTTP header parameter `Accept` to provide the media type when the API is accessed directly without passing via the OSDD.  In case both parameters are provided, the `httpAccept` parameter has precedence.  

+++

<a name='Further-Reading'></a>  
## Further Reading

| **ID**  | **Title** | 
| -------- | --------- | 
| `RD1` <a name="RD1"></a> | [CEOS OpenSearch Best Practice Document, Version 1.3](https://github.com/radiantearth/stac-spec/blob/master/catalog-spec/catalog-spec.md) | 
| `RD2` <a name="RD2"></a> | [OGC 10-032r8 - OpenSearch Geo and Time Extensions ](https://portal.ogc.org/files/?artifact_id=56866) | 
| `RD3` <a name="RD3"></a> | [OGC 13-026r9 - OpenSearch Extension for Earth Observation](http://docs.opengeospatial.org/is/13-026r9/13-026r9.html) | 
| `RD4` <a name="RD4"></a> | [RFC 4287 - The Atom Syndication Format](https://datatracker.ietf.org/doc/html/rfc4287) | 
| `RD5` <a name="RD5"></a> | [WGISS CDA OpenSearch Client Guide](https://ceos.org/document_management/Working_Groups/WGISS/Documents/Discovery-Access/WGISS%20CDA%20OpenSearch%20Client%20Guide-v1.2.pdf) | 
| `RD6` <a name="RD6"></a>| [OASIS searchRetrieve: Part 7. Explain Version 1.0](http://docs.oasis-open.org/search-ws/searchRetrieve/v1.0/os/part7-explain/searchRetrieve-v1.0-os-part7-explain.html) |
| `RD7` <a name="RD7"></a>| [OASIS searchRetrieve: Part 3. APD Binding for SRU 2.0 Version 1.0](http://docs.oasis-open.org/search-ws/searchRetrieve/v1.0/os/part3-sru2.0/searchRetrieve-v1.0-os-part3-sru2.0.html) |
| `RD8` <a name="RD8"></a>| [OpenSearch SRU Extension](https://github.com/dewitt/opensearch/tree/master/mediawiki/Community/Proposal/Specifications/OpenSearch/Extensions/SRU/1.0) |
| `RD9` <a name="RD9"></a>| [OpenSearch Semantic Extension](https://github.com/dewitt/opensearch/tree/master/mediawiki/Community/Proposal/Specifications/OpenSearch/Extensions/Semantic/1.0) |
| `RD10` <a name="RD10"></a>| [OGC 08-167r2 - Semantic annotations in OGC standards](https://portal.ogc.org/files/?artifact_id=47857) |
