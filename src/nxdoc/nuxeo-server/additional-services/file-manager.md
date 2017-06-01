---
title: File Manager
review:
    comment: ''
    date: '2016-12-13'
    status: ok
labels:
    - lts2016-ok
    - file-manager
    - file-upload-component
    - excerpt
    - link-update
toc: true
confluence:
    ajs-parent-page-id: '16089319'
    ajs-parent-page-title: Additional Services
    ajs-space-key: NXDOC
    ajs-space-name: Nuxeo Platform Developer Documentation
    canonical: File+Manager
    canonical_source: 'https://doc.nuxeo.com/display/NXDOC/File+Manager'
    page_id: '19235627'
    shortlink: K4MlAQ
    shortlink_source: 'https://doc.nuxeo.com/x/K4MlAQ'
    source_link: /display/NXDOC/File+Manager
tree_item_index: 100
history:
    -
        author: Solen Guitter
        date: '2016-08-31 08:39'
        message: ''
        version: '10'
    -
        author: Solen Guitter
        date: '2015-12-07 15:23'
        message: link update
        version: '9'
    -
        author: Antoine Taillefer
        date: '2015-10-13 10:32'
        message: ''
        version: '8'
    -
        author: Solen Guitter
        date: '2015-07-06 13:02'
        message: ''
        version: '7'
    -
        author: Manon Lumeau
        date: '2015-07-01 09:58'
        message: ''
        version: '6'
    -
        author: Manon Lumeau
        date: '2015-07-01 09:50'
        message: ''
        version: '5'
    -
        author: Solen Guitter
        date: '2015-06-15 08:03'
        message: Step formatting
        version: '4'
    -
        author: Michaël Vachette
        date: '2015-06-12 10:13'
        message: ''
        version: '3'
    -
        author: Alain Escaffre
        date: '2014-05-02 16:00'
        message: ''
        version: '2'
    -
        author: Alain Escaffre
        date: '2014-05-02 15:57'
        message: ''
        version: '1'

---
{{! excerpt}}

The File Manager is used to create documents from simple binaries.

{{! /excerpt}}

From a user perspective, the Nuxeo Platform offers many ways to capture a binary so as to make it a document with a binary property. It can be done from the browser's [drag and drop]({{page space='userdoc' page='creating-content'}}#importing-documents-using-drag-and-drop), from the [upload REST API]({{page page='howto-upload-file-nuxeo-using-rest-api'}}), from [WebDAV]({{page space='userdoc' page='working-with-webdav'}}), from [CMIS]({{page page='cmis'}}), from [Nuxeo Drive]({{page page='nuxeo-drive'}}), ...

The File Manager service is a traditional Nuxeo Platform service that offers some methods that help standardize what happens when a file is captured in the Platform, in regard to:

* What document type should be created.
* What is the property mapping logic.
* What is the versioning policy.
* How should the binary be processed (exploding a zip, doing some pre-persistence conversions).

## Customising and Using the File Manager Service

*   The File Manager service has a plugin architecture, so that it is possible to contribute different policies depending on the MIME type of the file and the context. The default Nuxeo Platform use cases of the File Manager can be customised reading the documentation of the [file manager service extension point](http://explorer.nuxeo.org/nuxeo/site/distribution/latest/viewExtensionPoint/org.nuxeo.ecm.platform.filemanager.service.FileManagerService--plugins). There are also means of [controlling the versioning policy](http://explorer.nuxeo.org/nuxeo/site/distribution/latest/viewExtensionPoint/org.nuxeo.ecm.platform.filemanager.service.FileManagerService--versioning) of documents updated via this channel. Finally, there is a helper for [implementing binary unicity checks](http://explorer.nuxeo.org/nuxeo/site/distribution/latest/viewExtensionPoint/org.nuxeo.ecm.platform.filemanager.service.FileManagerService--unicity).
*   The File Manager can be called and used in your custom Java code with the standard service call pattern:

    ```
    ... fileManager = Framework.getService(FileManager.class);
    DocumentModel createdDoc = fileManager.createDocumentFromBlob(coreSession, blob, path, true, fileName);
    ```

*   And you can also use the [dedicated](http://explorer.nuxeo.org/nuxeo/site/distribution/latest/viewOperation/FileManager.Import) Automation operation, which provides a way to create in one REST call a document from a binary, or to create easily a document from a blob in an Automation chain.

## Implementing Your Own Plugin

Let's take a simple example where the document type depends on the parent folder type. In order to do that in a fully automated way, you'll need to implement your own plugin. For example, we have a folder type `PurchaseOrderFolder` which accepts only `PurchaseOrder` documents as children, and an `InvoiceFolder` which accepts only `Invoice` documents.

1.  Write a class which extends `org.nuxeo.ecm.platform.filemanager.service.extension.AbstractFileImporter`.

    ```java
    public class SampleFilemanagerPlugin extends AbstractFileImporter {

        private static final long serialVersionUID = 1876876876L;

        private static final Log log = LogFactory.getLog(SampleFilemanagerPlugin.class);

        @Override
        public DocumentModel create(CoreSession session, Blob content, String path, boolean overwrite, String filename, TypeManager typeService)
                throws ClientException, IOException {

            PathRef root = new PathRef(path);
            DocumentModel rootDoc = session.getDocument(root);
            DocumentModel doc = null;
            switch (rootDoc.getType()) {
                case "PurchaseOrderFolder":
                  doc = createDocType(session, path, content, filename,"PurchaseOrder");
                  break;
                case "InvoiceFolder":
                  doc = createDocType(session, path, content, filename,"Invoice");
                  break;
                default:
                  break;
            }
            if (doc!=null) {
              doc = session.createDocument(doc);
            }
            return doc;
        }

        private DocumentModel createDocType(CoreSession session, String path, Blob content, String filename, String type) {
            DocumentModel doc = session.createDocumentModel(path,type,type);
            doc.setPropertyValue("dc:title",filename);
            doc.setPropertyValue("file:content", (Serializable) content);
            doc.setPropertyValue("file:filename",filename);
            return doc;
        }
    }
    ```

    The created method returns either a `DocumentModel` object or `null`. If `null` is returned, then the File Manager service will try with the next plugin. Within this method, we have access to the destination path which enables us to determine the type of the folder and thus create a document with the relevant type.

2.  Add a new contribution to the File Manager service.

    ```xml
    <component name="org.nuxeo.sample.filemanager">
        <extension target="org.nuxeo.ecm.platform.filemanager.service.FileManagerService" point="plugins">
            <plugin name="SampleImporter" class="org.nuxeo.sample.SampleFilemanagerPlugin" order="0">
                <filter>.*</filter>
            </plugin>
        </extension>
    </component>
    ```

    That's it!

See also the tutorial on [how to change the default document type used when importing files in the Nuxeo Platform]({{page page='how-to-change-the-default-document-type-when-importing-a-file-in-the-nuxeo-platform'}}).

* * *

<div class="row" data-equalizer data-equalize-on="medium"><div class="column medium-6">{{#> panel heading='Related Documentation'}}

- [REST API]({{page page='rest-api'}})
- [Automation]({{page page='automation'}})
- [Versioning]({{page page='versioning'}})
- [How to Change the Default Document Type When Importing a File in the Nuxeo Platform?]({{page page='how-to-change-the-default-document-type-when-importing-a-file-in-the-nuxeo-platform'}})

{{/panel}}</div><div class="column medium-6">

{{! Please update the label and target spaces in the Content by Label macro below. }}

</div></div>