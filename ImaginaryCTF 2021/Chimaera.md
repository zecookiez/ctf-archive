<div align="center">

## ImaginaryCTF 2021 - Chimaera (Forensics)
*From someone who fell for all the fake flags in the file*

</div>

### Tools Used

* A hex editor
* binwalk
* strings
* zipinfo
* zlib-flate
* 7z

## Solution

### Part 1/3 - `ictf{thr33_`

For this challenge, we are given `chimaera.pdf` to play around with. Opening this up with a PDF reader displays the flag `jctf{red_flags_are_fake_flags}`, which was not the flag unfortunately :(

Before trying out qpdf and the other PDF tools, we opened it up with a text editor to see what was going on:

```
00000000: 7f45 4c46 0201 0100 0000 0000 0000 0000  .ELF............
00000010: 0200 3e00 0100 0000 b000 4000 0000 0000  ..>.......@.....
00000020: 4000 0000 0000 0000 5801 0000 0000 0000  @.......X.......
00000030: 0000 0000 4000 3800 0200 4000 0400 0300  ....@.8...@.....
00000040: 0100 0000 0500 0000 0000 0000 0000 0000  ................
<...some fake flags below this line...>
```

Apart from the `jctf{red_and_fake_flags_form_an_equivalence_class}` found in the file, we notice that this is an executable file as well! After decompiling the program, it turns out that it just prints out a string. Running the file confirms this as well:

```
$ ./chimaera.pdf
ictf{thr33_
```

Oops, decompiling it was an overkill. But at least we got part of the flag, and it also suggests that there are three parts we need to find.

**TL;DR:** Run the PDF as an executable file, get the first part of the flag.

### Part 2/3 - `h34ds_l`

The remainder of the ELF file did not contain any interesting bits, so we need to analyze the rest of the PDF. Running binwalk on the file gave some interesting info:

```
$ binwalk chimaera.pdf
DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
0             0x0             ELF, 64-bit LSB executable, AMD x86-64, version 1 (SYSV)
600           0x258           Zip archive data, at least v1.0 to extract, compressed size: 5139, uncompressed size: 5139, name: chimaera.pdf
```

Looks like there's a zip file as well! After extracting the content, it turns out the zip contains the same `chimaera.pdf` and `notflag.txt` to bait us. It looks like the rest of the flag will be in the PDF itself.

Reading the content of the PDF in a text editor, three sections stand out:

* Two PDF stream objects which are encoded with zlib, and
* Another zip file above the xref table at the end of the file.

Carving out the first stream object into `stream_obj_1.pdf` and decompressing the content using zlib-flate, we obtain the following:

```
$ zlib-flate -uncompress < stream_obj_1.pdf > stream_obj_1.txt
$ strings stream_obj_1.txt
q 0.1 0 0 0.1 0 0 cm
1 1 1 rg
10 0 0 10 0 0 cm BT
/R7 1 Tf
1 0 0 1 0 0 Tm
(h34ds_l)Tj
0 0 0 rg
10 0 0 10 0 0 cm BT
/R7 12 Tf
1 0 0 1 200 400 Tm
(jctf{red_flags_are_fake_flags})Tj
```

Apart from the gibberish and another fake flag, we see `h34ds_l` wrapped in parentheses. Could this be the second part of the flag?

Combining it with the first third of the flag we obtain `ictf{thr33_h34ds_l`. This looks promising. Only one third to go!

**TL;DR:** Carve out the first stream object from the PDF and decompress using zlib-flate to get the second part of the flag.

### Part 2.5/3 - No flags here :(

We still have another stream object to decompress, does it contain some useful information?

Using the same approach from Part 2, we obtain the following:

```
$ zlib-flate -uncompress < stream_obj_2.pdf > stream_obj_2.txt
$ strings stream_obj_2.txt
Comic-Sans
(URW)++,Copyright 2014 by (URW)++ Design & DevelopmentNimbus Mono PS RegularNimbus Mono PSCopyright (URW)++,Copyright 2014 by (URW)++ Design & Development
rstacdefghjkl_{}34
\q3=
&@ZHj
ZLg69H
duF{T
+eJ~xT
JOWtF
Y]NtC
+>>+*?
cmCuO
/HU@
PQRL
E02DD00C
,ObwV[
J^_L
yQ][N
iuuj
kxul
CBN3SX
$VX{mc
ty}z
]Okylnx
```

Not only did we find nothing here, but we got Comic Sans'd as well. Pain.

<center>
    <img src="https://i.imgur.com/NpE0hMm.png">
    <p>They should've added this in the stream object as well</p>
</center>

Uploading `stream_obj_2.txt` to [TrID](http://mark0.net/soft-trid-e.html) further confirms that this is a [CFF (Compact Font Format)](https://en.wikipedia.org/wiki/PostScript_fonts#Compact_Font_Format) file, which can apparently be embedded into PDFs. Opening this in FontForge, it appears that this file contains the font used to display `jctf{red_flags_are_fake_flags}` in the original PDF. Baited again.

**TL;DR:** The second stream object does not contain anything useful, and we find more fake flags.

### Part 3/3 - `1k3_kerber0s}`

Before reading this part, we recommend you be familiar with the zip file structure. We found the following resources helpful:

* **Zip101**: [https://github.com/corkami/pics/blob/master/binary/zip101/zip101.pdf](https://github.com/corkami/pics/blob/master/binary/zip101/zip101.pdf)
* **The structure of a PKZip (Zip) file**: [https://users.cs.jmu.edu/buchhofp/forensics/formats/pkzip.html](https://users.cs.jmu.edu/buchhofp/forensics/formats/pkzip.html)


The last section we need to look through is the zip file at the end of the document. Carving it out into `zipped.zip` and running zipinfo on it we get the following:

```
$ zipinfo zipped.zip
Archive:  zipped.zip
Zip file size: 272 bytes, number of entries: 2
error [zipped.zip]:  missing 5200 bytes in zipfile
  (attempting to process anyway)
-rw----     1.0 fat       30 b- stor 80-000-00 00:00 notflag.txt
-rw----     1.0 fat     5139 b- stor 80-000-00 00:00 chimaera.pdf
2 files, 5169 bytes uncompressed, 5169 bytes compressed:  0.0%
```

Not only are there errors in the file, but it seems to contain some more bait. But this was at the end of the PDF file, how is it supposed to contain the original `chimaera.pdf` with zero compression? Something is not right.

Opening this up in the hex editor further confirms our suspicions:

```
00000000: 504b 0304 0a00 0000 0000 0000 0000 c380  PK..............
00000010: c290 0668 1e00 0000 1e00 0000 0b00 0000  ...h............
00000020: 6e6f 7466 6c61 672e 7478 746a 6374 667b  notflag.txtjctf{
00000030: 6661 6b65 5f66 6c61 6773 5f61 7265 5f72  fake_flags_are_r
00000040: 6564 5f66 6c61 6773 7d50 4b03 040a 0000  ed_flags}PK.....
00000050: 000d 0000 0000 000d c386 13c2 9321 0000  .............!..
00000060: 000d 0000 0000 0000 0009 0405 005d 0000  .............]..
00000070: c280 0000 18c2 9ac3 8266 01c2 8f45 607e  .........f...E`~
00000080: c388 c29c c3a0 5165 c289 c287 c39c c3bf  ......Qe........
00000090: c3bf 072c 0000 504b 0102 0a00 0000 0000  ...,..PK........
000000a0: 0000 0000 0000 c380 c290 0668 1e00 0000  ...........h....
000000b0: 1e00 0000 0b00 0000 0000 0000 0000 0000  ................
000000c0: 0000 5014 0000 6e6f 7466 6c61 672e 7478  ..P...notflag.tx
000000d0: 7450 4b01 020a 0000 0000 0000 0000 0000  tPK.............
000000e0: 00c3 9644 c2a0 1113 1400 0013 1400 000c  ...D............
000000f0: 0000 0000 0000 0000 0000 0000 0058 0200  .............X..
00000100: 0063 6869 6d61 6572 612e 7064 6650 4b05  .chimaera.pdfPK.
00000110: 0600 0000 0002 0002 0073 0000 00c3 9614  .........s......
00000120: 0000 3601 0a                             ..6..
```

Apart from the very clear `notflag.txt` and fake flags, there appears to be another zip between it and the central directory! And unlike the rest of the files, two things stand out:

* The file is compressed with some unknown algorithm. The header's compression algorithm is `0x0d` (address `0x51`), which isn't mapped to a standard compression algorithm since 13 is a reserved value.
* The file does not have an entry in the central directory, making it invisible to zip extraction tools.

Both of these combined make it impossible for standard tools to unzip it without modifying the header. Let's fix that.

There are three things we need to do to fix the file:

1. Add an entry to the central directory for the hidden file,
2. Remove the other two entries in the zip file, and
3. Figure out the compression algorithm used by the challenge author.

Rather than adding a new entry, we can modify one of the existing central directory entries to decompress the hidden file:

* Change the compression method to `0d` to match the hidden file's compression algorithm,
* Change the CRC32 signature to `0dc61393` to match the hidden file's signature found in the local header,
* Change the compressed and uncompressed sizes to `21` and `0d` respectively to match the hidden file (we also get a hint that the uncompressed data has 13 bytes, meaning the last third of the flag has 13 letters),
* Change the offset of the local header to `00` (the original value had an offset that took the entire PDF file into account, which also explains how unzipping the file gave us `chimaera.pdf` again!), and
* Remove the file name and change its length to 0.

We can remove the other entry to the central directory as well, and modify the end of the central directory record to match with the new directory. This is the resulting file with all the modifications:

```
00000000: 504b 0304 0a00 0000 0d00 0000 0000 0dc6  PK..............
00000010: 1393 2100 0000 0d00 0000 0000 0000 0904  ..!.............
00000020: 0500 5d00 0080 0000 189a c266 018f 4560  ..]........f..E`
00000030: 7ec8 9ce0 5165 8987 dcff ff07 2c00 0050  ~...Qe......,..P
00000040: 4b01 020a 0000 0000 000d 0000 0000 000d  K...............
00000050: c613 9321 0000 000d 0000 0000 0000 0000  ...!............
00000060: 0000 0000 0000 0000 0000 0000 0050 4b05  .............PK.
00000070: 0600 0000 0001 0001 002e 0000 003f 0000  .............?..
00000080: 0000 000a                                ....
```

Running zipinfo on it shows that no errors are found, but we can't unzip it just yet since we are still missing the compression algorithm. Thankfully, there are only 10-20 standardized compression methods, so we can try all of them and modify the file as we go.

Though there is a way to identify the compression algorithm. Carving the compressed data into `data.txt` and running binwalk on it, we get the following:

```
$ binwalk data.txt
DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
4             0x4             LZMA compressed data, properties: 0x5D, dictionary size: 8388608 bytes, missing uncompressed size
```

Compression algorithm 14 (`0e`) is the LZMA compression, perhaps this is the correct compression? Let's change the file's compression from `0d` to `0e` and see if we can extract it properly:

```
$ 7z x zipped.zip

<...>

Scanning the drive for archives:
1 file, 131 bytes (1 KiB)

Extracting archive: zipped.out
--
Path = zipped.out
Type = zip
Physical Size = 131

Everything is Ok

Size:       13
Compressed: 131
```

The extraction worked! Let's read the content of the zip now:

```
$ strings zipped
1k3_kerber0s}
```

Success! And just as the first part of the flag indicated, the full flag is made from three separate parts. Combining the three parts we obtain `ictf{thr33_h34ds_l1k3_kerber0s}`.

**TL;DR:** There is a hidden zip file between `notflag.txt` and `chimaera.pdf` at the end of the PDF. Fixing the structure of the file and identifying its compression algorithm, we extract the third part of the flag as `1k3_kerber0s}`.


## Resources Used

* **binwalk**: [https://github.com/ReFirmLabs/binwalk](https://github.com/ReFirmLabs/binwalk)
* **Compact Font Format**: [https://en.wikipedia.org/wiki/PostScript_fonts#Compact_Font_Format](https://en.wikipedia.org/wiki/PostScript_fonts#Compact_Font_Format)
* **The structure of a PKZip (Zip) file**: [https://users.cs.jmu.edu/buchhofp/forensics/formats/pkzip.html](https://users.cs.jmu.edu/buchhofp/forensics/formats/pkzip.html)
* **TrID**: [http://mark0.net/soft-trid-e.html](http://mark0.net/soft-trid-e.html)
* **Zip101**: [https://github.com/corkami/pics/blob/master/binary/zip101/zip101.pdf](https://github.com/corkami/pics/blob/master/binary/zip101/zip101.pdf)