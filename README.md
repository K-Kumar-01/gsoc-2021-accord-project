# gsoc-2021-accord-project
Report of the work done in gsoc 2021

# GSOC 2021: [CiceroMark](https://docs.accordproject.org/docs/markup-cicero) <-> OOXML

## Accord Project
Accord Project is an organistaion which is developing an ecosysten and tools specifically for the smart legal contracts.

## About the Project
Accord Project create [templates](https://templates.accordproject.org/) for contracts and clauses using the [cicero](https://docs.accordproject.org/docs/started-installation.html), [ergo](https://docs.accordproject.org/docs/logic-ergo.html), and [concerto](https://docs.accordproject.org/docs/model-concerto.html). These templates, which are essentially `.cta` files can then be converted into different formats like `md`, `pdf`, `json` using the [transformer](https://github.com/accordproject/markdown-transform) library.

My task was to improve the transformer by allowing the `CiceroMark`<->`OOXML` interconversion by including the logic for missing entities in the transformer and improve the existing transformations ensuring proper roundtrip between the two.

## Use Case
By integrating the above transformation process, one can convert the templates into an `xml` file which can be opened with MS Word, making it easier for non technical people to work with the contracts and clauses. An [add-in](https://github.com/accordproject/cicero-word-add-in) is already created(not fully capable) which can make the interaction easier with these Word documents even before.

## OOXML
Office Open Extensible Markup Language abbreviated as [`OOXML`](https://en.wikipedia.org/wiki/Office_Open_XML) is a zip-based XML file format which was developed by Microsoft. It can be used to represent documents, presentations, spreadsheets, etc.
OOXML acts as backcbone of these documents as it is completely responsible for rendering the content which is visible on a word document or a presentation.
The docx format is a representation of an OOXML which is supposed to be a “word processing document“.
A basic structure of OOXML in word document:
```xml
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<?mso-application progid="Word.Document"?>
<pkg:package ...>
  <pkg:part pkg:name="/_rels/.rels" ...>
  </pkg:part >
  <pkg:part pkg:name="/word/document.xml" ...>
    <w:document>
    </w:document>
  </pkg:part>
  <!--More packages for styling, relationships, etc.-->
</pkg:package>
```

`<w:document>` is the area which includes the text and styling tags. This was the area which on which I worked.

## Pre-Coding
Apart from geeting familiar with the project, I explored more about OOXML and CiceroMark and the existing transformer. There were discussion which involved the approach to integrate the transformer. Things like which nodes to transform have a higher prioerity and which can be made a lower priority were also decided.
The transformer lied in the [`algoo-ooxml`](https://github.com/accordproject/markdown-transform/tree/algoo-ooxml) which was not updated frequently and was out of sync with `main` branch. So, rebasing was done to sync them at the start.
Besides this, a basic setup was done for the project.
The corresponding details can be found in this [blog](https://github.com/accordproject/cicero-word-add-in/wiki/Blog).

## Journey
After the community bonding period, it was time to get the hands dirty. The complete work was done on  [`algoo-ooxml`](https://github.com/accordproject/markdown-transform/tree/algoo-ooxml) branch. The initial transformation was done for `text-and-emphasis` transformer. Graudally, different transformers were being added and the process went on smoothly until we found that the transformation which was used currently not only missed entities which I was adding but could not handle with nesting of elements. This means it was unable to parse the following correctly:
```js
  {
    "$class": "org.accordproject.commonmark.Paragraph",
    "nodes": [
      {
        "$class": "org.accordproject.commonmark.Emph",
        "nodes": [
          {
            "$class": "org.accordproject.commonmark.Text",
            "text": "hello "
          },
          {
            "$class": "org.accordproject.commonmark.Strong",
            "nodes": [
              {
                "$class": "org.accordproject.commonmark.Text",
                "text": "world"
              }
            ]
          }
        ]
      }
    ]
  }
```

This resulted in rewriting the transformation logic by using the [depth first search(DFS)](https://en.wikipedia.org/wiki/Depth-first_search) for converting `CiceroMark<->JSON`.

#### Explanation of the DFS for CiceroMark<->JSON
1. If the node belongs to `block` nodes(paragraph, clause, heading), use DFS for the current node.
2. If the node belongs to `properties` nodes(emphasis, link, strong, etc.) append them to properties
3. If the node belongs to `terminating` nodes(text, inline-code, softbreak, thematic break, etc.) then use the properties to generate the OOXML.


```js
     /**
     * Traverses CiceroMark nodes in a DFS approach
     *
     * @param {object} node             CiceroMark Node
     * @param {array}  properties       Properties to be applied on current node
     * @param {string} parent           Parent element of the node(paragraph, clause, heading, and optional)
     * @param {object} parentProperties Properties of parent on which children depend for certain styles(vanish property for hidden nodes)
     * @returns {string} OOXML for the inline nodes of the current parent
     */
    traverseNodes(node, properties = [], parent, parentProperties = {}){
      if(typeOf(node)===block){
        traverseNodes(node, properties = [], node.$class, parentProperties = {})
      }else if(typeOf(node)===terminating){
        updatedProperties=[...properties,node.$class)
        traverseNodes(node, properties = [], node.$class, parentProperties = {})
      }else{
        generateOOXML
      }
    }
```

#### Explanation with an example
Example: Let nodes be `Paragraph->Strong->Emph->Text`.
1. Node= Paragraph. Dive deep
2. Node = Strong. properties= [...properties, Strong]
3. Node = Emph. properties= [...properties, Emph]
4. Node = Text. Iterate over properties in the order (1st to last). Found strong, append strong property. Found emph, append emph property. Properties end. Place the text value in <w:t> . Place the property elements and <w:t>  in <w:r> . The full ooxml for the given text node is generated.

The approach for the `OOXM->CiceroMark` is also DFS based and similar to the above.

Rewriting of the [`CiceroMark->OOXML`](https://github.com/accordproject/markdown-transform/pull/418) and [`OOXML->CiceroMark`](https://github.com/accordproject/markdown-transform/pull/421).


After the rewriting it was time to resume the transformation of different entities on which I worked upon.

Finally, I was able to write a transformer which can convert major of the CiceroMark nodes to corresponding OOXML tags.

Side by side, I [integrated](https://github.com/accordproject/markdown-transform/pull/424) the current transformation with the [markus]https://github.com/accordproject/markdown-transform/tree/master/packages/markdown-cli). This esnures that one can use the transformation using the CLI. The command for transforming ciceromark_parsed to ooxml looks like:
```bash
markus transform --from ciceromark_parsed --to ooxml --input <path> --output <path>.xml
```
**Note**: Save files with `xml` format to ensure that they can be properly rendered when you open them in MS-WORD
