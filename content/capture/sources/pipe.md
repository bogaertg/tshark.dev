---
title: "Pipes"
description: "Packet Headwaters"
date: 2019-04-08T12:44:45Z
author: Ross Jacobs

summary: '[Wireshark Docs](https://wiki.wireshark.org/CaptureSetup/Pipes)'
weight: 30
draft: false
---

## Piping with *shark

Piping is important to using many of these utilities. For example, it is not
really possible to use rawshark without piping as it expects a FIFO or stream.

| Utility        | stdin formats        | input formats      | stdout formats     | output formats (default) |
|----------------|----------------------|--------------------|--------------------|--------------------------|
| **capinfos**   | -                    | *pcaps<sup>1</sup> | report<sup>2</sup> | -                        |
| **dumpcap**    | -                    | -                  | rawpcap            | *pcaps (pcapng)          |
| **editcap**    | -                    | *pcaps             | -                  | *pcaps (pcapng)          |
| **mergecap**   | -                    | *pcaps             | -                  | *pcaps (pcapng)          |
| **randpkt**    | -                    | -                  | -                  | (pcap)                   |
| **rawshark**   | raw pcap<sup>3</sup> | -                  | report             | -                        |
| **reordercap** | -                    | *pcaps             | -                  | (Same as input)          |
| **text2pcap**  | hexdump<sup>4</sup>  | -                  | -                  | (pcap), pcapng           |
| **tshark**     | raw pcap             | *pcaps             | *many<sup>5</sup>  | *pcaps, (pcapng)         |

---

1. __*pcaps__: All pcap types available on the system (use `tshark -F` to list).
2. __report__: Tabular or "machine-readable" data about a file.
3. __rawpcap__:  The raw bytes of the pcap header and packets. Can be generated
  with `cat $file | ...`, read by piping to `... | tshark -r -`, and saved with
  `... > $file`.
4. __hexdump__: A formatted hexdump can be canonically generated by `od -Ax -tx1 -v`. As of
  Wireshark v3.0.0, `tshark -r <my.pcap> -x` will
  [usually](https://bugs.wireshark.org/bugzilla/show_bug.cgi?id=14639) generate
  this as well. If hexdump is stream, send to text2pcap as
  `<commands>... | text2pcap - <outfile>`.  Otherwise if it's a file, use
  `text2pcap <infile> <outfile>`.
5. __*many__: Tshark is the most versatile in terms of output:
  - rawpcap (`-w -`)
  - Report (`-G`)
  - Packet Representations (accessible with `-T`)
    - text-based: text and tab are the same except for the tab delimiter
      - text/tabs (default): Abbreviated packets with one per line
      - text/tabs -V: PDU Subtrees
    - JSON-based
      - json
      - jsonraw
      - ek
    - XML-based
      - pdml
      - psml

## Using temp files instead of pipes

In bash, it's possible to create temporary files to mimic using a pipe. In
this example, editcap can only read files, so create a temp file, send filtered
tshark output to it, and then read it from editcap to make further alterations.

```bash
tempfile=$(mktemp)
tshark -r dhcp.pcap -Y "dhcp.type == 1" -w $tempfile
editcap $tempfile dhcp2.pcap -a 1:"Cool story bro!"
```

## Pipe Types

An anonymous pipe sends the output of one command to another.
A named pipe (aka FIFO) is a file created by `mkfifo` from which data can be read and to which data can be sent, by different processes.
More information about each can be found in this [stackoverflow post](https://unix.stackexchange.com/questions/436864/how-does-a-fifo-named-pipe-differs-from-a-regular-pipe-unnamed-pipe)

### Anonymous Pipe

In this example, tshark reads packets and sends the packet bytes to stdout. The stdout is written to the pipe which is sent to the stdin of a second tshark process.

```bash
# You may need to use sudo to capture
tshark -w - | tshark -r -
```

This is equivalent to `tshark -r $file`, only using a pipe and an extra tshark process to demonstrate send/recv on `|`.

{{% notice warning %}}
If you are reading from stdin, then the data stream MUST confrom to a capture type that
tshark knows how to parse. This means, for example, that a pcap file needs to
send the pcap header first or the packets that come after won't be parsed.
{{% /notice %}}

### Named Pipe

You can also read from a pipe like so:

```bash
mkfifo myfifo
# You may need to use sudo to capture
tshark -w myfifo & tshark -i myfifo
```

Confusingly, reading a pipe is through `-i` even though a named pipe is a file descriptor.