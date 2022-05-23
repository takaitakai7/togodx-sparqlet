# Detected organs of microbes in human from MicrobeDB.jp

## Endpoint

https://microbedb.jp/sparql

## `leaf`
```sparql
PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>
PREFIX sio: <http://semanticscience.org/resource/>
PREFIX meo: <http://purl.jp/bio/11/meo/>
PREFIX mdbv: <http://purl.jp/bio/11/mdbv#>
PREFIX obo: <http://purl.obolibrary.org/obo/>
PREFIX tax: <http://ddbj.nig.ac.jp/ontologies/taxonomy/>
PREFIX idtax: <http://identifiers.org/taxonomy/>
SELECT DISTINCT ?child ?child_label ?parent
WHERE {
  [] a sio:SIO_001050 ;
     sio:SIO_000255/sio:SIO_000255 [
       a mdbv:HostTaxonIDAnnotation ;
       sio:SIO_000671 idtax:9606
     ] ;
     obo:RO_0002162 ?child ;
     sio:SIO_000255/sio:SIO_000255 [
       a mdbv:EnvAnnotation ; 
       sio:SIO_000671 ?parent
     ] .
  ?parent rdfs:isDefinedBy	meo: ;
       rdfs:subClassOf* meo:MEO_0000175 . # foundamental organ
  ?child rdfs:label ?child_label_pre .
  BIND(STR(?child_label_pre) AS ?child_label) # uniform with/without xsd:string datatype
  MINUS { ?child rdfs:subClassOf* <http://identifiers.org/taxonomy/408169> } # w/o metagenome
}
```

## `graph`
```sparql
PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>
PREFIX meo: <http://purl.jp/bio/11/meo/>
SELECT DISTINCT ?child ?child_label ?parent
WHERE {
  ?parent rdfs:subClassOf* meo:MEO_0000175 .
  ?child rdfs:subClassOf ?parent ;
       rdfs:label ?child_label .
}
```

## `return`
```javascript
({leaf, graph}) => {
  const idPrefix = "http://identifiers.org/taxonomy/";
  const categoryPrefix = "http://purl.jp/bio/11/meo/";

  let tree = [
    {
      id: "MEO_0000175",
      label: "foundamental organ",
      root: true
    }
  ];
  
  // 親子関係
  graph.results.bindings.map(d => {
    tree.push({
      id: d.child.value.replace(categoryPrefix, ""),
      label: d.child_label.value,
      parent: d.parent.value.replace(categoryPrefix, "")
    })
  })
  // アノテーション関係
  leaf.results.bindings.map(d => {
    tree.push({
      id: d.child.value.replace(idPrefix, ""),
      label: d.child_label.value,
      leaf: true,
      parent: d.parent.value.replace(categoryPrefix, "")
    })
  })
  
  return tree;
}
```