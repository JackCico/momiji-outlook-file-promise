# Momiji support for Apple URL-based file promises

This experimental Momiji Stable 140 branch adds the missing bridge between
Apple's URL-based pasteboard file-promise protocol and the standard web
`File` API on legacy macOS.

Some native Mac applications do not place a finished file URL on the
pasteboard when a drag begins. Instead, they promise to create the file after
the destination accepts the drop and supplies a private destination folder.
Without a receiver for that protocol, a web drop target may see the native
drag but receive no usable entry in `event.dataTransfer.files`.

This is a Momiji feature branch, not a stock Firefox release. Momiji is built
from the Gecko/Firefox source tree, which is why this repository still
contains the normal Mozilla source layout. The repository name records the
original Outlook use case; the implementation itself contains no Outlook
application check, private Outlook API, mail-only rule, `.eml` restriction, or
file-type allowlist.

## What the patch changes

For compatible external drags, Momiji now:

1. advertises the standard `Files` type before the drop so an ordinary web
   drop target can accept the operation, while leaving the pre-drop
   `FileList` empty;
2. waits until the page accepts the drop before asking the source application
   to materialize anything;
3. supplies a private, browser-controlled paste location and requests the
   promised local file URLs;
4. validates the complete promise result and creates browser-owned snapshots;
5. replaces the placeholder drag data with standard DOM `File` objects before
   dispatching the web `drop` event; and
6. retains those snapshots until orderly browser shutdown so delayed
   `FileReader`, `Blob`, `FormData`, and upload operations remain valid.

Rejected or cancelled drags do not request an export from the source
application. Existing drops that already contain ordinary file URLs, such as
Finder files, continue through Momiji's existing path.

## Exact protocol scope

The receiver is intentionally limited to Apple's URL-based pasteboard file
promise representation. Every native pasteboard item must provide:

- `kPasteboardTypeFileURLPromise`
  (`com.apple.pasteboard.promised-file-url`); and
- non-empty `kPasteboardTypeFilePromiseContent`
  (`com.apple.pasteboard.promised-file-content-type`) metadata.

The patch does not directly implement or claim the older
`NSFilesPromisePboardType` / `namesOfPromisedFilesDroppedAtDestination:`
contract. It also does not claim support for arbitrary drag data,
directories, packages, symlinks, special files, mixed promise/non-promise
item sets, or every application using newer `NSFilePromiseProvider` APIs.

Any source application that correctly exposes the required URL-promise
representations may be compatible, regardless of the promised file's
extension or content type. Outlook 2011 is currently the only source
application validated end to end.

## Validated configuration and result

The implementation has been built and tested on the real target system:

- Momiji Stable 140.17.0 experimental build;
- macOS 10.9.5 Mavericks (`13F34`);
- Microsoft Outlook 2011 14.7.7; and
- a normal standards-based HTML drop target.

The acceptance run confirmed that:

- `Files` was advertised before the accepted drop;
- a real EML appeared in `event.dataTransfer.files` at drop time;
- the complete 99,538-byte file was readable immediately, after one second,
  and after 30 seconds;
- the message contained the expected RFC 822 headers, plain-text and HTML
  bodies, and a readable 49,164-byte PDF attachment; and
- the original message remained fully parseable as `multipart/mixed`.

This proves the tested Outlook 2011/Mavericks path. It does not by itself
claim compatibility with every possible file-promise sender.

## Web drop-target contract

No Momiji-specific widget is required. A normal HTML drop target works when
it follows these rules:

- call `preventDefault()` during `dragover` so the drop is accepted;
- do not reject the drag because `dataTransfer.files` is empty before drop;
- read and retain `dataTransfer.files` immediately inside the `drop` handler;
- do not require `dataTransfer.items`, `getAsFile()`,
  `webkitGetAsEntry()`, or the File System Access API;
- do not require a non-empty `File.type`; and
- upload the original `File`/`Blob` bytes rather than converting a message to
  UTF-8 text first.

The included manual test page implements this contract and performs
immediate, one-second, and 30-second reads.

## Implementation and reproducible build

- Implementation branch:
  [`outlook-file-promise-140.17`](https://github.com/JackCico/momiji-outlook-file-promise/tree/outlook-file-promise-140.17)
- Verified build commit:
  [`8ceceaca069a2ff00d8dc725ee0d4442f3ce8844`](https://github.com/JackCico/momiji-outlook-file-promise/commit/8ceceaca069a2ff00d8dc725ee0d4442f3ce8844)
- Successful GitHub Actions run:
  [`38460481100`](https://github.com/JackCico/momiji-outlook-file-promise/actions/runs/38460481100)
- Receiver patch:
  [`.github/workflows/momiji-modern-file-promise.patch`](https://github.com/JackCico/momiji-outlook-file-promise/blob/outlook-file-promise-140.17/.github/workflows/momiji-modern-file-promise.patch)
- Patch SHA-256:
  `cce2c2d60b54e9bed49adb152c573f237f2ccfa9726753099291f85ffeb03c1d`
- Mavericks artifact SHA-256:
  `bf0f5ff0f7b6c15fe1f29ef28a28f31ecae51cef5e70d5892a748b082f196a2a`

The branch stores the receiver changes as a reviewed patch file. The Actions
workflow verifies its SHA-256, applies it to the Momiji/Gecko checkout, builds
Momiji, and repackages the result for the legacy macOS targets. The five Gecko
source files are therefore changed in the produced build even though they are
not committed as permanently modified files in this experimental branch.

The tested artifact is an unsigned experimental build. It is not an official
Momiji or Mozilla release.

## Safety boundaries and limits

The receiver accepts only local `file:` results that are direct children of
the supplied private destination. It rejects incomplete or mixed results,
duplicate names, non-regular files, symlinks, special files, hard-linked
files, ownership mismatches, identity changes, and byte-copy mismatches.

Built-in limits are:

| Boundary | Limit |
| --- | ---: |
| Promised items per drop | 128 |
| Size per file | 128 MiB |
| Aggregate size per drop | 256 MiB |
| Retained promised-file data per process | 512 MiB |
| Promise metadata per item | 64 KiB |
| Direct staging entries | 1,024 |

The browser-owned snapshots preserve the leaf filename and verified data-fork
bytes. They do not claim to preserve original timestamps, POSIX modes,
extended attributes, Finder metadata, or resource forks.

## Upstream relationship

- [Momiji browser project](https://github.com/aobaharuki2005/momiji-web-browser)
- [Momiji Firefox/Gecko source fork](https://github.com/aobaharuki2005/firefox)
- [Mozilla Firefox source documentation](https://firefox-source-docs.mozilla.org/)

The generic receiver change may ultimately be suitable for upstream review,
but this repository currently documents and builds the tested Momiji
integration.
