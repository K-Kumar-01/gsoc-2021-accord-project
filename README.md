# GSoC-2021-accord-project
Report of the work done in GSoC 2021

# GSOC 2021: [CiceroMark](https://docs.accordproject.org/docs/markup-cicero) <-> OOXML

## Accord Project
Accord Project is an organization that is developing an ecosystem and tools specifically for smart legal contracts.

## About the Project
Accord Project create [templates](https://templates.accordproject.org/) for contracts and clauses using the [cicero](https://docs.accordproject.org/docs/started-installation.html), [ergo](https://docs.accordproject.org/docs/logic-ergo.html), and [concerto](https://docs.accordproject.org/docs/model-concerto.html). These templates, which arethat are stored in `.cta` files can then be converted into different formats like `md`, `pdf`, `json` using the [transformer](https://github.com/accordproject/markdown-transform) library.

My task was to improve the transformer by allowing the `CiceroMark`<-> `OOXML` interconversion by including the logic for missing entities in the transformer and improve the existing transformations ensuring proper roundtrip between the two.

## Use Case
By integrating the above transformation process, one can convert the templates into an `XML` file which can be opened with MS Word, making it easier for non-technical people to work with the contracts and clauses. An [add-in](https://github.com/accordproject/cicero-word-add-in) is already created(not fully capable) which can make the interaction easier with these Word documents even before.

The blog post for the add-in can be read [here](https://accordproject.org/news/gsoc-2020-cicero-word-add-in/).

## OOXML
Office Open Extensible Markup Language abbreviated as [`OOXML`](https://en.wikipedia.org/wiki/Office_Open_XML) is a zip-based XML file format which was developed by Microsoft. It can be used to represent documents, presentations, spreadsheets, etc.
OOXML acts as the backbone of these documents as it is completely responsible for rendering the content which is visible on a word document or a presentation.
The docx format is a representation of an OOXML which is supposed to be a ???word processing document???.
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

`<w:document>` is the area that includes the text and styling tags. This was the area which on which I worked.

## Pre-Coding
Apart from getting familiar with the project, I explored OOXML, CiceroMark, and the existing transformer. Some discussions involved the approach to integrate the transformer. Things like which nodes to transform have a higher priority and which can be made a lower priority were also decided.
The transformer lied in the [`algoo-ooxml`](https://github.com/accordproject/markdown-transform/tree/algoo-ooxml) which was not updated frequently and was out of sync with the `main` branch. So, rebasing was done to sync them at the start.
Besides this, a basic setup was done for the project.
The corresponding details can be found in this [blog](https://github.com/accordproject/cicero-word-add-in/wiki/Blog).

## Journey
After the community bonding period, it was time to get the hands dirty. The complete work was done on  [`algoo-ooxml`](https://github.com/accordproject/markdown-transform/tree/algoo-ooxml) branch. Below is a sample CiceroMark and a tree structure for the same.
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
![basic-structure](https://user-images.githubusercontent.com/59891164/130433050-34259c41-68ed-4e22-b1c4-344bdb8506f6.png)


Things were going smooth and the transformer was being improved with each PR until... a challenge came.

### Challenge-1: Transformation Logic Failure
The transformation logic which was used currently could not handle the nesting of elements. This means it was unable to parse the below text:  
*Hello __world__*.

**Solution:** Rewriting the transformation logic by using the [depth first search(DFS)](https://en.wikipedia.org/wiki/Depth-first_search) for converting `CiceroMark<->JSON`.  
_Reason_: From the image of the JS tree above, it is visible that each node that is responsible for rendering the text will always be present as the leaf node. To generate an OOXML for the current leaf node, keep track of the styling properties it has visited and use it to generate OOXML for the same.

The image below shows the tree structure for the CiceroMark and OOXML.
![trees](https://user-images.githubusercontent.com/59891164/130236278-5d3b29b5-affd-49d1-bcc0-a07f5729b5d8.png)


#### Explanation of the DFS for CiceroMark<->JSON
1. If the node belongs to `block` nodes(paragraph, clause, heading), use DFS for the current node.
2. If the node belongs to `properties` nodes(emphasis, link, strong, etc.) append them to properties
3. If the node belongs to `terminating` nodes(text, inline-code, softbreak, thematic break, etc.) then use the properties to generate the OOXML.
##### Demo of the process flow
![dfs](https://user-images.githubusercontent.com/59891164/130235957-c3b999c0-e90f-49a0-9a6e-b3cc921aea05.gif)

Rewriting of the [`CiceroMark->OOXML`](https://github.com/accordproject/markdown-transform/pull/418) and [`OOXML->CiceroMark`](https://github.com/accordproject/markdown-transform/pull/421).

After the rewriting it was time to resume the transformation of different entities on which I worked upon and it went fine before I stumbled on another challenge:

### Challenge-2: Optional and Conditional Nodes
The optional and conditional nodes, unlike others, can have different nodes depending on conditions.
![cond-opt](https://user-images.githubusercontent.com/59891164/130319714-5ce64b68-b54a-405c-b8c3-3f08d1f784bf.png)
The main problem that arose here was to hide the `nodes` of false conditions.
**Solution:** The problem was resolved by using the `w:vanish` property of `OOXML`. It specifies that the content is to be hidden from display at display time. In other words, it is similar to ` display:none ` of CSS. More information [here](http://officeopenxml.com/WPtextFormatting.php)

Finally, I wrote a transformer that can convert the major of the CiceroMark nodes to corresponding OOXML tags.

Side by side, I [integrated](https://github.com/accordproject/markdown-transform/pull/424) the current transformation with the [markus](https://github.com/accordproject/markdown-transform/tree/master/packages/markdown-cli). This esnures that one can use the transformation using the CLI. The command for transforming ciceromark_parsed to ooxml looks like:

## Create and view the transformed file
To create a file, use: 
```bash
markus transform --from ciceromark_parsed --to ooxml --input pathName --output pathName.xml
```
**Note**: Save files with `XML` format. See [here](https://raw.githubusercontent.com/accordproject/markdown-transform/algoo-ooxml/packages/markdown-transform/transformations.png) for the conversions possible.  
To use the created `XML` file, one can follow these steps:
1. Copy the `XML` in the file.
2. Open MS-WORD. Insert the OOXML using the [Script Lab](https://docs.microsoft.com/en-us/office/dev/add-ins/overview/explore-with-script-lab) add-in.
   Be sure to install the add-in.
   
Alternatively, one can also right-click on the document and open it with MS-WORD to see the transformed content.

In the future, integration with the [add-in](https://github.com/accordproject/cicero-word-add-in) will ensure interaction with the transformed file with advanced features. Currently, the features are limited via the above two methods.

## Result
```
# For testing purposes only

### Checking Heading Level 3 *with* some **styling**

Now let us create a paragraph for the same which _will use with **nesting and making `inline codes`**_ to ensure` it works finely.

Also here is a [link](https://www.google.com)

[*Links*](https://www:google.com) can also be styled.

<codeblock tags>
Finally, a codeblock to ensure everything's working fine and nicely.
<codeblock tags>
```
Paste the above [here](https://templatemark-dingus.netlify.app/) and switch to the `ast` tab to see the corresponding CiceroMark.

#### Markdown Representation
![Screenshot (120)](https://user-images.githubusercontent.com/59891164/130188842-f3eb582e-dd11-45bf-83ba-ca7c29a0218e.png)

#### OOXML Representation
![Screenshot (119)](https://user-images.githubusercontent.com/59891164/130188961-9c30b512-266b-4ea5-bfc4-712fbfe3b7b6.png)

#### Sample CiceroMark and OOXML 
![Screenshot (126)](https://user-images.githubusercontent.com/59891164/130321418-461f8422-b6a5-4b58-a5b6-f47e7eeac341.png)

## Current Standing
Currently, the `CiceroMark<->OOXML` can do the following conversions:
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


## Future Goals
The future seems bright for the transformer. There are still few nodes left that require transformation. Some of them are:
- List
- Blockquote
- Image

In addition, the transformer needs to be integrated properly with the [add-in](https://github.com/accordproject/cicero-word-add-in) so that the working can be made more smooth and easier for users.


## Experience
GSoC was a fun journey for me. I got to learn many things this summer by working on the project. From a person, who didn't know much about MS Word (didn't use it much) to understanding the syntax on which word documents are built upon was enthralling.
I would like to thank my mentors [Aman](https://github.com/algomaster99) and [Dan](https://github.com/dselman) for helping me out whenever I got stuck with or was not clear of the approach for solving a particular problem. I would also like to thank the Accord Project Community for providing me the opportunity to work with them and helping me understand the ecosystem and its workings.

I am hoping to contribute to Accord Project in the future by improving the services they offer either by coding or by providing ideas for the same. I will also help others who are looking to contribute to Accord Project by explaining things and the ecosystem passing the torch to the successors. Ultimately, **GSoC is just the beginning**.
