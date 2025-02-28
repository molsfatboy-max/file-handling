## Explainer

Author: Eric Willigers &lt;<ericwilligers@chromium.org>&gt;<br>
Author: Jay Harris &lt;<harrisjay@chromium.org>&gt;<br>
Author: Raymes Khoury &lt;<raymes@chromium.org>&gt;

### Motivation

This proposal gives web applications a way to register their ability to handle (read, stream, edit) files with given MIME types and/or file extensions.

This has many use cases. An image editor might be able to display and edit a number of image formats. A movie player might understand a video format. A data visualizer might generate graphs from .CSV files. A Logo environment might interactively step through Logo programs.

Some more use cases and motivations are listed in the [Ballista explainer](https://github.com/chromium/ballista/blob/master/docs/explainer.md).

There has historically been no standards-track API for MIME type handling. For some years, Chrome packaged apps have been able to register one or more file handlers using a [Chrome-specific API](https://developer.chrome.com/apps/manifest/file_handlers) where each handler may handle specific MIME types and/or file extensions. This provides some insight into the demand for such functionality. As of August 2018, 619 file handlers handle MIME types, while 509 file handlers handle file extensions. Some of the packages apps have more than one file handler. Overall, 580 packaged apps handle MIME types and 337 packaged apps handle file extensions. This usage, and the use cases above, demonstrate the value of being able to associate web applications with MIME types and/or file extensions.

### Example

The following web application declares in its manifest that it can handle CSV and SVG files.

```json
    {
      "name": "Grafr",
      "file_handler": {
        "action": "/open-files",
        "files": [
          {
            "name": "raw",
            "accept": [".csv", "text/csv"]
          },
          {
            "name": "graph",
            "accept": [".svg", "image/svg+xml"]
          }
        ]
      }
    }
```

> Note: `action` **MUST** be inside the app scope.

Each accept entry is a sequence of MIME types and/or file extensions.

On platforms that only use file extensions to describe file types, user agents can match on the extensions ".csv" and ".svg".

On a system that does not use file extensions but associates files with MIME types, user agents can match on the "text/csv" and "image/svg+xml" MIME types. If the web application accepts all text and image formats that the browser supports, "text/\*" and "image/\*" could be used, i.e. "\*" may appear in place of a subtype. "\*/\*" can be used if all files are accepted.

The user can right click on CSV or SVG files in the operating system's file browser, and choose to open the files with the Grafr web application. (This option would only be presented if Grafr has been [installed](https://w3c.github.io/manifest/#installable-web-applications).)

This would create a new top level browsing context, navigating to '{origin}{action}?name={HANDLER_NAME}'. Assuming the user opened `graph.csv` in Graphr the url would be `https://graphr.com/open-files/?name=raw`. When the `load` event is fired, an additional `launchParams` property will be available on the event, containing a list of the files that the application was launched with.

> Note: `load` is possibly not the correct place for this. We are considering other options, including `DOMContentLoaded` or a completely new event.

The shape of `LoadEvent` and `LaunchParams` is described below:
```cs
interface LaunchParams {
  // Cause of the launch (e.g. file_handler|share_target|shortcut|link). Initially only file_handler will be supported but more will likely be added in future.
  // The values of this enum should be based on entries in the manifest (such as 'file_handler' and 'share_target'), where appropriate.
  readonly attribute DOMString cause;
  // The files the application was launched with. 
  sequence<FileSystemFileHandle>? files;
}

interface LoadEvent : Event {
  // An instance of the LaunchParams object above, detailing how the launch happened.
  attribute LaunchParams launchParams;
}
```

An application could then extract the [FileSystemFileHandles](https://github.com/WICG/native-file-system/blob/master/EXPLAINER.md) in a `load` event listener, and handle the files.

```js
window.addEventListener('load', event => {
  // Launch params could be undefined if the browser doesn't support it.
  if (!event.launchParams || !event.launchParams.cause === 'file_handler')
    return;

  const fileHandles = event.launchParams.files;
  // TODO: Handle the files.
});
```

This API is sufficient to allow native files to be opened in web applications. However, it doesn't cater for more advanced use cases, such as opening a file in an existing window, or simply displaying a notification when a file is opened. For these cases applications can add a [launch event handler](https://github.com/WICG/sw-launch/blob/master/explainer.md). The `launchParams` object would be attached to the `launch` event in an identical manner to the `load` event. 

```js
self.addEventListener('launch', event => {
  if (event.launchParams.cause !== 'files')
    return;

  const fileHandles = event.launchParams.files;

  event.waitUntil(async () => {
    const allClients = await clients.matchAll();
    const client = allClients.filter(/*clever logic...*/)[0];

    // No suitable client open, make a new one.
    if (!client)
      await clients.openWindow(url);
      return;
    }

    // Open the files in the existing client we found.
    client.postMessage({ files: fileHandles, type: '/open-files' });
    client.focus();
  });
});
```

> Note: The launch event is likely to have an aggressive timeout, so all File IO should be done in a client window. The details are being considered in [sw-launch](https://github.com/WICG/sw-launch/blob/master/explainer.md#addressing-malicious-or-poorly-written-sites).

### Differences with Similar APIs on the Web

It is worth noting that the proposed method of getting launched files is somewhat different to similar APIs on the web.

- Web Share Target: All relevant data is contained in the POST request to the page
- registerProtocolHandler: Relevant data is contained in the query string the page navigates to

In contrast, when we perform a navigation to the file-handling url, the files are not available as part of a request, so the page has to wait for an additional event to fire. We briefly considered encoding the `FileSystemFileHandles` in the query string in a blob-like format (e.g. `file-handle://<GUIDish>`). However, this presents some problems:
- The page would have to parse file handles from the url itself, adding boilerplate.
- Lifetimes for blob-like urls are complicated, as we can't predict when/where the handle will be used.

In addition, were we designing the existing APIs again today, there is a good change we might take this approach for them too.

### Previous Solutions
There are a few similar, non-standard APIs, which it may be useful to compare this with.

#### [registerContentHandler](https://developer.mozilla.org/en-US/docs/Web/API/Navigator/registerContentHandler)

Register content handler was [deprecated](https://blog.chromium.org/2016/08/from-chrome-apps-to-web.html) due to the lack of two interoperable implementations and the implementation that was available [did not conform to that standard](https://github.com/whatwg/html/commit/b143dbc2d16f3473fcadee377d838070718549d3). This API was only available in Firefox.

Example usage, as per MDN
```js
navigator.registerContentHandler(
    "application/vnd.mozilla.maybe.feed",
    "http://www.example.tld/?foo=%s",
    "My Feed Reader"
);
```

Presumably this API provided readonly access to the file.

##### Why not resurrect this now more vendors are interested?
`registerContentHandler` was not designed to handle files being opened, rather it was designed to handle link clicks that resolve to certain mime types (see [Example 17](https://www.w3.org/TR/html52/webappapis.html#custom-scheme-and-content-handlers-the-registerprotocolhandler-and-registercontenthandler-methods) from the spec).

For Example:
1. It requires file handles to be encodable as a string (this will not be possible with the [native-file-system](https://github.com/WICG/native-file-system/blob/master/EXPLAINER.md) API, which is required to write to files).
2. It can only register handlers for content types. This is a problem because some operating systems only support file extensions and we don't want to spec and maintain a mapping. See this [issue](https://github.com/WICG/web-share-target/issues/74) for more context.

#### [Chrome Apps File Handlers](https://developer.chrome.com/apps/manifest/file_handlers)

Chrome Apps are in the process of being [deprecated](https://arstechnica.com/gadgets/2017/12/google-shuts-down-the-apps-section-of-the-chrome-web-store/) in favour of PWAs. The API was never intended to be a web standard. This API is only available in Chrom(e|ium), and is only available to apps published in the Chrome App Store.

Example manifest.json
```json
{
  ...
  "file_handlers": {
    "graph": {
        "extensions": [ "svg" ],
        "include_directories": false,
        "types": [ "image/svg+xml" ],
        "verb": "open_with"
    },
    "raw": {
        "extensions": [ "csv" ],
        "include_directories": false,
        "types": [ "text/csv" ],
        "verb": "open_with"
    }
  }
}
```

This would cause a `chrome.app.runtime.onLaunched` event to be fired. This event could be handled in the background process or in a client.

Example Handler
```js
function(launchData) {
  if (launchData.source !== 'file_handler') return;

  // TODO File handling
  // launchData.items[0] is the first file.
}
```

#### [WinJS File Handlers](https://msdn.microsoft.com/en-us/windows/desktop/hh452684)

The WinJS API is surprisingly similar to that of Chrome Apps, except that the registration was done in XAML and the name of the event is different. The API is intended to provide file handling integration to UWP Web Apps, available through the Microsoft store. This API is only available in Edge, is isn't accessible from the general web.

Example Registration
```xml
<Package xmlns="http://schemas.microsoft.com/appx/2010/manifest" xmlns:m2="http://schemas.microsoft.com/appx/2013/manifest">
   <Applications>
      <Application Id="AutoLaunch.App">
         <Extensions>
            <Extension Category="windows.fileTypeAssociation">
                <FileTypeAssociation Name="alsdk">
                  <DisplayName>SDK Sample File Type</DisplayName>
                  <Logo>images\logo.png</Logo>
                  <InfoTip>SDK Sample tip </InfoTip>
                  <EditFlags OpenIsSafe="true" />
                  <SupportedFileTypes>
                     <FileType ContentType="image/jpeg">.alsdk</FileType>
                  </SupportedFileTypes>
               </FileTypeAssociation>
            </Extension>
         </Extensions>
      </Application>
   </Applications>
</Package>
```

Example Handler
```js
function onActivatedHandler(eventArgs) { 
    if (eventArgs.detail.kind !== Windows.ApplicationModel.Activation.ActivationKind.file)  
      return;
    // TODO: Handle file activation. 

    // The number of files received is eventArgs.detail.files.size 
    // The first file is eventArgs.detail.files[0].name 
} 
```

## Security and Privacy Considerations

There is a large category of attack vectors that are opened up by allowing websites access to native files. These are dealt with in the [native-file-system](https://github.com/WICG/native-file-system/blob/master/EXPLAINER.md#proposed-security-models) explainer.

The additional security-pertinent capability that this specification provides over the [native-file-system](https://github.com/WICG/native-file-system/blob/master/EXPLAINER.md) API is the ability to grant access to certain files through the native operating system UI, as opposed to through a file picker shown by a web application. Any restrictions as to the files and folders that can be opened via the picker will also be applied to the files and folders opened via the native operating system.

There is still a risk that users may unintentionally grant a web application access to a file by opening it. However, it is generally understood that opening a file allows the application it is opened with to read and/or manipulate that file.

In addition, the following mitigations are recommended.
- User agents **SHOULD NOT** register every site that can handle files as a file handler. Instead, registration should be gated behind installation. Users expect installed applications to be more deeply integrated with the OS. 
- Users agents **SHOULD NOT** register web applications as the default file handler for any file type without explicit user confirmation.
