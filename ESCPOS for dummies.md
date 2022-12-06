# ESC/POS for dummies

So you are looking into ESC/POS and you can't find any tutorials, or any documentation that actually makes sense? Perhaps you're a self-taught developer and you skipped that part of computer science where they talk about bit manipulation, bytecode, text encoding, and all that? Well, you're in the right place!

The goal of this text is not to teach you everything about ESC/POS, but to explain the basic concepts and provide references, so that you can understand that documentation you were just looking at, the incomprehensibility of which made you google "ESC/POS tutorial for dummies", a bit better. Hopefully that will save you _days_ of trial and error that it took me.

## What is ESC/POS?
ESC/POS is a variant of [ESC/P](https://en.wikipedia.org/wiki/ESC/P) or 'Epson Standard Code for Printers' that is centred around receipt printers for point of sale (POS) devices. It is a universal command language used by most modern receipt printers for controlling the print and its formatting. This includes simple features like printing text, manipulating text alignment, font size, or styling, but also more advanced printing of barcodes or QR codes.

## ESC/POS basics
Depending on the particular printers you have, you may connect to them in a different way - USB, Bluetooth, Ethernet, or maybe even a REST API in the case of cloud printers. ESC/POS enabled printers expect input in a [hexadecimal](https://en.wikipedia.org/wiki/Hexadecimal) byte format, that is 2-digit values in range 0x00-0xff (or 0-255 in decimal). Generally speaking, the text to be printed should be formatted to a hex string and then preceeded by relevant commands.

Of course you cannot possibly work in ESC/POS directly, it's not human-readable (unless you spend as much time on it as I have). You need to write your own parser library that will translate into the hex bytes that the printer wants. The exact implementation will of course depend on the language you choose to use as well as your use-case. We'll get back to that later.

If you're not entirely clear on how the hexadecimal numbers work, stop reading this and look into that. It is absolutely crucial to understand it completely before proceeding.

### ESC/POS commands
Like the rest of the input, the commands are hexadecimal values. They're distinguished from printable text by the use of the [ASCII control codes](https://en.wikipedia.org/wiki/Control_character#In_ASCII) (ASCII values under 32 or 0x20). 

In some cases, the control codes are full commands themselves, e.g 'LF' (`0a`) applies line feed or new line, and 'HT' (`09`) applies a horizontal tab. 

Generally however, they're used as escape characters (hence "*ESC*/POS") for specifying a function. The most commonly used include 'ESC' (or `1b`), 'FS' (`1c`) and 'GS' (`1d`). For example, 'ESC @' (or `1b40`) is a basic command initialising printer and resetting previously applied formatting.

Most commands expect arguments. These are usually **integer** values converted to hexadecimal, but on occasion the rules operate on single bits. For example:
- 'ESC a' sets alignment, based on int argument: 0 - start | 1 - centre | 2 - end. E.g. `1b6101` sets alignment to centre.
- 'ESC E' turns bold on/off: if the lowest bit of the argument is 1, bold is on, else bold is off. E.g. `1b4501` switches bold on, and so does `1b4521`, but `1b4540` and `1b4500` both switch it off.

Some commands may expect multiple arguments, for example 'ESC p' sends a signal to a connector pin (typically an external port, like for cash drawers), and it takes 3 arguments:
1. specifies the pin/port
2. specifies the "on" signal length multiplier
3. specifies the following "off" signal length multiplier
E.g. `1b700019f9`, where `1b70` is the command, `00` represents the first pin, `19` is a multiplier (_25 x 2ms_) of the "on" signal, and `f9` is a multiplier (_250 x 2ms_) of the "off" signal.

Detailed instructions for each command are to be found in the printer's documentation.

Multiple different formats can be applied to the same text at once. For example, it is possible to make the text bold, underlined, and align it to centre at the same time.

Commands are applied in order to any subsequent text, meaning that the last command will always take precedence over the others in cases where they cancel themselves out. For example, `1b4501` + `48656c6c6f` prints "**Hello**" (bold), but `1b4501` + `1b40` + `48656c6c6f` prints "Hello" (regular), because the last applied command before the text (`1b40`) resets the formatting to default.

### Command cavieats
Although ESC/POS is called a "standard language", not all commands work the same way (or at all) on all devices. For example, a standard command 'ESC 4' for setting italic styling doesn't work on any of the printers I tried.

Some commands are also limited by hardware capabilities, for example the 'ESC p' command mentioned earlier assumes that we have external ports, but that's not always the case. Similarly, the 'GS V' command assumes the printer has an auto-cutter, but in fact most don't.

Sometimes manufacturers can also add extra functionality beyond the basic ESC/POS command set. These are typically quite elaborate, like this command: `1d28450300060300` - all but the first three bytes are arguments that select a specific function, so you can imagine there's a lot of room for extra features.

Commands usability can also be limited by the way we connect to the printer. For example, the 'DLE EOT' command returns useful hardware state information, such as if the printer is out of paper or overheated. However the output is written to the buffer, and the buffer cannot be accessed remotely, which renders this command useless in the case of cloud printers.

Generally, if the printer cannot process the command for whatever reason, it safely ignores it, without it affecting the rest of the data. However these are low level operations and it is possible to make a printer reach a state from which it cannot recover on its own. In most cases restarting the device solves the issue, but on cloud printers with an automated message queue the device might get stuck in a loop. So when working with cloud printers, make sure that you know how to reset the message queue in case something goes wrong!

### Character encoding
As already stated, the printers expect hexadecimal input. Most languages have the capability to convert a string into a hex string. For example, in Kotlin we can do it like so:
```Kotlin
fun String.encodeHex() = encodeToByteArray() // Turn string to a byte array
  .asUByteArray() // Converting to unsign bytes to avoid negative values
  .joinToString("") { 
    it.toString(16) // Convert byte to hex string
      .padStart(2, '0') // Add a '0' prefix for single-digit values
  } // Join back to a single string
```
and in Bash like so:
```Bash
xxd -pu <<< "$your_string" | tr -d \\n
```

A single byte can only represent 256 different values. While the Latin alphabet is easily contained within that range, things get complicated when we consider other writing systems like Chinese, Hebrew, or Greek. Most printers usually have all those different character sets (a.k.a. "code pages") built in, but they can only use one at a time.

By default, printers generally operate on [ASCII](https://en.wikipedia.org/wiki/ASCII) values. Chinese printers on the other hand usually operate in the [CJK mode](https://en.wikipedia.org/wiki/CJK_characters) by default. On some printers it is possible to set the encoding to UTF-8 (with multiple-byte encoding), on others it is necessary to select a specific [Code Page](https://en.wikipedia.org/wiki/Code_page) in single-byte encoding. For most use cases the CP-437 (or "extended ASCII") is sufficient, however if your printer allows it, UTF-8 is of course a much more robust solution.

Note: To use code pages, single-byte encoding has to be enabled first. To use CJK or UTF-8 modes, multiple-byte encoding is necessary. See documentation for relevant commands.

### Fonts
Most printers use so-called [dot-matrices](https://en.wikipedia.org/wiki/Dot_matrix_printer) for printing, and have two dot-matrices to select different font sizes (12x24 and 9x17). With a dot-matrix, fonts aren't fully scalable, only option being multiplying their width and height two or more times. 

Receipt printers tend to use paper rolls of around 50-80mm width, so it's unlikely that anything greater than a double 12 font will be useful for anything. In that configuration, combining the 'ESC !' (`1b21`) and 'GS !' (`1d21`) commands you can achieve 4 reasonable font sizes:
- 9 - `1b2101` selects the small font
- 12 - `1b2100` selects the regular font
- 18 - `1b2101` selects the small font + `1d2111` doubles width and height
- 24 - `1b2100` selects the regular font + `1d2111` doubles width and height

Some devices only have a single matrix, which proves problematic, because a double size 12x24 matrix is rather large. On the other hand, some modern printers are capable of also using vector fonts, which are fully scalable. See your printer documentation for details.

## ESC/POS limitations

### Text layout
ESC/POS spacing functionality is very basic, therefore to achieve complex text layouts, you might need to do so manually.

For example, let's say you want to print the following line:
```
2x Asahi Special Edition half-pint
```
This line consists of **34** characters. Depending on the size of the font and the width of the paper roll, this text might be broken down into two lines. For example, if the font size is of width **18**, and the paper roll's width is **58mm**, this line would look like this:
```
2x Asahi Special Edit
ion half-pint
```
This breaks the word "Edition" in an unnatural way. It would be much better if it looked like this:
```
2x Asahi Special 
Edition half-pint
```
To achieve such a result it is necessary to calculate the maximum length for the given font size, divide the text into words, and arrange it into lines manually.

Another example, let's say you want to display related information in columns, like so:
```
| Large Margherita   x3  27.00 |
| Pizza                        |
| Guinness Pint      x2  12.00 |
| Asahi Pint         x1   6.00 |
```
ESC/POS can only handle simple alignment, so once again you need to fill the blank space in between items, wrap text and break it into lines, and deal with aligning the columns manually.

## Final words
That's it. Hopefully you can read the various ESC/POS documents a bit better now.

## Links
- [ESC/POS documentation for Pyramid Printers (for reference)](https://escpos.readthedocs.io/en/latest/)
- [ESC/POS documentation for Star printers (for reference)](https://www.starmicronics.com/support/Mannualfolder/escpos_cm_en.pdf)
