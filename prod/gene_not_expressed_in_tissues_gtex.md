# Tissues where a gene is not expressed （池田）

## Description

- Data sources
    - [GTEx version 8](https://gtexportal.org/home/datasets)

## Endpoint

https://integbio.jp/togosite/sparql

## `data`
```sparql
PREFIX obo: <http://purl.obolibrary.org/obo/>
PREFIX enso: <http://rdf.ebi.ac.uk/terms/ensembl/>
PREFIX dcterms: <http://purl.org/dc/terms/>
PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>
PREFIX refexo:  <http://purl.jp/bio/01/refexo#>
PREFIX sio: <http://semanticscience.org/resource/>
PREFIX schema: <http://schema.org/>
PREFIX ensg: <http://rdf.ebi.ac.uk/resource/ensembl/>

SELECT DISTINCT ?parent_label ?parent ?child ?child_label
WHERE {
  GRAPH <http://rdf.integbio.jp/dataset/togosite/refex_gtex_v8_summary> {
    ?refex a refexo:RefExEntry ;
           sio:SIO_000216 [
             a refexo:logTPMMax ;
             sio:SIO_000300 0
             #sio:SIO_000300 ?logtpmmax
           ] ;
           refexo:isMeasurementOf ?child ;
           refexo:refexSample ?refexs .
    #FILTER(?logtpmmax >= 0 && ?logtpmmax < 1)
  }
  
  GRAPH <http://rdf.integbio.jp/dataset/togosite/refexsample_gtex_v8_summary> {
    ?refexs dcterms:description ?parent_label ;
            dcterms:identifier ?parent .
  }

  GRAPH <http://rdf.integbio.jp/dataset/togosite/ensembl> {
    ?child a ?type ;
           rdfs:label ?child_label .
    VALUES ?type { enso:lncRNA obo:SO_0001217 obo:SO_0000336 enso:TEC } # lncRNA, protein_coding_gene, pseudogene
    MINUS { ?child a enso:rRNA_pseudogene }
  }

}
```

## `all`
```sparql
PREFIX obo: <http://purl.obolibrary.org/obo/>
PREFIX enso: <http://rdf.ebi.ac.uk/terms/ensembl/>
PREFIX rdfs: <http://www.w3.org/2000/01/rdf-schema#>
PREFIX refexo:  <http://purl.jp/bio/01/refexo#>

SELECT DISTINCT ?child ?child_label
WHERE {
  GRAPH <http://rdf.integbio.jp/dataset/togosite/refex_gtex_v8_summary> {
    ?refex a refexo:RefExEntry ;
           refexo:isMeasurementOf ?child .
  }
  GRAPH <http://rdf.integbio.jp/dataset/togosite/ensembl> {
    ?child a ?type ;
           rdfs:label ?child_label .
    [] obo:SO_transcribed_from ?child .
    VALUES ?type { enso:lncRNA obo:SO_0001217 obo:SO_0000336 enso:TEC }
    MINUS { ?child a enso:rRNA_pseudogene }
  }
}
```

## `return`

```javascript
({data, all}) => {
  const idPrefix = "http://rdf.ebi.ac.uk/resource/ensembl/";

  let tree = [{
    id: "root",
    label: "root node",
    root: true
  }];
  let chk = {};
  let unclassified = {};
  all.results.bindings.forEach(d => {
    unclassified[d.child.value] = d.child_label.value;
  });
  data.results.bindings.forEach(d => {
    if (!chk[d.parent.value]) {
      chk[d.parent.value] = true;
      tree.push({     
        id: d.parent_label.value,
        label: d.parent_label.value,
        leaf: false,
        parent: "root"
      });
    }
    delete unclassified[d.child.value];
    tree.push({
      id: d.child.value.replace(idPrefix, ""),
      label: d.child_label.value,
      leaf: true,
      parent: d.parent_label.value
    });
  });

  tree.push({     
    id: "unclassified",
    label: "Expressed in all tissues",
    leaf: false,
    parent: "root"
  });

  Object.keys(unclassified).forEach(d => {
    tree.push({
      id: d.replace(idPrefix, ""),
      label: unclassified[d],
      leaf: true,
      parent: "unclassified"
    });
  });
  return tree;
};

```
