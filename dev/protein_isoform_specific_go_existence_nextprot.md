# Isoform specific GO annotation existence from nextprot (守屋)

## Endpoint
https://integbio.jp/togosite/sparql

## `hasSpecific`
```sparql
PREFIX : <http://nextprot.org/rdf#>
PREFIX skos: <http://www.w3.org/2004/02/skos/core#>
PREFIX core: <http://purl.uniprot.org/core/>
SELECT ?np ?label
FROM <http://rdf.integbio.jp/dataset/togosite/nextprot>
FROM <http://rdf.integbio.jp/dataset/togosite/uniprot>
WHERE {
  VALUES ?go { :GoBiologicalProcess :GoCellularComponent :GoMolecularFunction }
  ?np a :Entry ;
      skos:exactMatch/core:mnemonic ?label ;
      :isoform/:generalAnnotation ?annotation .
    ?annotation a ?go ; 
                :isoformSpecificity :SPECIFIC .         
}
```

## `hasIsoform`
```sparql
PREFIX : <http://nextprot.org/rdf#>
PREFIX skos: <http://www.w3.org/2004/02/skos/core#>
PREFIX core: <http://purl.uniprot.org/core/>
SELECT ?np ?label
WHERE {
  {
    SELECT DISTINCT ?np ?label (COUNT (?iso) AS ?iso_count) 
    FROM <http://rdf.integbio.jp/dataset/togosite/nextprot>
    FROM <http://rdf.integbio.jp/dataset/togosite/uniprot>
    WHERE {
      ?np a :Entry ;
          skos:exactMatch/core:mnemonic ?label ;
          :isoform ?iso .
    }
  }
  FILTER (?iso_count > 1)
}
```


## `return`
```javascript
({hasSpecific, hasIsoform})=>{
  let idPrefix = "http://nextprot.org/rdf/entry/NX_";
  let withId = "hasSpecific";
  let withoutId = "unclassified";
  
  let tree = [
    {
      id: "root",
      root: true
    },{
      id: withId,
      label: "With isoform specifcity",
      parent: "root"
    },{
      id: withoutId,
      label: "Unknown",
      parent: "root"
    }
  ];

  let specific = {};
  hasSpecific.results.bindings.map(d => {
    specific[d.np.value] = true;
    tree.push({
      id: d.np.value.replace(idPrefix, ""),
      label: d.label.value,
      parent: withId,
      leaf: true
    });
  });
  hasIsoform.results.bindings.map(d => {
    if (!specific[d.np.value]) {
      tree.push({
        id: d.np.value.replace(idPrefix, ""),
        label: d.label.value,
        parent: withoutId,
        leaf: true
      });
    }
  });

  return tree;
}
```
