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
Office Open Extensible Markup Language abbreviated as OOXML is a zip-based XML file format which was developed by Microsoft. It can be used to represent documents, presentations, spreadsheets, etc. 
OOXML acts as backcbone of these documents as it is completely responsible for rendering the content which is visible on a word document or a presentation. 
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

