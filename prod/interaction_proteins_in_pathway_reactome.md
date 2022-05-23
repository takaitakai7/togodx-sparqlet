# UniProt reactome-pathway classification（守屋）

## Description

- Data sources
    - Reactome-classified pathways for each protein
    - This item based on the data of Reactome Version 71 (02, December 2019).
       - The latest data can be obtained from the URL below. https://reactome.org/download-data
- Query
    - Input
        - Reactome ID, Uniprot ID
    - Output
        - The number of Uniprot entries included in each pathway in Reactome
        - If a Uniprot id is entered, it returns the pathway to which the protein belongs

## Endpoint
https://integbio.jp/togosite/sparql

## `leaf`
- pathway と UniProt のアノテーション関係
- 産物と制御
  - 制御: reaction ^biopax:controlled/biopax:controller control-component
  - 産物: reaction biopax:left|biopax:right|biopax:product product-component
```sparql
PREFIX biopax: <http://www.biopax.org/release/biopax-level3.owl#>
PREFIX xsd: <http://www.w3.org/2001/XMLSchema#>
SELECT DISTINCT ?parent ?child ?parent_label
FROM <http://rdf.integbio.jp/dataset/togosite/reactome>
WHERE {
  ?top_path a biopax:Pathway ;
            biopax:organism ?org .
  ?org biopax:name "Homo sapiens"^^xsd:string .
  MINUS { [] biopax:pathwayComponent ?top_path . }
  ?top_path biopax:pathwayComponent* ?path .
  ?path a biopax:Pathway ;
        biopax:displayName ?parent_label ;
        biopax:xref [
          biopax:db "Reactome"^^xsd:string ;
          biopax:id ?parent ;
        ] ;
        biopax:pathwayComponent ?reaction .
  ?reaction a biopax:BiochemicalReaction .
  ?reaction biopax:left|biopax:right|biopax:product|^biopax:controlled/biopax:controller ?component .
  ?component (biopax:memberPhysicalEntity|biopax:component)*/biopax:entityReference/biopax:xref [
    biopax:db "UniProt"^^xsd:string ;
    biopax:id ?child
  ] .
}
```

## `graph`
- pathway の親子関係
```sparql
PREFIX biopax: <http://www.biopax.org/release/biopax-level3.owl#>
PREFIX xsd: <http://www.w3.org/2001/XMLSchema#>
SELECT DISTINCT ?parent ?child ?parent_label ?child_label
FROM <http://rdf.integbio.jp/dataset/togosite/reactome>
WHERE {
  ?top_path a biopax:Pathway ;
            biopax:organism ?org .
  ?org biopax:name "Homo sapiens"^^xsd:string .
  MINUS { [] biopax:pathwayComponent ?top_path . }
  ?top_path biopax:pathwayComponent* ?parent_path .
  ?parent_path a biopax:Pathway ;
               biopax:displayName ?parent_label ;
               biopax:xref [
                 biopax:db "Reactome"^^xsd:string ;
                 biopax:id ?parent 
               ] ;
               biopax:pathwayComponent ?child_path .
  ?child_path a biopax:Pathway ;
              biopax:displayName ?child_label ;
              biopax:xref [
                biopax:db "Reactome"^^xsd:string ;
                biopax:id ?child 
              ] .
}
```

## `top`
- ヒト pathway の top 階層
```sparql
PREFIX biopax: <http://www.biopax.org/release/biopax-level3.owl#>
PREFIX xsd: <http://www.w3.org/2001/XMLSchema#>
SELECT DISTINCT ?top ?top_label
FROM <http://rdf.integbio.jp/dataset/togosite/reactome>
WHERE {
  ?top_path a biopax:Pathway ;
            biopax:pathwayComponent ?child_path .
  MINUS { [] biopax:pathwayComponent ?top_path . }
  ?top_path biopax:organism ?org ;
            biopax:displayName ?top_label ;
            biopax:xref [
              biopax:db "Reactome"^^xsd:string ;
              biopax:id ?top ;
            ] .
  ?org biopax:name "Homo sapiens"^^xsd:string .
}
```

## `allLeaf`
- 全 UniProt (without annotation 用)
```sparql
PREFIX up: <http://purl.uniprot.org/core/>
PREFIX taxon: <http://purl.uniprot.org/taxonomy/>
SELECT DISTINCT ?leaf ?leaf_label
FROM <http://rdf.integbio.jp/dataset/togosite/uniprot>
WHERE {
  ?leaf a up:Protein ;
        up:organism taxon:9606 ;
        up:mnemonic ?leaf_label ;
        up:proteome ?proteome .
  FILTER(REGEX(STR(?proteome), "UP000005640"))
}
```

## `return`
- 整形
```javascript
({top, leaf, graph, allLeaf}) => {
  const idPrefix = "http://purl.uniprot.org/uniprot/";
  const withoutId = "unclassified";

  let tree = [
    {
      id: "root",
      label: "root node",
      root: true
    },{
      id: withoutId,
      label: "Not in the pathway",
      parent: "root"
    }
  ];
  
  let id2label = {};
  allLeaf.results.bindings.map(d => {
    id2label[d.leaf.value.replace(idPrefix, "")] = d.leaf_label.value;
  })

  let withAnnotation = {};
  // トップとルートの関係
  top.results.bindings.map(d => {
    tree.push({
      id: d.top.value,
      label: d.top_label.value,
      parent: "root"
    })
  })
  // 親子関係
  graph.results.bindings.map(d => {
    tree.push({
      id: d.child.value,
      label: d.child_label.value,
      parent: d.parent.value
    })
  })
  // アノテーション関係
  leaf.results.bindings.map(d => {
    withAnnotation[d.child.value] = true;
    if (id2label[d.child.value]) {  // proteome check
      tree.push({
        id: d.child.value,
        label: id2label[d.child.value],
        leaf: true,
        parent: d.parent.value
      })
    }
  })
  // アノテーション無し要素
  allLeaf.results.bindings.map(d => {
    let id = d.leaf.value.replace(idPrefix, "");
    if (!withAnnotation[id]) {
      tree.push({
        id: id,
        label: d.leaf_label.value,
        leaf: true,
        parent: withoutId
      })
    }
  })
  
  return tree;
}
```