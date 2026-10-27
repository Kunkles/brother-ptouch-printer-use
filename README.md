# Brother P-touch Printer Use

Last updated 26 October 2026.

## What this is

You can print to a Brother P-touch TZe label printer straight from your own code, over USB or Bluetooth, with no Brother driver in between. This is what building [Ringr](https://github.com/Kunkles/Ringr-releases), a Mac app that prints iris marking rings, taught us about doing it, model by model.

It covers the 128-pin TZe models: PT-D610BT, PT-P710BT, PT-D460BT, PT-D600, PT-E550W and PT-P750W. Each claim says where it comes from:

- **Confirmed** means a real printer did it.
- **Brother's manual** means the *PT-E550W/P750W/P710BT Raster Command Reference*.
- **Brother's data** means the per-model driver files and Print SDK that ship inside P-touch Editor.
- **Inferred** means our reading of that data, not yet seen on hardware.

Two models are confirmed: the PT-D610BT over USB and Bluetooth, and the PT-P710BT over USB. The rest are as solid as their source.

## Models at a glance

Six models share one 128-pin head and one command set, but they differ in three bytes that decide whether a job prints at all. Model code is byte 4 of the status reply. Page byte is n9 in `ESC i z` for a one-label job.

| Model | Model code | USB product ID | 180 × 360 fine | Status notifications | Page byte | Printing confirmed |
| --- | --- | --- | --- | --- | --- | --- |
| PT-D610BT | 0x7C | 0x20E1 | Yes | Yes over USB; silent during a job over Bluetooth | 2 (last) | Yes, USB and Bluetooth |
| PT-P710BT | 0x76 | 0x20AF | Yes (Brother's data) | Yes (Brother's data) | 0 (first) | Yes, USB |
| PT-D460BT | 0x7B | 0x20E0 | No: Brother's data lists 180 × 180 only | Yes (inferred) | 2 (last) | No |
| PT-D600 | 0x6A | 0x2074 | No: Brother's SDK builds 180 × 180 when asked for fine | No (Brother's SDK never sends it) | 0 (first) | No |
| PT-E550W | 0x66 | — | Yes (Brother's data) | No (Brother's manual) | 0 (first) | No |
| PT-P750W | 0x68 | — | Yes (Brother's data) | No (Brother's manual) | 0 (first) | No |

The page bytes and notification columns are from Brother's own Print SDK, recorded through the mock printer it ships with. A PT-P710BT reports 0x76, not the 0x6F that several sources quote.

## Connecting on macOS

On current macOS, talk to the printer's USB interface directly. A print queue won't work: `lpadmin -m raw` answers "raw queues are no longer supported". Both USB and Bluetooth are two-way and carry the same bytes.

**USB, through IOUSBHost**

- Brother's vendor ID is 0x04F9. The printer is interface 0, class 7, subclass 1, protocol 2: a bidirectional printer. It runs at full speed.
- Bulk OUT is endpoint 0x02, bulk IN is 0x81.
- IOKit matching on vendor plus printer class matches nothing. Fetch every `IOUSBHostInterface` and filter class and vendor in code. The vendor, serial and name live on the parent device.
- Ask too early and a bulk read returns an empty packet instead of waiting. Keep reading until 32 bytes arrive.
- Send a job in large transfers, back to back. A bulk pipe has flow control: the printer refuses a packet it has no room for and the host retries. Confirmed: a 64 KB fine job went to a PT-D610BT in 0.29 s and printed.
- A charge-only cable looks like a printer that won't answer. The port reports a cable, the host takes the USB role, nothing enumerates, and accessory power goes off about 30 s later.
- Other apps that open every new USB device, such as Stream Deck, did not block printing.

**Bluetooth**

- Classic Bluetooth RFCOMM (serial port profile). `ESC i S` returns the same 32-byte status as USB.
- The printer takes one session at a time. Brother's own app holding a session, on the Mac or on a phone, can block yours. Close it first.
- Pace the job to the print speed. A PT-D610BT's receive buffer overran when the job arrived faster than the tape came out; batches of about 512 bytes, released just ahead of the print head, fixed it.
- A paired but sleeping printer accepts writes into a local buffer. Insist on a status reply before sending a job, or the job vanishes and looks like a success.

## Reading the printer

Send `ESC i S` (`1B 69 53`) and the printer answers with 32 bytes. Read it before every job: it gives the model, the tape width and any error, and refusing a bad job here wastes no tape. Brother's own software opens every job with this request.

| Offset | Field | Values worth knowing |
| --- | --- | --- |
| 0–2 | Head mark, size, Brother code | `80 20 42`, fixed |
| 3 | Series code | `30` for these P-touch models |
| 4 | Model code | See *Models at a glance* |
| 8 | Error information 1 | 0x01 no media, 0x04 cutter jam, 0x08 weak batteries |
| 9 | Error information 2 | 0x01 replace media, 0x10 cover open, 0x20 overheating |
| 10 | Media width, mm | 18 for 18 mm TZe |
| 11 | Media type | 0x01 laminated, 0x03 non-laminated, 0x11 / 0x17 heat-shrink, 0xFF incompatible, 0x00 none |
| 18 | Status type | 0x00 reply, 0x01 printing completed, 0x02 error occurred, 0x06 phase change |
| 19 | Phase type | 0x00 ready, 0x01 printing |
| 24–25 | Tape and text colour | |

- **Don't key on the model code alone.** Brother's driver data gives some of its TD-series label printers the same codes as the PT-E550W (0x66), PT-P750W (0x68) and PT-D600 (0x6A). Check the USB product name too.
- **Expect undocumented media types.** One PT-P710BT reported 0x14, which no table lists, and 0x01 weeks later. Don't refuse a job on the media type.
- **Weak batteries matter.** A printer warning about low batteries halts the job to show it, so the tape stops part way and is never cut.

## Building a job

Copy Brother's job, byte for byte. A PT-D610BT forgives a lot, but other models refuse commands the D610BT ignores, and they do it silently. This is the order Brother's Print SDK sends for one label, read off the jobs it wrote for each model:

1. `1B 69 53`: status request. Read the reply first.
2. 100 bytes of `00`: invalidate, clearing any half-sent job. The SDK sends 200 to a PT-D610BT.
3. `1B 69 61 01`: switch to raster mode, before initialising.
4. `1B 40`: initialise.
5. `1B 69 21 00`: status notifications on. **Skip this on the PT-D600, PT-E550W and PT-P750W.** The SDK never sends it to them.
6. `1B 69 7A` + 10 bytes: print information. The bytes are flags, media type, width in mm, length (`00`), raster line count as 4 bytes little-endian, page byte, `00`.
    - Normal quality: flags `84` (width + recover), media type `01`.
    - Fine quality: flags `C6`, media type `09`.
    - Page byte `00` (first page) on the PT-P710BT, PT-D600, PT-E550W and PT-P750W; `02` (last page) on the PT-D610BT and PT-D460BT.
7. `1B 69 4B 08`: advanced mode, no chain printing. `48` adds fine quality.
8. `1B 69 4D 40`: various mode, auto cut.
9. `1B 69 41 01`: cut every label. Brother's manual says the P710BT lacks this; the SDK sends it anyway.
10. `1B 69 64 0E 00`: margin, 14 dots. Use 28 (`1C 00`) at fine quality.
11. `4D 02`: TIFF compression.
12. One `47 nn nn` + data per raster line, the count little-endian. Brother sends every line this way, blank ones included.
13. `1A`: print with feed. The job ends here.
14. `1B 69 61 FF`, sent separately once the printer reports the page done.

For example, a PT-P710BT job for a 1,457-line label at normal quality on 18 mm tape opens like this after the invalidate (`B1 05` is 1,457 little-endian):

```
1B 69 61 01  1B 40  1B 69 21 00
1B 69 7A 84 01 12 00 B1 05 00 00 00 00
1B 69 4B 08  1B 69 4D 40  1B 69 41 01  1B 69 64 0E 00  4D 02
```

**`5A` (zero raster) is valid only under TIFF compression.** Brother's manual says so, and it is how one model hung. Simplest is to do what Brother does: never send it, and send blank lines in full.

The SDK also sends a PT-D610BT four extra commands (`1B 69 70`, `1B 69 6B`, `1B 69 4C`, `1B 69 43`). Ringr leaves them out and the D610BT prints correctly.

## Tape geometry

The head is 128 pins at 180 dpi across the tape. Along the tape it steps at 180 dpi, or 360 at fine quality. Each raster line is 16 bytes, most significant bit first: pin 0 is bit 7 of byte 0.

| Quantity | Value |
| --- | --- |
| 18 mm tape | 8 offset dots, 112 printable, 8 unused: 15.8 mm of print on 18 mm |
| One raster line | 0.1411 mm at 180 dpi, 0.0706 mm at 360 |
| Minimum margin (feed) | 14 dots, about 2 mm; 28 at fine quality |
| Head to cutter | 172 dots (24.3 mm) on PT-D600, D610BT, P710BT; 175 on P750W, E550W; 177 on D460BT |
| Longest label | 1,000 mm |

The head-to-cutter gap is why a printer feeds about 24 mm of blank tape before the first printed line can be cut off. Plan short labels around it. Other tape widths sit at other offsets; take them from Brother's manual.

**Stamp marks at whole dots.** A 0.8 mm line is 5.67 dots at 180 dpi. Drawn with anti-aliasing and then thresholded to black and white, every such line leaned 0.088 mm the same way: invisible, systematic, and fatal where position is the point. Round each line's edges to whole dots yourself.

**Lengths drift by printer.** Feed rate varies slightly from one machine to another. Print a strip of known length, measure it, and scale from there. Or, as Ringr does for its ruler and rings, print both strips on the same printer so the error cancels.

## Knowing it printed

A job is printed when the printer says it started and then says it finished. A "ready" afterwards is not enough: a printer that ignored the whole job also comes back ready, with no error bits set.

With `ESC i !` on, a PT-D610BT over USB reports on its own, in this order:

1. Phase change to printing: status type 0x06, phase 0x01.
2. Printing completed: status type 0x01. This comes before the cut, about 7 s after the job for a 20 mm strip, because the tape still has to feed past the cutter.
3. Phase change to ready: status type 0x06, phase 0x00.

Wait for those reports, never a time estimate, and give the printer about a second after "completed" before closing the connection so the cutter is not interrupted. Don't send status requests while it prints: a printer asked every couple of seconds mid-job stopped answering, and ignored the next job until it was power cycled.

Where no reports come, say so honestly:

- **Over Bluetooth, a PT-D610BT sends nothing during a job**, even a good one. Ask for status once the tape has had its time, and report the result as unconfirmed, not failed.
- **The PT-D600, PT-E550W and PT-P750W have no notifications at all.** Same: unconfirmed.
- **Over USB, a model that does notify but never said it started did not print.** Report that as a failure and suggest a power cycle. Ringr once called exactly this a printed label.

## Pitfalls that cost us

Most of these looked like something else at first. Find your symptom; the cause and the fix are beside it.

| Symptom | Cause | Fix |
| --- | --- | --- |
| PT-P710BT: light blinks, nothing prints, has to be switched off and on | `5A` zero raster sent without TIFF compression | Send every line as `47`; send `5A` only under `4D 02`, or never |
| PT-P710BT: takes the job in silence, never starts, comes back ready | Its job differed from Brother's: page byte `02` (last) where Brother sends `00` (first), plus command order and compression. Brother's job byte for byte fixed it; USB jobs began going out in one write at the same time, so which difference mattered is not isolated | Send Brother's job for the model, page byte included |
| PT-D610BT: whole image printed but held inside on "Printing, please wait"; the next job feeds and cuts it | Page byte `00` (first) on a model that wants `02` (last) | Use the page byte per model |
| Strip comes out twice as long | Fine quality sent to a model that has none; it stepped at 180 | Print the PT-D460BT and PT-D600 at 180 × 180 |
| Tape stops part way and is never cut | The printer halts a job to show a weak-battery warning | Run on the mains adapter, or warn before printing |
| Bluetooth job vanishes, yet reads as sent | A paired but sleeping printer buffers writes locally | Require a status reply before sending |
| Can't connect, or the job is ignored | Another app holds the printer's one session (Brother's app, a phone) | Close it first |
| Printer stops answering, and ignores the next job too | Status requests sent every couple of seconds while it printed | Listen during a job; ask only after it should have finished |
| A failed job reported as printed | "Ready" taken as finished, though the printer never started | Require a "printing" report on models that send them |
| Every mark sits 0.088 mm to one side | Anti-aliased lines thresholded to one bit | Stamp edges at whole dots |
| "raw queues are no longer supported" | macOS no longer takes raw CUPS queues | Talk to the USB interface directly |
| Nothing enumerates; port power off after about 30 s | Charge-only cable | Use a data cable |
| A failed printer stays failed on the next job | Printers latch an error state | Start every job with the invalidate and `1B 40`; after a hang, power cycle |

## Adding a model

A printer that answers a status request has not yet proved it will print. One PT-P710BT answered every status check perfectly and ignored every job, until it was sent Brother's own job byte for byte. Work through these before calling a model supported:

- [ ] **Read its status reply.** Take the model code from the printer itself, not from the web.
- [ ] **Read Brother's driver data for it.** On a Mac with P-touch Editor, open `P-touch Editor.app/Contents/Frameworks/BRLBXWrapperMac.framework/Versions/A/Resources/ptd.bundle/`. Each model has a `bsp*.json` file:
    - `dwheadpinnum`: 128 for everything here.
    - `wresolutions`: `180,180` only means no fine mode.
    - `wmaxcuteverynum`.
    - `dwcommandability01`: a bit field. Compare it with a model you know.
    - `driverlist.json` gives the head-to-cutter gap (`precut_free`, in dots).
- [ ] **Get Brother's own job for it, if you can.** The Print SDK inside P-touch Editor ships a mock printer that records every byte the SDK writes. Its job settles the page byte, which commands to send and their order.
- [ ] **Print a short strip on the real printer.** Check that it reports printing, then completed; that the tape is fed out and cut; and that the strip measures the length you sent.
- [ ] **Then try fine quality, then Bluetooth if it has it.** Each is a separate pass.
- [ ] **Ask the first owners to report back.** Print outcomes, not just status replies, are what tell you a model works.

## Sources and caveats

- **Brother, *Software Developer's Manual: Raster Command Reference, PT-E550W/P750W/P710BT*, version 1.02**, from Brother's developer support site: commands, the status layout, media types, error bits.
- **Brother P-touch Editor 5.3.24 for Mac**: per-model driver data in its `ptd.bundle`, and its Print SDK (`MacPrintSDK.framework`), whose mock printer recorded the jobs quoted here. Nothing here comes from a real printer except where marked confirmed.
- **Ringr's own printing**: USB and Bluetooth on a PT-D610BT, macOS 15, 18 mm laminated TZe. [Ringr releases](https://github.com/Kunkles/Ringr-releases).

Caveats:

- The PT-D610BT is confirmed over USB and Bluetooth, the PT-P710BT over USB only. The zero-raster fix alone did not make the P710BT print; Brother's whole job did.
- Rows marked inferred rest on our reading of Brother's capability bits, which Brother does not document.
- Only 18 mm tape has been measured. Other widths have other offsets.

Corrections and reports from other models are welcome: open an issue with your model, the link (USB or Bluetooth) and what happened.

## Licence

[CC BY 4.0](https://creativecommons.org/licenses/by/4.0/): free to share and adapt for any purpose, with credit and a link back here. See [LICENSE](LICENSE).

Brother, P-touch and the model names used here are trademarks of Brother Industries, Ltd. This guide is not affiliated with or endorsed by Brother.
