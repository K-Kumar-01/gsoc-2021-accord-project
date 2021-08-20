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
After the community bonding period, it was time to get the hands dirty. The complete work was done on  [`algoo-ooxml`](https://github.com/accordproject/markdown-transform/tree/algoo-ooxml) branch. Below is a sample CiceroMark and a tree strucure for same.
```js
{
  "$class": "org.accordproject.commonmark.Paragraph",
  "nodes": [
    {
      "$class": "org.accordproject.commonmark.Strong",
      "nodes": [
        {
          "$class": "org.accordproject.commonmark.Text",
          "text": "strong "
        },
        {
          "$class": "org.accordproject.commonmark.Code",
          "text": "code"
        }
      ]
    },
    {
      "$class": "org.accordproject.commonmark.Emph",
      "nodes": [
        {
          "$class": "org.accordproject.commonmark.Text",
          "text": "italics"
        }
      ]
    }
  ]
}
```
The above JS can be represented in a tree form as:
![basic-structure](https://user-images.githubusercontent.com/59891164/130233702-fee6b84f-0aa6-4832-9261-868bf8bd6222.png)

Things were goging smooth and the transformer was being improved with each PR until... a challenge came.

### Challenge-1: Transformation Logic Failure
The transformation logic which was used currently could not handle nesting of elements. This means it was unable to parse the below text:  
*Hello __world__*.

**Solution:** Rewriting the transformation logic by using the [depth first search(DFS)](https://en.wikipedia.org/wiki/Depth-first_search) for converting `CiceroMark<->JSON`.  
_Reason_: From the image of the JS tree above, it is visible that each node which is responsible for rendering the text will always be present as the leaf node. So generate a OOXML for the current leaf node keeping track of the styling properties it has visited and use it finally to generate OOXML for the same.

The image below shows the tree structure for the CiceroMark and OOXML.
![trees](https://user-images.githubusercontent.com/59891164/130236278-5d3b29b5-affd-49d1-bcc0-a07f5729b5d8.png)


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
##### Demo of the process flow
![dfs](https://user-images.githubusercontent.com/59891164/130235957-c3b999c0-e90f-49a0-9a6e-b3cc921aea05.gif)

Rewriting of the [`CiceroMark->OOXML`](https://github.com/accordproject/markdown-transform/pull/418) and [`OOXML->CiceroMark`](https://github.com/accordproject/markdown-transform/pull/421).

After the rewriting it was time to resume the transformation of different entities on which I worked upon and it went fine before I stumbled on another challenge:

### Challenge-2: Optional and Conditional Nodes
The optional and conditional nodes 

Finally, I was able to write a transformer which can convert major of the CiceroMark nodes to corresponding OOXML tags.

Side by side, I [integrated](https://github.com/accordproject/markdown-transform/pull/424) the current transformation with the [markus](https://github.com/accordproject/markdown-transform/tree/master/packages/markdown-cli). This esnures that one can use the transformation using the CLI. The command for transforming ciceromark_parsed to ooxml looks like:

#### Create and view the transformed file
To create a file, use: 
```bash
markus transform --from ciceromark_parsed --to ooxml --input <path> --output <path>.xml
```
**Note**: Save files with `xml` format.
To use the created `xml` file, one can follow these steps:
1. Copy the `xml` in the file.
2. Open MS-WORD. Insert the ooxml using the [Script Lab](https://docs.microsoft.com/en-us/office/dev/add-ins/overview/explore-with-script-lab) add-in.
   Be sure to install the add-in.
   
Alternatively, one can also right click on the document and open with MS-WORD to see the transformed content.

In future, integration with the [add-in](https://github.com/accordproject/cicero-word-add-in) will ensure interaction with the transformed file with advanced features. Currently, the features are limited via above two methods.

## Result
```
# For testing purpose only

### Checking Heading Level 3 *with* some **styling**

Now let us create a paragraph for the same which _will use with **nesting and making `inline codes`**_ to ensure` it works finely.

Also here is a [link](https://www.google.com)

[*Links*](https://www:google.com) can also be styled.

<codeblock tags>
Finally a codeblock to ensure everything's working fine and nicely.
<codeblock tags>
```
Paste the above [here](https://templatemark-dingus.netlify.app/) and switch to `ast` tab to see the corresponding CiceroMark.

#### Markdown Representation
![Screenshot (120)](https://user-images.githubusercontent.com/59891164/130188842-f3eb582e-dd11-45bf-83ba-ca7c29a0218e.png)

#### OOXML Representation
![Screenshot (119)](https://user-images.githubusercontent.com/59891164/130188961-9c30b512-266b-4ea5-bfc4-712fbfe3b7b6.png)


## Current Standing
Currently, the `CiceroMark<->OOXML` is able to do the following conversions:
- Text
- Emphasis
- Heading
- Variable
- Softbreak
- Strong
- Code
- Thematic Break
- Codeblock
- Clause
- Link
- Optional
- Conditional
- Formula

The project was started as an improvement to the already existing transformer and a total of 29 [PR](https://github.com/accordproject/markdown-transform/pulls?q=is%3Apr+author%3AK-Kumar-01+is%3Aclosed) were made including various commit messages and 4 issues which were made.


